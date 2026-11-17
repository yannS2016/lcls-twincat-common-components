# Migrating a PLC project to the ND motion library

Based on `lcls-plc-kfe-motion` (commits `d18ab0b`..`00471b6`). Library-side interface: [migration.md](migration.md).

**Idea:** the PLC project now owns its motion primitives. Each axis is one `FB_MotionStageNC`, all parameters live in one `FB_PersistentDataStorage`, and state tables are filled by configurator FBs. Library device FBs receive those by reference.

Needs TwinCAT 4026, `LCLS_Tc_Motion`, `LCLS_Tc_DevAbs`, and a common-components release that includes the slit bridge gating.

## 1. References (`.plcproj`)

| Old | New |
|---|---|
| `lcls-twincat-motion` | `LCLS_Tc_Motion` + `LCLS_Tc_DevAbs` |
| `PMPS` (older) | `PMPS` 3.15.0+ (`FB_FallibleRealFromEtherCAT`, used by slits) |
| `Temp/FB_SLITS`, `Temp/FB_SLITS_POWER` local copies | deleted; use the library FBs |

Also removed here: the `MAX_STATES`, `MAX_STATE_MOTORS` and `TRANS_MARGIN` placeholder parameters, so library defaults apply. Check your state-array bounds and pytmc `array:` pragmas still match.

## 2. Axis declarations

| | Old | New |
|---|---|---|
| Where | `Main.TcGVL`, one `ST_MotionStage` per axis | `GVL_Axes.Axes : ARRAY[1..33] OF AXIS_REF`, plus an `FB_MotionStageNC` in each device PRG |
| Name | `Main.M33` | `GVL_Axes.Axes[33]` (same number, so NC "Axis 33" is unchanged) |
| IOC link | `pytmc: pv: ...` on `Main.M33` | `axis-link: GVL_Axes.Axes[33]` on the stage |

Old:
```iecst
// Main.TcGVL
{attribute 'pytmc' := 'pv: ST1K0:MMS:01'}
{attribute 'TcLinkTo' := '.bLimitForwardEnable := TIIB[ST1K0-EL7041-E4]^STM Status^Status^Digital input 1;
                          .bBrakeRelease       := TIIB[ST1K0-EL2202-E3]^Channel 1^Output;
                          .nRawEncoderULINT    := TIIB[ST1K0-EL5042-E6]^FB Inputs Channel 1^Position'}
M33: ST_MotionStage := (sName := 'ST1K0:MMS:01');
```
New:
```iecst
// PRG_ST1K0
{attribute 'pytmc' := 'pv: ST1K0:MMS:01
                       io: io
                       axis-link: GVL_Axes.Axes[33]'}
{attribute 'TcLinkTo' := '.fbLimSwStatus.bLimitFwd := TIIB[ST1K0-EL7041-E4]^STM Status^Status^Digital input 1;
                          .fbMotionAxis.fbBrake.bBrakeRelease := TIIB[ST1K0-EL2202-E3]^Channel 1^Output;
                          .fbEncoderScaling.nRawEncoderULINT := TIIB[ST1K0-EL5042-E6]^FB Inputs Channel 1^Position'}
fbMotor: FB_MotionStageNC(AxisRef := GVL_Axes.Axes[33], iPersistentDataStorage := GVL.fbPDS, sName := 'Y');
```

Link renames: `bLimitForwardEnable` -> `fbLimSwStatus.bLimitFwd`, `bLimitBackwardEnable` -> `fbLimSwStatus.bLimitBwd`, `bHome` -> `fbLimSwStatus.bHomeSensor`, `bBrakeRelease` -> `fbMotionAxis.fbBrake.bBrakeRelease`, `nRawEncoder*` -> `fbEncoderScaling.nRawEncoder*`. Unlinked limits default to TRUE.

Keep the `axis-link:` element and `AxisRef` identical. For arrays (slit blades) keep both lists in the same order.

## 3. Shared globals and the task PRG

| Old | New |
|---|---|
| (none) | `GVL.fbPDS: LCLS_Tc_DevAbs.FB_PersistentDataStorage;` and `GVL.fbPDS();` first in `PRG_1_PlcTask` |
| `MOTION_GVL.fbStandardPMPSDB(io_fbFFHWO := GVL.fbFastFaultOutput1, bEnable := TRUE, sPLCName := 'plc-kfe-motion');` in `PRG_2_PMPS_PRE` | `GVL.fbStandardPMPSDB: FB_Standard_PMPSDB(sPlcName := 'plc-kfe-motion', io_fbFFHWO := fbFastFaultOutput1);` in `GVL`, then `GVL.fbStandardPMPSDB(bEnable := TRUE);` |

