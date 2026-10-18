# LPLC — Coding & Naming Conventions

**Read this before adding or renaming ANYTHING.** These rules were extracted from the
codebase 2026-07-18 (evidence counts in brackets) and resolve the splits that keep
causing rename churn. Where old code deviates, the rule below is the CURRENT intent —
follow the rule, don't imitate the deviation (§10 lists the known legacy offenders).
**§5** (FBs reach out to nothing) and **§12** (one PROGRAM; sequences under `MachineGlobal`) are
the two structural rules — read those even if you skip the naming tables.
Starting a new machine PLC project? **§13** is the layout: folders, the five GVLs, `MAIN`, axis
wiring, sequences and the HMI contract.

## 1. Type names

| Kind | Rule | Example |
|---|---|---|
| Function block | `fb_` + PascalCase | `fb_Interlock`, `fb_HmiEventLog` |
| Device FB (models a physical device) | `fb_Dev_<Device>`; lives in `POUs/FBs/Devices/` | `fb_Dev_Actuator`, `fb_Dev_XPendant` |
| Abstract base FB | `abs_fb_<Name>`, declared `FUNCTION_BLOCK ABSTRACT`; lives in `POUs/FBs/Abstract/` | `abs_fb_Device` |
| Struct | `ST_<Domain>_<Purpose>` | `ST_Axis_MoveDefinition` |
| HMI structs | `ST_<Device>_HMI_Interface` (+ `_HMI_Status` / `_HMI_Commands` / `_HMI_Config`) | `ST_Axis_HMI_Interface` |
| Enum | **`E_`** + `<Domain>_<Purpose>` — NOT `EN_` (legacy, 8 files) | `E_Axis_JogSource` |
| Union | `U_` + PascalCase | `U_ByteToLReal` |
| Function | `F_<Verb><Noun>`; lives in `POUs/FC/` | `F_AnalogToEngUnit` |
| Interface | `I_` + PascalCase (`.TcIO`, `POUs/Interfaces/`) | `I_DataSink` |
| Retired type | prefix `_del_`, keep the file (soft delete) | `_del_ST_FileManager_HMI_Interface` |

## 2. Member prefixes — role, NOT type

This library does **not** use Hungarian type prefixes (`b`/`n`/`r`/`s`/`t` — do not
introduce them). A member's prefix encodes its ROLE; its type is visible in the
declaration:

| Prefix | Role | Where |
|---|---|---|
| `inp` | input into the FB — **ALL `VAR_INPUT` and METHOD inputs** [188 uses; enforced by recent renames] | `inpAuto`, `inpMessage` |
| `sts` | status / readback (FB → consumer / HMI) — **ALL `VAR_OUTPUT`** [155, 92%] | `stsFaulted`, `stsSetpoint` |
| `cfg` | configuration — exposed as PROPERTY get/set (never a public VAR), seeded via `FB_Init` [139] | `cfgDeviceName`, `cfgJogVelocity` |
| `ocmd` | **o**perator command, HMI-writable (lives only in `_HMI_Commands` structs; edge-consumed by the FB) | `ocmdEnable`, `ocmdRunMove` |
| `cmd` | command METHOD (PLC-side callers) | `cmdReset()`, `cmdStart(inpMoveIndex)` |
| `ref` | pointer / reference input | `refFaultMonitor` |
| `_` | internal state (`VAR`) — composed FB instances and the `HMI` struct stay unprefixed PascalCase | `_wdLast`, but `Interlock : fb_Interlock` |
| `c` | `VAR CONSTANT` (`cRING_SIZE`) — not bare `UPPER_SNAKE` (legacy Modbus style) | `cBLOCK_SIZE` |
| `get`/`set`/`send`/`actions` | METHOD verbs: accessor / mutator / messaging / state-hook overrides | `getHMIdata`, `setFault`, `actionsRun` |

