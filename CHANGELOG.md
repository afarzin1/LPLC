# Changelog

All notable changes to LPLC are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project uses
[Semantic Versioning](https://semver.org/). Changes to any `.HMI` struct are listed explicitly,
because they change the contract HMIs bind to.

## [2.2.0] — 2026-10-18

### `.HMI` contract

- `ST_Axis_HMI_Status` gains `stsHomeRetained`, `stsPendantMode` and `stsPendantDeadman`
  (additive, existing bindings keep working).
- `ST_Axis_HMI_Status.stsErrorID` now reports the last `MC_*` block error (refused jog, move,
  home…) when the NC axis itself has no error, so the HMI's NC error lookup can describe a refused
  command. It is cleared by `cmdReset`. Same type, different meaning when the axis error is 0.

### `fb_Axis`

- **Retained home.** A taught home (`cmdHome`, MC_Direct) is stored as a `VAR PERSISTENT` offset
  from the raw encoder position and re-applied after an NC restart (`MC_SetPosition`, then
  `MC_ForceCalibration`, no motion). Needs an absolute encoder; new `cfgRetainHome` (default TRUE)
  turns it off for an incremental one.
- **Retained operator settings.** Jog velocity, jog acceleration and velocity override survive a
  restart and are saved as soon as an operator edits them. A value set from PLC code is no longer
  reverted by a stale HMI field.
- **Pendant mode.** New `inpPendantMode`: selecting the pendant locks out HMI jog, torque-jog and
  go-to straight away; `inpPendantEnable` stays the dead-man that motion needs.
- **Pendant jog follows the stick live.** The pendant jog runs on a re-triggered `MC_MoveVelocity`
  and stops with `MC_Halt`, instead of `MC_Jog` (which takes its speed only at the start of a
  jog). New `inpPendantJogFraction` (0..1) scales the pendant speed after the NC-max clamp.
- **Jog speed clamped to the NC maximum velocity**, read once with `MC_ReadParameterSet` (retried
  every 2 s until it succeeds). A jog above it used to be refused by the NC and the axis did not
  move. New output `stsNcMaxVelocity`.
- **`eModulo` moves** (`MC_MoveModulo` with the move's `direction`): an absolute target within one
  modulo period, in a forced direction.
- Fixes:
  - `MC_Jog` errors fault the axis (they were silent).
  - A move whose mode has no implementation faults instead of hanging in execution with
    `stsMoveDone` never set.
  - The halt request clears when `MC_Halt` is aborted or errors, so the next `cmdHalt` always
    halts (an axis handed back from external control could keep running).

### Camming (not yet verified on hardware)

- `fb_CamTeach`'s download **creates** the NC table with `MC_CamTableSelect` (filled from the
  taught points, periodicity set there) instead of `MC_WriteMotionFunction`, which only rewrites a
  table already in the NC. The table id must not also be a System Manager table.
- The motion function sets `PointType := MOTIONPOINTTYPE_MOTION` (the default `IGNORE` made the NC
  reject the table as having too few points), uses `POLYNOM3_VV` segments (plain `POLYNOM3` stops
  the slave at every point), and the last point links to no next point. `MC_CAM_REF` is sized to
  the points written.
- `fb_CamCoupler` no longer selects a table; it engages `tableId` with `MC_CamIn`. The `periodic`
  input is informational and `E_CamCoupler_State.eSelecting` is no longer entered.

### Docs

- `CONVENTIONS.md` §13: how a machine project using LPLC is laid out (folders, the five GVLs,
  `MAIN`, axis wiring, sequences, HMI contract).

## [2.1.0] — 2026-10-10

First open-source release, under the MIT License. Versions before 2.1.0 were not publicly released.

- The library is named **LPLC**: title, company and default namespace are `LPLC` (consumers write
  `LPLC.fb_Axis`), the solution is `LPLC.sln`.
- Added `LICENSE`, `README.md`, `AGENTS.md`, `CONTRIBUTING.md`, `SECURITY.md` and `ROADMAP.md`.
- Data types are organised in folders that mirror the POU tree, one folder per FB named exactly
  like it (`DUTs/Devices/fb_Dev_XPendant/`). Folders have no effect on type names.
- `fb_Dev_PSU_ITECH` talks to the supply through `fb_Modbus_Driver` (one connection, requests in
  series). The first failed read or write stops all traffic to the supply and faults the device
  until a reset; writes are held after every (re)start until the supply's current settings have
  been read back, and a restart after a comms fault commands the output off. New:
  `stsCommFaulted`, `cfgBaselineCurrentLimit` (default 20.5 A, the previous hard-coded value).
  The old public `runModbusComms()` method is gone. No `.HMI` struct changed.
- `fb_Modbus_Driver` gains `inpHoldWrites` (poll reads, start no write cycle; default FALSE) and
  `cmdReset()`. Its failure handling now works as documented: after `maxConsecFails` failures it
  drops the link and retries after `reconnectDelay`, and after `maxReconnectAttempts` it stops in
  Faulted until `cmdReset()` or an enable toggle (previously it never reconnected or faulted, and a
  Faulted driver could not be recovered without a PLC restart). `timeSinceLastGoodRead` only resets
  on a read cycle in which every block succeeded. The VFD blocks' `cmdReset` also resets a
  faulted driver.
- Fixes:
  - `fb_Axis`: a jog or torque-jog held across the manual-to-auto switch no longer stays latched
    in Auto (PLC-started jogs via `cmdJogPositive`/`cmdJogNegative` are unaffected).
  - `fb_FaultMonitor` rebuilds its fault lists every scan, so cleared faults leave the list and the
    device and description lists stay aligned. New `stsFaultCount`.
  - `fb_Dev_Pump_SPAL`: faults latch until reset (a "Normal" pump status used to clear them), and a
    faulted pump is not commanded to run.
  - `fb_Dev_Actuator`: `cmdRetract` works with the interlock not met.
  - `fb_Dev_IsolationMonitor`: a failed Modbus read faults the device and is not parsed; new
    `stsCommOK`.
  - `fb_Dev_StackLight3SegHV_BannerTL50Pro`: the HV segment follows only `inpHVOn`.
  - `abs_fb_Device.setFault`: every distinct additional fault is reported, not only the ones
    before the first repeat.
  - Analog sensors: `cfgCalGain` defaults to 1.

[2.2.0]: https://github.com/afarzin1/LPLC/releases/tag/v2.2.0
[2.1.0]: https://github.com/afarzin1/LPLC/releases/tag/v2.1.0
