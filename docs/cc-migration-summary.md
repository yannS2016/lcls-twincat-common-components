# CC library migration — handoff summary

Branch: `feature/motion-nd-migration` (HEAD `c99c4c6`), 55 commits over `master`.
Full plan and rules live in [docs/migration-motion-nd.md](docs/migration-motion-nd.md).
This file is the short orientation to pick up in a fresh chat.

## Goal

Move every motion consumer in `lcls-twincat-common-components` off the old
pre-ND surface (`FB_MotionStage`, `ST_MotionStage` via `VAR_IN_OUT`,
`FB_PositionStatePMPS1D`, `FB_PositionState1D`) onto the new
`LCLS_Tc_Motion` (ND) master surface (`FB_MotionStageNC`, `I_MotionStage`,
`FB_PositionStatePMPSND`, `FB_PositionStateND`, `FB_StateConfigurator`).

## What changed (the big strokes)

- **Library reference swapped** to `LCLS_Tc_Motion, * (SLAC - LCLS)`,
  namespace `LCLS_Tc_Motion`. It is a renamed library, not a version bump
  (placeholder, company, and namespace all change).
- **Dependency injection, not `VAR_IN_OUT`.** Devices own `FB_MotionStageNC`
  instances and pass `I_MotionStage` into the state mover via `FB_Init` /
  `CallAfterInit`.
- **2-D jagged position-state arrays** `[motor][state]` replace 1-D arrays.
- **Motion params leave the state.** Velocity/accel/decel now set per motor
  via `afbMotorsNC[i].SetMotionParam(...)` in the first-scan block; only
  `sName` and `fDelta` stay state-resident.
- **State setup via `FB_StateConfigurator`** (one per motor/state pair,
  apply-once gated by the `bUpdated` latch) replaces per-state
  `FB_PositionState_Defaults` calls.
- **Two helpers retired** (kept on disk with an obsolescence note, removed
  from `Library.plcproj` build): `FB_PositionState_Defaults` and
  `FB_CheckPositionStateWrite`. Both premises are gone under ND
  (`ST_PositionState` is read-only from EPICS; per-slot `bUpdated` latch
  replaces the concurrent-writer mediator).

## Status by phase

| Phase | Scope | State |
|---|---|---|
| 0 | Library reference swap | Done (`cc244b8`) |
| 1 | Retire in-library helpers | Done (both excluded from build) |
| 2 | Per-device migration (9 FBs) | Done in commits — REF, LIC, PPM, GBS, ATM, IPM, WFS, XPIM, SATT all migrated |
| 3 | SLITS family | Migrated (`5194cd5`); dispatch moved into `FB_SLITS_POWER` (`7a4f701`) |
| 4 | Tests rewritten | Done — all device tests + PMPS fake-DB harness (`FB_CC_PMPSTestSetup`) |
| 5 | Cleanup / release | **Not done** |

> Note: the checkboxes in `docs/migration-motion-nd.md` are stale — the
> commit history is the source of truth. Devices and tests are migrated;
> only Phase 5 remains.

## Device migration facts (confirmed in pre-flight)

- **R2 already satisfied everywhere** — every state enum has `Unknown`/`UNKNOWN`
  at index 0, `OUT` at index 1. No renumber, no API break.
- **State-select motor count is 1 for every device.** Multi-stage devices
  (ATM, IPM, WFS, XPIM) wire only the Y/state stage to `fbStates`; aux
  stages (X / Z / Zoom / Focus) stay as independent `FB_MotionStageNC`
  members.
- **SATT is the odd one out** — uses `FB_PositionStateND` (non-PMPS), does
  its own PMPS DB lookup and transmission math, raises its own fast-fault.

## New / reworked POUs

- `POUs/FB_CC_InternalStateConfig.TcPOU`, `POUs/FB_CC_StateDefaultsConfig.TcPOU`,
  `POUs/FB_CC_StatePMPSConfig.TcPOU` — split state setup into device-defaults
  + PMPS-config configurators.
- `Tests/Helpers/FB_CC_PMPSTestSetup.TcPOU` — per-device PMPS fake-DB harness,
  seeded via `call_after_init`, unique assertion IDs.
- `Devices/REF/E_REF_States.TcDUT`, `Devices/SLITS/E_SlitsBlade.TcDUT` — new enums.
- NC sim axes added: `_Config/NC/Axes/SLIT_{Top,Bottom,North,South}.xti`.

## Remaining work (Phase 5)

- [ ] Remove any temporary shims.
- [ ] Pin the `LCLS_Tc_Motion` library version in `Library.plcproj`
      (currently floating `* (SLAC - LCLS)`).
- [ ] Add the API-break notice to top-level `README.md`.
- [ ] Tag a release.
- [ ] Final gate: TcUnit `PRG_TEST` 0/0, static analysis 0 new findings,
      `Library.plcproj` builds standalone and via the tsproj.

## Constraints (carry into the new chat)

- Never push / open PRs (read-only deploy key). Commit only to
  `feature/motion-nd-migration`.
- Do **not** stage scratch files left in the worktree: `CLAUDE.md`,
  `FB_SLITS.TcPOU.txt`, `PRG_SL1L0_POWER.txt`, `findings.md`,
  `slits-ffo-handoff.md`, `slits-issue-summary.md`,
  `slits-silent-close-report.md`, `slits-report-img/`, and this file.
- Apply stop-slop to prose. Do not build/activate without asking.

## Separate open thread (not part of this migration)

The SL1K0 silent-FFO diagnosis is tracked in `slits-ffo-handoff.md` /
`findings.md`. Keep it out of the migration chat.