**Exception — raw external mirrors:** structs that mirror an external contract
byte-for-byte (vendor CAN/Modbus registers, an IPC service's write surface) use
**unprefixed PascalCase members** matching the external names (`ST_Pump_SPAL_CAN_*`,
`ST_XPendant_In`). Don't force role prefixes onto them — the mirror IS the contract.

## 3. The HMI pattern (every device)

- The FB carries exactly ONE member named **`HMI`** : `ST_<Device>_HMI_Interface`
  (plain `VAR`, not a pointer). HMI backends bind `<instance>.HMI.*` — nothing else.
- The interface struct nests: `HMIbaseDevice : ST_Device_HMI_Interface` (always first)
  + `Status` / `Commands` / `Config` sub-structs, **each its own DUT file**. Omit a
  group the device genuinely doesn't have (an input-only device has no `Commands`)
  rather than shipping an empty struct.
- Prefixes inside: `Status.sts*`, `Commands.ocmd*`, `Config.cfg*`.
- The FB body's LAST section publishes to HMI and starts with
  `HMI.HMIbaseDevice := HMIbaseDevice;` then fills `HMI.Status.*`.
- Commands are edge-consumed: `IF HMI.Commands.ocmdX THEN HMI.Commands.ocmdX := FALSE; cmdX(); END_IF`.
- Optional accessor: `METHOD PUBLIC getHMIdata : ST_<Device>_HMI_Interface` → `getHMIdata := HMI;`.

## 4. Device FBs extend `abs_fb_Device`

- `FUNCTION_BLOCK fb_Dev_X EXTENDS abs_fb_Device` (exact casing). The base gives you:
  `inpAuto/inpReset/inpInterlockBypass/faultMonitor` inputs, `stsFaulted/stsFaultDescription`
  outputs, `Interlock`, TcAlarm plumbing, `HMIbaseDevice`, and the `cfgDeviceName/cfgEnable/
  cfgInitEventClass` properties.
- **Call `DeviceLogic();` as the FIRST line of your body** (never override it; `SUPER^`
  is not used in this library). Raise faults via `setFault(state, description)`; add
  interlocks via `cmdSetInterlock(...)`; override `cmdReset` if you need reset behavior.
- Sequenced devices extend `abs_fb_stateDevice` instead (implement the `actions*` hooks,
  drive with `stateLogic()`). That base additionally gives you `permStartPermissive`, an
  `fb_Permissive` gating entry to Running — wire slots with `cmdSetStartPermissive(slot,
  description, condition)`, leave it alone and the device starts exactly as it always did
  (unwired slots default TRUE). **Use it for conditions that must hold to START; use
  `cmdSetInterlock` for conditions that must hold to KEEP RUNNING** — only the interlock
  aborts a running device.
- `FB_Init`'s first two inputs are ALWAYS `bInitRetains, bInCopyCode` (Beckhoff-standard,
  exempt from `inp`), followed by the `cfg*` seeds: `THIS^.cfgX := cfgX;` (guard-validate
  where sensible).
- No ACTIONs anywhere — behavior goes in METHODs with explicit access modifiers
  (`PRIVATE` helpers may sit in a `Privates` virtual folder).

## 5. FBs are SELF-CONTAINED — no external references

**A function block reaches out to nothing.** Everything it needs arrives through its own
interface; everything it knows sits inside it. A library FB is dropped into machine projects
we do not control, so anything it reaches for is something that must exist there too.

- **No GVL references from a library FB or DUT.** Not for state, and **not for sizing
  constants either** — `ARRAY[1..GVL_X.cMax] OF BOOL` is the same violation as reading a
  global flag. Hardcode the bound as a literal, like `fb_Interlock` does with
  `ARRAY[1..10]`, and use the same literal in the `FOR` loop that walks it.
- **Dependencies come IN, never get fetched.** Plain values as `VAR_INPUT`; when the FB
  genuinely needs a live external object, take a `REFERENCE TO`/`POINTER TO` **passed in
  through the interface** — `VAR_INPUT`, `VAR_IN_OUT`, or a setter. That is the one sanctioned
  form of coupling, because the caller — who owns the thing — decides. `abs_fb_Device`'s
  `faultMonitor : POINTER TO fb_FaultMonitor` is the pattern:
  declared in `VAR_INPUT`, wired by the machine.
- **A DUT is bound by this too, and more tightly.** Array bounds and string lengths are
  literals. A DUT is a type — it has no interface to receive anything through, so there is no
  legal way for it to depend on anything, and **an FB-typed pointer does not belong in one.**
  Storing one there makes a plain data type un-compilable without that FB and lets engine
  internals dereference an object nobody passed them. Hold such a pointer in the FB's own
  `VAR`, from a setter, where the ownership is visible.
- Constants an FB needs only *internally* go in its own `VAR CONSTANT` block (§7). That is
  self-contained; a GVL is not.

**Why no GVL, when a `VAR_GLOBAL CONSTANT` looks harmless:** it is a second place the truth
lives, in a file the consuming project must also have and could edit. Two machines on the same
library then disagree about a struct's layout while both compile, and the HMI mask contract
(which encodes 64 steps as 2×UDINT) silently stops matching. A literal cannot drift.

The cost is real and accepted: raising a capacity means editing the bound and its loop
together. That edit is visible in review; a GVL edit that resizes a struct three files away
is not.