## 4. Motor setup and call

Old: per-cycle assignments, then pass the struct.
```iecst
Main.M33.nEnableMode := ENUM_StageEnableMode.DURING_MOTION;
Main.M33.bHardwareEnable := TRUE;
Main.M33.bPowerSelf := TRUE;
Main.M33.nBrakeMode := ENUM_StageBrakeMode.IF_ENABLED;
fbMotion(stMotionStage := Main.M33);
```
New: one call on the first scan, then tick the stage.
```iecst
IF bFirstScan THEN
    fbMotor.SetModeEnableLimits(LimBackward := TRUE, LimForward := TRUE, HomeEnable := FALSE,
        BrakeMode := E_StageBrakeMode.IF_ENABLED, EnableMode := E_StageEnableMode.DURING_MOTION,
        HomeMode := E_EpicsHomeCmd.NONE, UserEnable := TRUE, HardwareEnable := TRUE);
    fbMotor.SetVelocity(Velocity := 1);
    bFirstScan := FALSE;
END_IF
...
fbMotor();
```
- Enums renamed `ENUM_*` -> `E_*`.
- Velocity is now per stage (`SetVelocity`, or `SetMotionParam` for slits). `fVelocity` is gone from the state defaults.
- Stages with dynamic limits or locks (IM1K0/IM1K4 lenses) also call `SetLimitsAndEnables(...)` every cycle.

## 5. State tables

Old: flat array, helper called every cycle, state mover takes the motor struct.
```iecst
fbStates: FB_PositionState1D;
fbStateSetup: FB_StateSetupHelper;
astPositionState: ARRAY[1..GeneralConstants.MAX_STATES] OF ST_PositionState;
...
fbStateSetup(stPositionState := astPositionState[E_EpicsInOut.IN],  sName := 'IN',  fPosition := 3);
fbStateSetup(stPositionState := astPositionState[E_EpicsInOut.OUT], sName := 'OUT', fPosition := 40.9);
fbStates(stMotionStage := Main.M33, astPositionState := astPositionState,
         eEnumSet := eEnumSet, eEnumGet := eEnumGet, bEnable := TRUE);
```
New: `[motor][state]` table, configurators declared once and applied at start-up, state mover bound at declaration.
```iecst
aiMotionStage: ARRAY[1..MotionConstants.MAX_STATE_MOTORS] OF I_MotionStage := [fbMotor];
astPositionState: ARRAY[1..MotionConstants.MAX_STATE_MOTORS]
                    OF ARRAY[1..GeneralConstants.MAX_STATES] OF ST_PositionState;
afbStateConfig: ARRAY[1..2] OF FB_StateConfigurator[
    (stPositionState := astPositionState[1][E_EpicsInOut.IN],  stDefault := stDefault, bSetDefault := TRUE, sName := 'IN',  fPosition := 3),
    (stPositionState := astPositionState[1][E_EpicsInOut.OUT], stDefault := stDefault, bSetDefault := TRUE, sName := 'OUT', fPosition := 40.9)
];
fbStates: FB_PositionStateND(aiMotionStage := aiMotionStage, astPositionState := astPositionState,
    nActiveMotorCount := 1, eEnumSet := eEnumSet, eEnumGet := eEnumGet, bEnableMotion := TRUE,
    eMotionRequestPolicy := E_MotionRequestPolicy.INTERRUPT);
...
fbStates();
```
The pytmc state array also becomes two-dimensional: `array: 1..1 | 1..2` with `expand-names: Y`.

| State needs | Old | New |
|---|---|---|
| Name + position | `FB_StateSetupHelper` | `FB_StateConfigurator` |
| Name + position + PMPS key | `FB_StateSetupHelper` (`sPmpsState`) | `FB_CC_StatePMPSConfig` |
| Shared defaults only | `FB_StateSetupHelper` (`bSetDefault`) | `FB_CC_StateDefaultsConfig` |

## 6. Library device FBs (imagers, slits, ...)

