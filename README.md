# LPLC

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**LPLC** is a TwinCAT 3 PLC library of standardized device, motion and machine function blocks.
It is the PLC-side companion to [**lhmi**](https://github.com/afarzin1/LHMI), an LLM-first web HMI
platform: every LPLC device exposes one `.HMI` struct, and lhmi's faceplates bind to exactly that
shape. LPLC works on its own as well; nothing in it depends on lhmi.

> **Using an AI coding agent** (Claude Code, Codex, Cursor, Copilot, Gemini CLI, …)?
> [`AGENTS.md`](AGENTS.md) is the entry point. Agents that follow the
> [AGENTS.md](https://agents.md) convention load it automatically.

## What's in it

| Area | Blocks |
|---|---|
| **Device base** | `abs_fb_Device` (faults, interlocks, TcEventLogger alarms, the `HMIbaseDevice` block), `abs_fb_stateDevice` (sequenced devices with a start permissive) |
| **Devices** (`fb_Dev_*`) | actuators, analog sensors (incl. ELM accelerometers and current transducers), modulating control valves, VFDs (ABB ACS580, DuraPulse), pumps, power supplies, isolation monitor, stack and strip lights (Banner), web relay, Xbox-controller pendant |
| **Motion** | `fb_Axis` (moves, jog, interlocks), camming: `fb_CamTeach`, `fb_CamTeachFile`, `fb_CamCoupler` |
| **Machine logic** | `fb_Machine`, `fb_Interlock`, `fb_Permissive`, `fb_FaultMonitor`, `abs_fb_MachineFlow` (PLC-defined sequence graphs rendered by lhmi) |
| **Data** | `fb_SampleBuffer` (oversampled rings for FFT), `fb_WriteDataToFile` |
| **Comms** | `fb_MQTT_Connector`, `fb_Modbus_Driver` |
| **Utilities** | `fb_PID_Clamping`, `fb_RampGenerator`, `fb_TimeAverage`, `fb_HmiEventLog`, analog scaling and string functions |

Design notes for the larger blocks are in [`Docs/`](Docs/).

## Requirements

- TwinCAT 3.1 XAE, build 4026 or newer.
- The Beckhoff libraries the project references (resolved by TwinCAT, not shipped here):
  `Tc2_MC2`, `Tc2_MC2_Camming`, `Tc2_ModbusSrv`, `Tc2_Standard`, `Tc2_SUPS`, `Tc2_System`,
  `Tc2_Utilities`, `Tc3_EventLogger`, `Tc3_IotBase`, `Tc3_JsonXml`,
  `Tc3_Module`. Some of them belong to TwinCAT functions (TF) that need their own licenses on the
  target.

## Using it

1. Open `LPLC.sln` in TwinCAT XAE and build `PLCLibrary`.
2. Either reference the project directly from your machine solution, or save it as a library
   (*PLCLibrary → Save as library and install*) and add the installed **LPLC** library to your
   project's references.
3. Use the `LPLC` namespace: `Devices.Pump1 : LPLC.fb_Dev_Pump_SPAL;`.

Declare device instances in a `Devices` GVL and call them from `MAIN`. That layout, the `.HMI`
pattern, and every naming rule are in [`CONVENTIONS.md`](CONVENTIONS.md), which is required reading
before adding or renaming anything.

## Conventions in brief

- No Hungarian notation. Member prefixes encode **role**: `inp` inputs, `sts` status outputs, `cfg`
  configuration properties, `ocmd` operator commands, `cmd` command methods, `ref` references,
  `_` internal state, `c` constants.
- Every device carries exactly one `HMI : ST_<Device>_HMI_Interface` member. HMIs bind
  `<instance>.HMI.*` and nothing else.
- Library FBs are self-contained: no GVL references; dependencies come in through the interface.

## Roadmap

What is planned next, and the known issues not yet fixed, are in [`ROADMAP.md`](ROADMAP.md).

## Versioning

LPLC uses one SemVer number, set in `PLCLibrary/PLCLibrary.plcproj` (`<ProjectVersion>`) and
mirrored in `Version/Global_Version.TcGVL`. The major version is aligned with lhmi: lhmi's
`shared/v2` mirrors the `.HMI` interfaces of LPLC v2. A change to any `.HMI` struct is a contract
change for the HMI and is called out in [`CHANGELOG.md`](CHANGELOG.md).

## Safety

LPLC is provided "AS IS", without warranty of any kind (see [`LICENSE`](LICENSE)). It is not a
safety-rated library. Safety functions belong in certified safety hardware and logic (for example
TwinSAFE), and you are responsible for validating every block on your machine before use.

## License

Copyright 2026 Arya Farzin. Licensed under the [MIT License](LICENSE). Contributions are welcome;
see [`CONTRIBUTING.md`](CONTRIBUTING.md). To report a vulnerability, see [`SECURITY.md`](SECURITY.md).

Beckhoff, TwinCAT, TwinSAFE and EtherCAT are trademarks of Beckhoff Automation GmbH. Other product
names (ABB, Banner, DuraPulse, ITECH, SPAL and others) are trademarks of their respective owners
and are used only to identify the devices a block supports. LPLC is not affiliated with or endorsed
by any of them.
