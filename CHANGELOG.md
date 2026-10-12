# Changelog

All notable changes to LPLC are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project uses
[Semantic Versioning](https://semver.org/). Changes to any `.HMI` struct are listed explicitly,
because they change the contract HMIs bind to.

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

[2.1.0]: https://github.com/afarzin1/LPLC/releases/tag/v2.1.0
