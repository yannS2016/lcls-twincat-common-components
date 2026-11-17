<!--- Provide a general summary of your changes in the Title above -->
# PR Title: ENH: Port FB_KBO_Coating_States and FB_Coating_States_NoPMPS to the ND composition framework

## Description

Ported `FB_KBO_Coating_States` off the old
`EXTENDS FB_PositionStateBase_WithPMPS` pattern to the new motion library.

Two things came up while doing this:

- **KBO's states are raw encoder counts, not scaled positions**, so each
  state needs its own `nEncoderCount`. The existing helper,
  `FB_CC_StatePMPSConfig`, pulls `nEncoderCount` from one shared template,
  which doesn't work here. Added a new helper, `FB_CC_StatePMPSEncConfig`,
  that takes `nEncoderCount` per state instead of changing the existing
  helper (that would force all 10 of its other call sites to pass an
  argument they don't need). It also leaves out `fPosition`, since that
  value gets recalculated every cycle from `nEncoderCount` anyway, so
  anything passed in would just get overwritten.
- **Removed a `SetMotionParam` call** the FB had early on (it picked an
  accel/decel pair based on a `bVerticalCoating` flag, copying old
  behavior). The ND framework's rule is that state movers don't set motion
  parameters; that's the integrator's job, not the device's.

Also added a second device, `FB_Coating_States_NoPMPS`: same number of
states, no PMPS, no raw counts, and every field owned by the integrator
instead of split with the device. It shows the other ownership style this
library uses, next to KBO's.

Last, wrote a migration guide (`docs/common-components-library-migration.html`),
based on the existing `docs/migration.md`, in the Confluence format the
team uses, with both new devices as worked examples.

## Motivation and Context

`FB_KBO_Coating_States` and its sibling `FB_Offset_Coating_States` were the
last real devices still on the old, EXTENDS-based
`FB_PositionStateBase_WithPMPS` family that were due for migration soon.
Other devices in production (`FB_MAA_States`, `FB_TM1K4_States`,
`FB_XS_YAG_States`, `FB_ZonePlate_States`) still extend the same old base
and will need the same work later, so this PR also leaves a documented,
reusable recipe for that instead of making the next person figure it out
from scratch.

## How Has This Been Tested?

- One TcUnit smoke test per device (`FB_KBOTest`, `FB_Coating_States_NoPMPSTest`),
  built the same way as the existing `FB_REFTest`/`FB_SATTTest`: a
  simulated `AXIS_REF` from `FB_MotionStageNC`, a fake PMPS JSON DB seeded
  through `FB_CC_PMPSTestSetup` (KBO only), commanded state changes, and
  checks on position and `eEnumGet` within a timeout. No real hardware
  involved.
- Built in real TwinCAT XAE, which caught one real compile error that
  static checks missed: both new state enums defaulted to TwinCAT's
  implicit `INT` base type, which breaks the `REFERENCE TO UINT` bridge
  `FB_PositionStatePMPSND`/`FB_PositionStateND` expect
  (`C0201: Type 'INT' is not equal to type 'UINT'`). Fixed by giving both
  enums an explicit `UINT` base type, matching the existing
  `E_REF_States.TcDUT`.
- `FB_Coating_States_NoPMPSTest` passes. `FB_KBOTest` has gotten past
  wiring issues (PMPS DB lookup, PLC-to-NC axis linking for the two new
  sim axes); a full pass/fail still needs one more run with the PMPS DB
  load turned back on.

## Where Has This Been Documented?

- Docstrings on every new FB, method, and enum, explaining the parts that
  aren't obvious from the code (why `FB_CC_StatePMPSEncConfig` exists
  instead of changing its sibling, why `fPosition` isn't a parameter, why
  the device FB doesn't set motion parameters).
- `docs/common-components-library-migration.html`: a Confluence page
  (titled "Common Components Library Migration"), based on
  `docs/migration.md`, covering the ownership contract, the two
  conventions, the raw-counts helper, a step-by-step porting guide, and
  which devices still need migrating.

## Pre-merge checklist
- [ ] Code works interactively
- [ ] Code contains descriptive docstrings, including context and API
- [ ] `FB_KBOTest` passes end-to-end with the PMPS DB load turned on
- [ ] Migration guide reviewed by someone who didn't write it

