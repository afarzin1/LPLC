# LPLC — Roadmap

> **Living document.** Update it in the same change as the work. Status: ✅ done · 🟡 in progress ·
> ⬜ todo · 🔲 needs a decision. Release history lives in [`CHANGELOG.md`](CHANGELOG.md); the rules
> every change follows live in [`CONVENTIONS.md`](CONVENTIONS.md).

## Where the library stands

The architecture is the asset: every device extends one base with the same `.HMI` struct,
interlock, TcEventLogger alarms and fault-monitor registration, which is what lets
[lhmi](https://github.com/afarzin1/LHMI) faceplates work across machines without per-device UI
work. The newer blocks (`fb_Axis`, `abs_fb_MachineFlow`, `fb_Permissive`, the cam blocks,
`fb_Modbus_Driver`) are careful and documented.

What it lacks is the unglamorous part. There are no automated tests and no compile gate, failure
handling (comms loss, stale data, missing IO) is the weakest area, and the older device blocks
predate the conventions. The roadmap is ordered to fix that first.

## 1. Verification

- [ ] ⬜ **TcUnit tests for the pure logic.** `fb_FaultMonitor`, `fb_Permissive`, `fb_Interlock`,
      `fb_RampGenerator`, `fb_PID_Clamping`, `fb_TimeAverage`, the Modbus driver's address
      parsing, block optimisation and word-order encoding, and `abs_fb_MachineFlow`'s token engine.
      Every bug found in the 2.1.0 review would have been caught by a test or a compile.
- [ ] ⬜ **Compile gate in CI** on a Windows agent with TwinCAT XAE: build `LPLC.sln` on every push
      and pull request, fail on errors and new warnings. (Shared with lhmi's "TwinCAT CI" item.)
- [ ] ⬜ **Hardware validation of the never-run blocks.** `abs_fb_MachineFlow` and `fb_Permissive`
      are documented as "compiled, never run"; `fb_CamTeach`, `fb_CamTeachFile` and
      `fb_CamCoupler` carry `VERIFY` markers against the installed Tc2_MC2 / TF5050 version. Run
      each on a test rig, resolve the markers, and record the result in its design note. 🟡 2.2.0
      reworked the cam download (`MC_CamTableSelect` creates the table) and checked the motion
      function fields against the TF5050 manual; a full teach → download → verify → couple run on
      hardware is still open.
- [ ] ⬜ **Hardware test of the 2.2.0 `fb_Axis` changes:** retained home across an NC restart
      (absolute encoder), live pendant jog (`MC_MoveVelocity` re-trigger and `MC_Halt` release),
      the NC-max jog clamp, and `eModulo` moves in each direction.
- [ ] ⬜ **Hardware test of the reworked comms.** `fb_Dev_PSU_ITECH` (latch on first failure,
      baseline before write) on an IT6000 and an IT6600C, and `fb_Modbus_Driver`'s reconnect /
      Faulted / `cmdReset()` path on an ABB ACS580 and a DuraPulse drive. Confirm the ITECH ON/OFF
      word order against the supply manual (see the comment in `writeImage`).

## 2. IO module health monitoring

Today nothing watches the IO modules. A missing terminal, a wrong module in a slot, or a terminal
that drops out of OP leaves the program running on stale or zero process data. The design:

**Detection — `fb_IO_EtherCATMonitor`, one per EtherCAT master.** Extends `abs_fb_Device`, so its
faults are alarmed, latched and listed in `fb_FaultMonitor` like any device.

- *Fast path, same scan:* linked inputs from the master's InfoData — `DevState`, and the `WcState`
  bit of every sync unit (an array input). A terminal that drops off or stops exchanging data
  flips its sync unit's `WcState` in the cycle it happens.
- *Detail path, slower (ADS, ~500 ms, Tc2_EtherCAT):* `FB_EcGetAllSlaveStates` for every slave's
  AL state and link status, so a fault names the module (address, position, type) instead of
  just "sync unit 3".
- *Configuration check, at startup:* `FB_EcGetConfSlaves` against `FB_EcGetScannedSlaves`, compared
  by position and product code. A missing, extra or wrong module is a configuration fault with
  the slot in the text, and it blocks the machine from starting.
- *Outputs:* `stsAllOK`, `stsConfigOK`, `stsFaultCount`, first-fault text, `isSyncUnitOK(i)` and
  `isSlaveOK(addr)` methods, and an `HMI` struct with the per-slave list.

**Enforcement — through the interlock mechanism devices already have.** No new concept:

- `abs_fb_Device` (and `abs_fb_stateDevice` until the bases are unified) gets
  `inpIOOK : BOOL := TRUE`. `DeviceLogic` folds it into the interlock as a reserved slot 10,
  "IO module OK". A device whose IO is gone therefore stops exactly like one with any broken
  interlock: it faults, its outputs go to their de-energised state, and the faceplate's interlock
  list says why. The default TRUE leaves every existing machine unchanged. (Devices use slots 1–4
  today; slot 10 is reserved from here on.)
- The machine wires it — library blocks stay self-contained (CONVENTIONS §5):
  `Pump1(inpIOOK := EcMon.isSyncUnitOK(2), ...)`. Grouping each device's terminals into its own
  sync unit makes one `WcState` bit exactly "this device's IO is valid this cycle".
- `fb_Machine` gets the same input, wired to `EcMon.stsAllOK`. A machine with an IO fault cannot
  start or enter auto, and a configuration mismatch keeps it from starting at all.

**Display.** lhmi already draws the EtherCAT chain (EcDiagnostic, read straight from the master
over ADS). The PLC monitor adds what lhmi cannot know on its own: which devices are inhibited
because of IO. Each faceplate shows it through the interlock slot it already renders.

Safety stays where it is: EtherCAT outputs fall to their safe values on watchdog, and safety
functions belong in TwinSAFE. This layer is about the standard program noticing, stopping cleanly
and saying which module.

- [ ] ⬜ **`fb_IO_EtherCATMonitor`** (fast path, detail path, configuration check, HMI struct).
- [ ] ⬜ **`inpIOOK` + reserved interlock slot 10** in the device base classes.
- [ ] ⬜ **`fb_Machine` integration** (start / auto gating, configuration fault).
- [ ] ⬜ **lhmi faceplate** for the monitor's per-slave list, next to EcDiagnostic.

## 3. Architecture

- [ ] ⬜ **One device base.** `abs_fb_stateDevice` should extend `abs_fb_Device`. Today the alarm,
      fault and interlock plumbing exists twice and has already drifted.
- [ ] ⬜ **One pattern for wired-in IO.** Blocks dereference `POINTER TO` inputs without checks,
      so leaving one unwired stops the PLC. Move to `REFERENCE TO` with `__ISVALIDREF`, or guarded
      pointers, consistently. Unguarded today: `fb_Dev_Pump_SPAL` (4), `fb_Dev_IsolationMonitor`
      (3), the INT/REAL control-valve wrappers, the stack light, and the step-variable pointer in
      `abs_fb_stateDevice`.
- [ ] 🔲 **Core vs devices.** The framework (device bases, interlock, permissive, fault monitor,
      MachineFlow, axis, Modbus driver, IO monitor) could be held to a higher standard than one-off
      vendor drivers. Options: a separate optional device library, or a `Devices/` examples tier.
- [ ] ⬜ **Interlock evaluated once.** Several devices call `Interlock(...)` again after
      `DeviceLogic()` already did, without the bypass input. Remove the second call.
- [ ] ⬜ **`cfgEnable` that does something.** Most devices ignore `_enabled`. Define what disabled
      means (no outputs, no faults, no comms) and implement it in the base.

## 4. Comms

- [ ] ⬜ **Move the remaining hand-rolled Modbus devices onto `fb_Modbus_Driver`:**
      `fb_Dev_WebRelay`, `fb_Dev_IsolationMonitor`. Comms-loss behaviour then lives in one place,
      and `DUTs/Comm/ST_Modbus_Commands` / `ST_Modbus_Device_Config` can go.
- [ ] ⬜ **`fb_IO_EL6751_Bus_Status`:** the bus is flagged faulted on any status bit, including
      "sync toggle". Confirm on hardware whether that bit toggles in normal operation; if so,
      mask it.

## 5. Smaller fixes from the 2.1.0 review

- [ ] ⬜ `fb_Dev_AnalogSensor`: the scaled path uses last scan's raw value (one-scan lag).
- [ ] ⬜ `fb_Dev_AnalogSensor_ELM_10x`: `stsEngValueRMS` is never assigned.
- [ ] ⬜ `fb_Machine`: event-log export runs once with method-local settings, and the clear passes
      FALSE to what looks like a trigger input. Verify both against Tc3_EventLogger and fix.
- [ ] ⬜ `fb_SampleBuffer`: the header claims `stsTotalSamples` survives a PLC restart; it does not.
- [ ] 🔲 `F_AddToArray_String` / `F_RemoveFromArray_String` have no users since the fault-monitor
      rewrite. Keep as utilities or remove.

## Done

- [x] ✅ **2.2.0** (see `CHANGELOG.md`): `fb_Axis` retained home and operator settings, pendant
      mode and live pendant jog, NC-max jog clamp, `eModulo` moves and halt / jog-error fixes; cam
      download rework; CONVENTIONS §13 machine project structure.

- [x] ✅ **2.1.0 — first open-source release** (see `CHANGELOG.md`): renamed to LPLC, test-stand
      blocks removed, TwinCAT HMI leftovers and dead types removed, `fb_` naming throughout,
      `fb_Dev_PSU_ITECH` on `fb_Modbus_Driver` with a comms latch, and the review fixes (driver
      failure handling, fault monitor, axis jog latch, SPAL fault latch, actuator retract,
      isolation-monitor comms check, stack-light HV segment, `setFault` duplicates,
      `cfgCalGain` default).