Imager, old vs new:
```iecst
// old
fbStateSetup(stPositionState := fbIM1K0.stYag, fPosition := -27.65, sPmpsState := 'IM1K0:XTES-YAG');
fbIM1K0(stYStage := Main.M1, stZoomStage := Main.M2, stFocusStage := Main.M3, ...);

// new
afbStatePMPS: ARRAY[1..4] OF FB_CC_StatePMPSConfig[
    (stPositionState := astPositionState[1][E_XPIM_States.YAG], stDefault := stDefault,
     fPosition := -27.65, sPmpsState := 'IM1K0:XTES-YAG'), ... ];
fbIM1K0: FB_XPIM(aiMotionStage := aiMotionStage, astPositionState := astPositionState,
    fbFFHWO := GVL.fbFastFaultOutput1, fbArbiter := GVL.fbArbiter,
    sDeviceName := 'IM1K0:XTES', sTransitionKey := 'IM1K0:XTES-TRANSITION',
    bFileReaderBusy := MOTION_GVL.fbPmpsFileReader.bBusy, bFileReaderError := MOTION_GVL.fbPmpsFileReader.bError, ...);
```
Zoom/focus (non-state motors) are no longer passed in; the PRG declares and ticks them itself.

Slits, old vs new:
```iecst
// old
fbSL1K0(stTopBlade := Main.M19, stBottomBlade := Main.M21, stNorthBlade := Main.M20, stSouthBlade := Main.M18, ...);

// new: blades are an array, indexed by E_SlitsBlade
afbBlade: ARRAY[1..4] OF FB_MotionStageNC[ (AxisRef := GVL_Axes.Axes[19], iPersistentDataStorage := GVL.fbPDS, sName := 'Top'), ... ];
aiMotionStage: ARRAY[1..4] OF I_MotionStage := [afbBlade[E_SlitsBlade.TOP], afbBlade[E_SlitsBlade.BOTTOM],
                                                afbBlade[E_SlitsBlade.NORTH], afbBlade[E_SlitsBlade.SOUTH]];
fbSL1K0: FB_SLITS_POWER(aiMotionStage := aiMotionStage, fbFFHWO := GVL.fbFastFaultOutput1,
                        sDeviceName := 'SL1K0:POWER', sPmpsState := 'SL1K0:POWER-RTD', ...);
// each cycle
fbSL1K0.bBridgeOK := GVL.fbArbiterIO.bBridgeOk;
fbSL1K0.bBridgeConfirmedBad := GVL.fbArbiterIO.bBridgeConfirmedBad;
fbSL1K0(); FOR i := 1 TO 4 DO afbBlade[i](); END_FOR
```
Aperture requests are trusted only while the bridge is OK and the request `xOK` is set. NaN/Inf requests are sanitized.

Power meter, old vs new:
```iecst
// old
.fbThermoCouple.iRaw := TIIB[PF1K0-EL3314]^TC Inputs Channel 1^Value;
fbPowerMeter();
// new
.fbTempSensor.iRaw := TIIB[PF1K0-EL3314]^TC Inputs Channel 1^Value;
fbPowerMeter(fbFFHWO := GVL.fbFastFaultOutput4, bVetoTempFFO := TRUE);   // veto keeps the old no-thermal-fault behaviour
```
Custom device FBs (`FB_XPIM_IM2K0`): rebuild on `FB_PositionStatePMPSND`, take dependencies as `REFERENCE TO` via `FB_Init`, write per-state defaults in `CallAfterInit` with `FB_CC_InternalStateConfig`.

## 7. Cut-over checklist

1. Swap references, delete `Temp/` library copies, resolve placeholders.
2. Add `GVL_Axes`, `GVL.fbPDS`, `GVL.fbStandardPMPSDB`; delete `Main.TcGVL`.
3. Convert one PRG at a time (section 4, 5, 6) and compile after each.
4. Link each `Axes[n]` to NC "Axis n".
5. Regenerate `.tmc`, then the IOC database and archive files with pytmc, and diff the PV list against the old one.
6. Check on hardware or simulation: stages enable, limits report, states move, PMPS keys resolve, slits gate on bridge loss.

## Simulation variant (same branch, optional)

- IO tree cut to Device 1 -> EK1200 -> `PMPS_PRE`; PLC links to removed terminals dropped.
- NC axes set to `EncType 1` / `DrvType 5` (simulation pair); hardware IO mappings, scaling, offsets, soft limits and polarity inversion removed.
- Project resolves common-components to `0.0.0` with `Released=false`. Restore a release version before merging.
