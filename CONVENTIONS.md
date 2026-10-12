# LPLC — Coding & Naming Conventions

**Read this before adding or renaming ANYTHING.** These rules were extracted from the
codebase 2026-07-18 (evidence counts in brackets) and resolve the splits that keep
causing rename churn. Where old code deviates, the rule below is the CURRENT intent —
follow the rule, don't imitate the deviation (§10 lists the known legacy offenders).
**§5** (FBs reach out to nothing) and **§12** (one PROGRAM; sequences under `MachineGlobal`) are
the two structural rules — read those even if you skip the naming tables.

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