> **One known exception: `GVLs/GVL_Cam.TcGVL`.** `fb_CamTeach` and `fb_CamTeachFile` size their
> point buffers with its `MAX_CAM_POINTS` (referenced unqualified, which is why it is easy to
> miss). It is listed in §10 until the bound moves into the FBs as a literal. Apart from it and the
> TwinCAT-generated `Version/Global_Version.TcGVL` there are no GVLs: the rule was written after
> the 2026-07 `fb_Permissive` / `abs_fb_MachineFlow` work broke it, and both were refactored back
> onto it the same pass, deleting `GVL_Permissive` (14 references) and `GVL_MachineFlow` (22). A
> new GVL reference is a regression.

## 6. Folders & files

- FBs: `POUs/FBs/` (general) · `POUs/FBs/Devices/` (all `fb_Dev_*`) · `POUs/FBs/Abstract/`
  · `POUs/FBs/Comm|Camming/` (domains) · `POUs/FC/` · `POUs/Interfaces/`.
- **DUTs mirror the POU tree.** An FB's DUTs live together in ONE subfolder named EXACTLY
  like the FB (matching case), under the same domain folder as the FB itself: interface +
  sub-structs + its enums. `fb_Dev_XPendant` → `DUTs/Devices/fb_Dev_XPendant/`,
  `fb_Modbus_Driver` → `DUTs/Comm/fb_Modbus_Driver/`, `abs_fb_Device` →
  `DUTs/Abstract/abs_fb_Device/`, `fb_Interlock` → `DUTs/fb_Interlock/`. A type used across a
  whole domain sits in that domain folder (`DUTs/Comm/ST_Modbus_Device_Config`); generic
  helpers (bit structs, unions) go in `DUTs/Common/`. Nothing lives in the `DUTs/` root.
- **Casing must match between the disk folder and the `.plcproj` paths** — this repo is
  edited on Linux too, where `DUTs\fb_Axis` ≠ `DUTs/FB_Axis`. Names are `fb_` everywhere,
  never `FB_` (that prefix is Beckhoff's: `FB_TcAlarm`, `FB_MBReadRegs`, `FB_Init`).

## 7. Declaration style

- Block order: `VAR_INPUT` → `VAR_OUTPUT` → `VAR` → `VAR CONSTANT`/`VAR PERSISTENT`.
- `//` for member/inline comments; a `(* … *)` header block at the top of every NEW FB
  stating purpose + usage (see `fb_HmiEventLog`).
- Enums: `{attribute 'qualified_only'}` + `{attribute 'strict'}` always; add
  `{attribute 'to_string'}` when the HMI displays the value. Members `e`-prefixed
  (`eNone := 0`), explicit base type (`) INT;`).
- Initial values inline (`_feedTimeout : TIME := T#1S500MS;`); bounded strings as
  `STRING(120)`; retained values in `VAR PERSISTENT`.

## 8. File format (TcPOU/TcDUT XML)

- Line 1 `<?xml version="1.0" encoding="utf-8"?>`, then `<TcPlcObject Version="1.1.0.1">`
  (no ProductVersion attribute in this repo).
- `<POU Name="…" Id="{guid}" SpecialFunc="None">` / `<DUT Name="…" Id="{guid}">`;
  declaration + code in `<![CDATA[…]]>`; each Method/Property/Get/Set/Folder gets its
  own fresh GUID.
- New files: UTF-8 **without BOM**, LF line endings, 4-space indentation (the current
  tooling style; pre-2026 files are BOM + tabs — leave them alone).

## 9. Registering a new file (or it won't compile)

In `PLCLibrary/PLCLibrary.plcproj`, per new file add to the Compile ItemGroup
(backslash paths, casing matching disk):

```xml
<Compile Include="DUTs\Devices\fb_Dev_XPendant\ST_XPendant_In.TcDUT">
  <SubType>Code</SubType>
</Compile>
```

and per NEW folder, one entry in the Folder ItemGroup: `<Folder Include="DUTs\Devices\fb_Dev_XPendant" />`.
`Version/Global_Version.TcGVL` is TwinCAT-generated from `<ProjectVersion>` — never
hand-edit it. TON/TRIG/TOF, `FB_TcAlarm`, `MC_*` need no new references (`Tc2_Standard`,
`Tc3_EventLogger`, `Tc2_MC2` are already in the placeholder list).

## 10. Known legacy deviations (do NOT imitate, do NOT mass-rename without a decision)

- `EN_*` enums (Banner/SPAL/VFD families) — new enums use `E_`.
- `E_Axis_*` enums missing the `qualified_only`/`strict` pragmas.
- `fb_Axis`: a device outside `Devices/`, pendant/jog inputs unprefixed in places.
- ~40 unprefixed `VAR_INPUT` members in older FBs (SPAL/ITECH/Modbus/MQTT);
  `faultMonitor` (should be `ref`-style) in `abs_fb_Device`.
- `UPPER_SNAKE` constants in `fb_Modbus_Driver`; `outInterlockOK` (`out` vs `sts`);
  `HMIbasedevice` member casing in `abs_fb_Device` vs `HMIbaseDevice` field;
  `EXTENDS abs_fb_device` (lowercase) in `fb_Dev_IsolationMonitor`;
  `GVL_Cam.MAX_CAM_POINTS` used by `fb_CamTeach` / `fb_CamTeachFile` (breaks §5).

## 11. New-device checklist

1. `POUs/FBs/Devices/fb_Dev_<X>.TcPOU` — `EXTENDS abs_fb_Device`, header docblock,
   `DeviceLogic();` first, publish-to-HMI block last, `FB_Init(bInitRetains, bInCopyCode,
   cfg…)`, config as `cfg*` properties.
2. `DUTs/Devices/fb_Dev_<X>/` — `ST_<X>_HMI_Interface` (+ `_Status`/`_Commands`/`_Config` as
   needed, `E_*` enums) — folder name == FB name.
3. Register every file + the folder in `PLCLibrary.plcproj` (§9).
4. Names: `inp*`/`sts*`/`cfg*`/`ocmd*`/`_*` per §2; no type prefixes.
5. Fresh GUID for every Id; file format per §8.

## 12. PLC project structure — ONE program, sequences under `MachineGlobal`

**A PLC project has exactly ONE `PROGRAM`: `MAIN`.** Everything else is a `FUNCTION_BLOCK`,
instantiated in a GVL and called from `MAIN`.

A second `PROGRAM` is justified **only** when it genuinely has to run separately:

- under a **different task**,
- at a **different cycle time**, or
- on a **different CPU / core**.

Convenience, grouping or "it felt like its own thing" are not reasons. A second PROGRAM in the
same task is a function call with extra ceremony and less of everything: it cannot be
instantiated twice, cannot be reused across machines, has no interface to pass values through,
and its execution order relative to `MAIN` is implicit rather than written down. An FB called
from `MAIN` has an explicit call order, a declared interface, and is reusable. When a PROGRAM
*is* warranted, its task, cycle time and reason belong in a comment at the top of it.

**Sequences are declared in `MachineGlobal` with a `seq` prefix**, so every sequence on a machine
is found in one place and reads as one at a glance:

```
// MachineGlobal.TcGVL
VAR_GLOBAL
    seqAuto     : fb_SEQ_MachineAuto;
    seqPurge    : fb_SEQ_Purge;
END_VAR
```

Called from `MAIN` like any other FB. The type keeps the `fb_SEQ_*` name (§1); `seq` is the
INSTANCE prefix, which is why it is not in the §2 table — that table covers FB *members*.

**This is about the MACHINE project, not the library.** Declaring instances in a machine GVL is
normal and is not what §5 forbids: §5 stops a *library* FB or DUT from reaching out to a GVL for
state or sizing. A machine's own `Devices` / `IO` / `MachineGlobal` GVLs are where its instances
live, and `MAIN` wires them together.

## 13. Machine project structure — GVLs, `MAIN`, axes, sequences

§1–12 are the rules; this section is how a **machine project** that uses LPLC is laid out so it
follows them. `<Machine>` is the machine's PascalCase name; everything in `<...>` is a placeholder.

### 13.1 Repo and PLC project

```
<Machine>-PLC/
  <Machine>.sln
  <Machine>.tsproj                   system project - edited by XAE, never by hand
  _Config/                           the tsproj's split-out config (IO/, NC/, PLC/) - XAE only
  PLC/
    PLC.plcproj                      the machine PLC project (ADS port 851)
    DUTs/  GVLs/  POUs/
  LPLC/                              the library, as a git submodule pinned to a release tag
  docs/
```

- One PLC project, named `PLC`, on ADS port **851**. Its task (`PlcTask`, e.g. 10 ms) calls
  **only `MAIN`** (§12).
- The machine uses the library through its **placeholder** (`LPLC`, namespace `LPLC`), never a
  project reference. Install the library from the pinned submodule before the first build; a `*`
  resolution picks the newest *installed* version.
- Pin the submodule to a release and say in the commit which library version the machine needs.

```
PLC/DUTs/    ST_Recipe_<Kind>          one recipe DUT per recipe type
             E_<Domain>_<Purpose>      machine enums
PLC/GVLs/    IO  Devices  MachineGlobal  MachineParameters  DAQ_Channels      (always these five)
PLC/POUs/    MAIN                      the ONE program
             fb_SEQ_<Name>             one per sequence
             fb_<Process>              process building blocks (control loops, measurements)
```

- Every file is registered in `PLC.plcproj` (§9); new files follow §8.
- Every GVL starts with `{attribute 'qualified_only'}`, so every reference reads
  `Devices.Axis_X`, never a bare `Axis_X`.
- Machine-specific FBs live in the machine project. Only genuinely reusable blocks go into the
  library, following §11.
- A dev or test harness gets its own GVL and never drives real axes.

### 13.2 The five GVLs

#### 13.2.1 `IO` — hardware, nothing else

Raw I/O only, named after the EtherCAT box and channel. **No logic, no scaling.**

```
{attribute 'qualified_only'}
VAR_GLOBAL
    RIO1000_AI_2            AT %I* : REAL;      // <what is wired here, units>
    RIO1001_DI_1            AT %I* : BOOL;      // <pedal / button / sensor>
    RIO1001_DO_1            AT %Q* : BOOL;      // <lamp / valve>

    RIO1005_<Drive>_SafetySet AT %Q* : BOOL;    // drive safety set (pulsed with Machine reset)
    RIO1005_<Drive>_Axis1          : AXIS_REF;  // <axis function> - NC Axis1_<Function>
END_VAR
```

Name boxes by remote-IO number in the system project (`RIO1000`, `RIO1001`, …) and follow them
here: `RIO<box>_<signal>_<n>` for I/O, `RIO<box>_<drive>_Axis<n>` for `AXIS_REF`s. Comment every
used channel with what is wired to it.

#### 13.2.2 `Devices` — the machine and its device FBs

```
{attribute 'qualified_only'}
VAR_GLOBAL
    Machine         : LPLC.fb_Machine(hasUPS := TRUE);      // modes, start/stop/reset, safety, fault monitor, event log

    Axis_<Name>     : LPLC.fb_Axis(cfgDeviceName := '<Name>', cfgEnable := TRUE, cfgInitEventClass := TRUE,
                              cfgIsModulo := FALSE, cfgJogVelocity := 50, cfgJogAcceleration := 150,
                              cfgStopDecel := 150, cfgHaltDecel := 200,
                              cfgAxisType := eRotary, cfgUnit := 'Deg');   // linear: cfgAxisType := 0, 'mm'

    <Sensor>_Sensor : REAL;                                  // the engineering value everything downstream reads
    Inst_<Sensor>   : LPLC.fb_Dev_AnalogSensor(cfgDeviceName := '<Sensor Name>', cfgEnable := TRUE,
                              cfgInitEventClass := TRUE, cfgAlarmEnable := TRUE, cfgEngUnit := '<unit>',
                              cfgAlarmH := .., cfgAlarmHH := .., cfgAlarmL := .., cfgAlarmLL := ..,
                              cfgCalOffset := 0, cfgCalGain := 1);

    teachPendant    : LPLC.fb_Dev_XPendant(cfgDeviceName := 'XPendant', cfgEnable := TRUE,
                              cfgInitEventClass := TRUE);   // only if the machine has a pendant
END_VAR
```

- **Instance names are a contract**: the HMI binds `Devices.<instance>.HMI.*` (§13.6). Pick them
  once.
- Axis instances are `Axis_<Name>`, where `<Name>` is the operator's name for the axis with `-`
  turned into `_` (`'Arm-Z'` → `Axis_Arm_Z`). Sensors are `Inst_<Sensor>`.
- Alarm limits are part of the process: when a sequence relies on one, say which in a comment.

#### 13.2.3 `MachineGlobal` — safety bits, HMI requests, sequences, process blocks

```
{attribute 'qualified_only'}
VAR_GLOBAL
    SafetyRun           AT %Q* : BOOL := TRUE;   // -> safety PLC Run
    SafetyErrAck        AT %Q* : BOOL;           // -> safety PLC error acknowledge  (= Machine.outReset)
    SafetyReset         AT %Q* : BOOL;           // -> safety PLC reset              (= Machine.outReset)
    SafetyESTOP_OK      AT %I* : BOOL;           // <- safety PLC: E-stop OK (STO released on every axis)

    EnableTeachPendant          : BOOL;          // HMI toggle; MAIN forces it FALSE in Auto
    ServoGlobalOverrideEnable   : BOOL;
    ServoGlobalOverrideSP       : INT;

    // HMI requests: one-shot BOOLs the HMI sets, MAIN consumes and clears.
    HMI_AutoFn_<Func>           : BOOL;          // buttons that start a sequence
    HMI_<Action>                : BOOL;          // page actions (capture, tare, reset ...)
    HMI_<Value>                 : LREAL;         // values that go with an action

    // Sequences - type fb_SEQ_*, instance seq*, all here (§12).
    seqHoming   : fb_SEQ_MachineHoming(cfgDeviceName := 'Homing Sequence');
    seqAuto     : fb_SEQ_<MainCycle>(cfgDeviceName := '<Main Cycle> Sequence');

    // Process building blocks the sequences switch on.
    <Block>     : fb_<Block>;
END_VAR
```

The safety bits are the standard program's view of the safety PLC. The safety logic itself lives
in the safety project (e.g. TwinSAFE); nothing here is a safety function.

#### 13.2.4 `MachineParameters` — `VAR_GLOBAL PERSISTENT`

Everything that must survive a restart: tuning (gains, limits, ramp times), calibration, the
**active recipe** and its name.

```
{attribute 'qualified_only'}
VAR_GLOBAL PERSISTENT
    <Loop>Kp            : REAL := ..;   // [units] what it does, which way to move it
    <Sensor>CalReading1 : LREAL;        // calibration
    Recipe              : ST_Recipe_<Kind>;   // written by the HMI on activate
    ActiveRecipeName    : STRING(80);         // written by the HMI after a clean activation; '' = none
END_VAR
```

- Every member gets a units-and-meaning comment; the HMI settings page can show it to the operator.
- A setpoint that comes from the recipe is an **effective copy**: `MAIN` overwrites it from
  `Recipe` every scan, and its comment says so.
- `ST_Recipe_<Kind>` is a flat STRUCT with units comments and sensible defaults. Its member names
  are the keys of saved recipe files, so renaming one breaks every saved recipe.
- Delete parameters the code no longer reads instead of leaving dead tuning knobs.

#### 13.2.5 `DAQ_Channels` — chart / historian signals

Flat `REAL`s with units comments, filled **only** in `MAIN.runDAQ`. An HMI chart lists this
namespace by name, so a signal that should be charted goes here, and its name is what the operator
sees.

### 13.3 Axis wiring

Per servo axis:

1. An NC axis `Axis<n>_<Function>`, linked to its drive channel and to
   `IO.RIO<box>_<drive>_Axis<n>`. Make it modulo only when it really turns endlessly, and then pass
   `cfgIsModulo := TRUE` as well.
2. `Devices.Axis_<Name> : LPLC.fb_Axis(...)`.
3. In `MAIN.runDevices`, every scan:
   - **Interlock slot 1** `'No faults present'` = `NOT Devices.Machine.faultMonitor^.stsFaultActive`,
     set **before** the axis call, so one fault anywhere stops every axis and a Reset clears it.
   - The drive's `SafetySet := Devices.Machine.outReset`.
   - The axis call:
     ```
     Devices.Axis_<Name>(inpAuto := Devices.Machine.stsAuto, inpReset := Devices.Machine.outReset,
         inpInterlockBypass := Devices.Machine.stsInterlocksBypassed, faultMonitor := Devices.Machine.faultMonitor,
         inpSTO := MachineGlobal.SafetyESTOP_OK, Axis := IO.RIO<box>_<drive>_Axis<n>,
         inpPendantMode := MachineGlobal.EnableTeachPendant, inpPendantEnable := _pendantEnable,
         inpPendantJogPos := <pad control>, inpPendantJogNeg := <pad control>,
         inpPendantJogFraction := <deflection 0..1>,
         inpGlobalOverrideEnable := MachineGlobal.ServoGlobalOverrideEnable,
         inpGlobalOverride := MachineGlobal.ServoGlobalOverrideSP);
     ```
4. **Re-enable after an E-stop.** On the E-stop NOK → OK edge, arm a pending enable (start the
   "previous" flag TRUE so a PLC start is not an edge). Fire `cmdEnable()` on every axis on the
   first scan with no active fault; drop the request if the E-stop opens again first. Enabling
   while anything is faulted would trip every axis's 'No faults present' interlock.
5. **External control.** A process block that drives an axis directly (a velocity loop, say):
   - takes it with `cmdSetExternalControl`;
   - waits until it is NC-ready (standstill and no error, with a timeout);
   - drives it with its own `MC_MoveVelocity` / `MC_Halt` on the **same** `AXIS_REF`;
   - on release, **halts first, then hands it back**.

   Tc2_MC2 `MC_MoveVelocity` has no ContinuousUpdate, so re-trigger Execute (one scan low) on a
   speed or direction change, rate-limited. One owner at a time.

Pendant: `_pendantEnable := EnableTeachPendant AND teachPendant.stsActive AND <dead-man input>`.

### 13.4 `MAIN` — the one program

```
PROGRAM MAIN
VAR
    _<seq>Request   : BOOL;     // one per HMI-startable sequence
    _safetyOkPrev   : BOOL := TRUE;
    _enablePending  : BOOL;
END_VAR

Devices.Machine(inpReset := <reset button>, inpStop := .., inpStart := .., inpAuto := ..,
                inpStartedFeedback := MachineGlobal.seqAuto.stsRunning,
                inpSafetyPowerOn := MachineGlobal.SafetyESTOP_OK, inpSafetyFaultActive := MachineGlobal.SafetyESTOP_OK,
                inpSafetyFaultDelay := MachineGlobal.SafetyESTOP_OK, inpSafetyResetButton := <reset button>);
IF Devices.Machine.stsAuto THEN MachineGlobal.EnableTeachPendant := FALSE; END_IF
MachineGlobal.SafetyErrAck := Devices.Machine.outReset;
MachineGlobal.SafetyReset  := Devices.Machine.outReset;

runDevices();       // I/O -> devices -> axes
runSequences();     // recipe copy, measurements, HMI requests, sequences, process blocks
runDAQ();           // DAQ_Channels
```

**`runDevices`**, in order:

1. Simple I/O mappings (potentiometers via `F_AnalogToEngUnit`, pedals).
2. Pendant.
3. Axis interlocks, then the axis calls (§13.3).
4. Re-enable after an E-stop.
5. Lamps and outputs.
6. Analog sensors (`inpScaledIOChannel` when the terminal already scales, else `inpRawIOChannel`
   with `cfgScalingNeeded := TRUE` and the channel / sensor / engineering ranges), then their
   `<Sensor>_Sensor` copies.
7. Timed outputs (e.g. a 500 ms tare pulse).

**`runSequences`**, in order:

1. Recipe → effective setpoints (`MachineParameters.X := MachineParameters.Recipe.X`).
2. Measurement blocks the sequences read (so they see this scan's values), and their HMI commands.
3. **HMI request handling**, the same shape for every startable sequence:
   ```
   IF MachineGlobal.HMI_AutoFn_<Func> THEN
       MachineGlobal.HMI_AutoFn_<Func> := FALSE;
       IF NOT MachineGlobal.seq<Func>.stsRunning AND NOT MachineGlobal.seq<Func>.stsAborting THEN
           MachineGlobal.seq<Func>.setStateLogicReset();     // back to Idle from Done / Faulted / Aborted
           _<func>Request := TRUE;
       END_IF
   END_IF
   ...
   IF Devices.Machine.outStart AND Devices.Machine.stsAuto THEN <same for seqAuto> END_IF
   IF NOT Devices.Machine.stsAuto THEN <clear every _xxxRequest> END_IF
   IF _<func>Request AND MachineGlobal.seq<Func>.stsIdle THEN
       _<func>Request := FALSE;
       MachineGlobal.seq<Func>.setStateLogicRun();
   END_IF
   ```
   If one sequence must stop another first, abort it here and gate the start on the other having
   stopped.
4. Every sequence call:
   ```
   MachineGlobal.seqX(inpReset := Devices.Machine.outReset, inpAuto := Devices.Machine.stsAuto,
                      faultMonitor := Devices.Machine.faultMonitor, refMachine := ADR(Devices.Machine));
   ```
5. Process blocks **after** the sequences, so they act on what the sequences asked for this scan:
   `Axis := IO.<AXIS_REF>, refAxis := ADR(Devices.Axis_X)`, with parameters from
   `MachineParameters`.

**`runDAQ`**: `DAQ_Channels.X := LREAL_TO_REAL(...)` lines, nothing else.

Use methods, not ACTIONs. No second PROGRAM unless it needs a different task or cycle (§12).

### 13.5 Sequences

Every sequence is `fb_SEQ_<Name> EXTENDS LPLC.abs_fb_stateDevice`:

```
(* fb_SEQ_<Name> - <what it does, how it is started, what Done leaves behind>
       0    <step>
       10   <step>
       999  done
   Interlocks: <list>. Abort: <what it stops>. Faults: <list>. *)
FUNCTION_BLOCK fb_SEQ_<Name> EXTENDS LPLC.abs_fb_stateDevice
VAR_INPUT
    refMachine  : POINTER TO LPLC.fb_Machine;
END_VAR
VAR
    _state<Name> : INT;
    _haltSent    : BOOL;
END_VAR
VAR CONSTANT
    c<LIMIT>     : <type> := ..;     // [units]
END_VAR

// --- body ---
stateLogic(ADR(_state<Name>));
cmdSetInterlock(1, 'Machine in Auto', _inAuto);
cmdSetInterlock(2, 'E-stop OK', MachineGlobal.SafetyESTOP_OK);
cmdSetInterlock(3, 'Teach pendant off', NOT MachineGlobal.EnableTeachPendant);
cmdSetInterlock(4, '<Other seq> not running', NOT MachineGlobal.seq<Other>.stsRunning);
IF stsRunning AND refMachine^.outStop THEN setStateLogicAbort(); END_IF

METHOD PROTECTED actionsRun     : BOOL   -> run<Name>();
METHOD PROTECTED actionsAbort   : BOOL   -> halt what it moves; setStateDoneAborting();
METHOD PROTECTED actionsFaulted : BOOL   -> IF NOT _haltSent THEN _haltSent := TRUE; <halt>; END_IF
METHOD PROTECTED actionsReset   : BOOL   -> clear latches (_haltSent := FALSE, owned blocks off)
METHOD PROTECTED actionsDone    : BOOL
METHOD PRIVATE   run<Name>      : BOOL   -> the CASE
```

Inside the `CASE`:

- Every step sets `stsStepDescription` (the operator reads it on the HMI), and a step that waits on
  hardware sets `setStateLogicStepTimeout(T#..)`.
- Step 0 resets the axes it needs. Step 5 waits for `faultMonitor^.stsFaultActive` to read FALSE
  (fault after ~3 s with "press Reset"), **then** `cmdEnable`.
- Faults: `setStateLogicFault('<Axis>: ' + text, FALSE, 0)`, as short operator sentences. A
  repeated advisory uses `sendWarning` (deduplicated).
- `cmdJogGoto` and move commands may refuse while an axis settles, so retry until they return TRUE.
- 999: `setStateLogicDone()`.

Rules:

- **Interlocks are simple global conditions** (mode, E-stop, pendant, other sequences, a sensor OK,
  "recipe active"). Anything that needs an explanation (recipe values against machine limits) is a
  **step that faults with a specific message**. Start-only conditions use `cmdSetStartPermissive`;
  only interlocks abort a running sequence.
- Sequences own policy (timeouts, "too far off", what faults). Process blocks own none.
- A process block has one owner at a time. The sequence that switched it on is the one that
  switches it off, in abort, fault and done.
- Build up from small sequences that are blocks of the main cycle, each startable on its own from
  an HMI request while commissioning.

### 13.6 HMI contract

An HMI (e.g. [lhmi](https://github.com/afarzin1/LHMI)) binds symbols **by name**, so renaming an
instance or a member breaks the binding. Bound by convention:

- `Devices.Machine.HMI.*`, `Devices.Axis_*.HMI.*`, `Devices.Inst_*.HMI.*`,
  `Devices.teachPendant.HMI.*`: the faceplates.
- `MachineGlobal.seq*`: sequence faceplates.
- `MachineGlobal.HMI_*`, `EnableTeachPendant`, `ServoGlobalOverride*`: buttons and requests.
- `MachineParameters.*`: settings, `Recipe`, `ActiveRecipeName`.
- `DAQ_Channels.*`: charts and historian.

### 13.7 New-project checklist

1. Repo, `.gitignore`, LPLC submodule pinned to a release, `docs/`.
2. XAE solution `<Machine>`: target, PlcTask, and NC tasks for servo machines.
3. EtherCAT scan; boxes renamed `RIO1000…`.
4. Drive commissioning; NC axes `Axis<n>_<Function>` with scaling, limits and soft limits.
5. PLC project `PLC` with the `LPLC` placeholder; library installed from the submodule.
6. GVLs `IO`, `Devices`, `MachineGlobal`, `MachineParameters`, `DAQ_Channels` (`qualified_only`).
7. `MAIN` with `runDevices` / `runSequences` / `runDAQ`; axes wired per §13.3.
8. Link `IO` ↔ terminals, `AXIS_REF` ↔ NC, safety bits ↔ the safety PLC.
9. Sequences, smallest first, each startable from an HMI request; then the main cycle on Machine
   Start in Auto.
10. Recipe DUT, `MachineParameters.Recipe` / `ActiveRecipeName`, and the HMI bindings against the
    final symbol names.
