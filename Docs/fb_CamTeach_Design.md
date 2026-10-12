# fb_CamTeach — Design Document

## Purpose

A **reusable, machine-agnostic** function block that teaches and manages **one** NC cam table:
the `master → slave` relationship for a single slave. It records taught points, persists the raw
points to file, post-processes them, converts them to an NC motion function, downloads that
function to a pre-created NC table, and verifies it. Its job **ends at a verified table** plus a
`stsVerified` permission flag — it does **not** couple axes and knows **nothing** about the machine
it serves (no fiber, no mold-w, no accumulator).

Instantiate **one per slave**. A machine that cams five slaves to one master instantiates five
`fb_CamTeach`. Coupling is done by [`fb_CamCoupler`](fb_CamCoupler_Design.md);
machine-specific teach policy and the run sequence live in the application layer. This block is a library primitive both of those compose.

---

## Layering — where this sits

```
  Library layer (reusable, this doc + fb_CamCoupler)
    fb_CamTeach   : teach → persist → convert → download → verify   (per slave)
    fb_CamCoupler : MC_CamTableSelect / MC_CamIn / MC_CamOut         (per slave)
  Machine layer (application, e.g. fb_Winder)
    owns teach policy, master source, run sequence, tension, sensors
    composes N × fb_CamTeach + N × fb_CamCoupler + M × fb_Axis
```

`fb_CamTeach` produces verified tables; `fb_CamCoupler` consumes the `stsVerified` flag and does the
coupling; the machine layer orchestrates both.

---

## Scope

**In scope**
- Append / delete-last / spot-edit / clear taught points for one slave.
- Persist **raw** taught points to/from file (source of truth); NC tables are volatile.
- Post-process: zero the master offset, enforce strict-monotonic master, optional light smoothing.
- Convert raw points → `MC_MotionFunctionPoint[]` (cubic, C1-smooth from position-only teaching).
- Download to a pre-created NC table (`MC_WriteMotionFunction`) and verify (`MC_ReadMotionFunctionPoint`).
- Expose a per-table **verified** flag gating any downstream coupling.
- Online point edit applied at a defined master position.

**Out of scope**
- Coupling / `MC_CamIn` / `MC_CamOut` → `fb_CamCoupler`.
- What "master" means, when to capture, teach sequencing → machine layer.
- Computing profiles mathematically (this is teach-based by design).

---

## Design Principles

1. **One instance, one table.** Each `fb_CamTeach` owns a single `master → slave` table. Phase lock
   between slaves is guaranteed by all slaves being cammed to the **same master axis** at runtime —
   **not** by tables sharing identical master breakpoints. So per-slave tables with independent
   breakpoints are correct and are the reusable granularity. (This deliberately drops the earlier
   "one shared master column" constraint, which was an authoring convenience, not a phase-lock
   requirement, and is impossible to satisfy when different slaves are taught by different methods.)

2. **Teach by value, not by axis.** The record command takes **scalar** `master` and `slave` values,
   not an `AXIS_REF`. The caller decides where they come from — a real axis position, a virtual
   master, or a derived signal (e.g. a fiber-payout accumulator that is not an axis at all). This is
   what makes the block reusable across machines.

3. **File is the source of truth; store raw.** Persist the **raw taught points**, never the
   converted motion-function structs — the interpolation strategy may change and must be
   regenerable. NC tables are volatile (empty after every TwinCAT restart) and are rebuilt from file.

4. **Ends at verified.** The block's output is a downloaded, read-back-verified table plus a
   `stsVerified` flag. It never issues motion.

5. **Async, non-blocking.** File I/O, download, and verify are multi-scan state machines. File I/O is
   isolated in a sub-FB (`fb_CamTeachFile`).

---

## Interface (skeleton)

No `AXIS_REF` — the block is fed values and a table ID.

```iecst
FUNCTION_BLOCK fb_CamTeach
VAR_INPUT
    inpReset  : BOOL;
END_VAR
VAR_OUTPUT
    stsState     : E_CamTeach_State;
    stsNumPoints : INT;
    stsVerified  : BOOL;     // table downloaded AND read-back-verified — gates fb_CamCoupler
    stsBusy      : BOOL;     // file / download / verify in progress
    stsError     : BOOL;
    stsErrorText : STRING;
END_VAR
VAR
    HMI     : ST_CamTeach_HMI_Interface;   // Commands / Status / Config (fb_Axis convention)
    _points : ARRAY[1..MAX_CAM_POINTS] OF ST_CamTeach_Point;   // raw taught points (working set)
    _n      : INT;
    _state  : E_CamTeach_State;

    // NC motion-function table this instance owns (pre-created under Motion → Tables)
    _tableId   : UDINT;                    // set via cfgTableId / FB_Init
    _mfPoints  : ARRAY[1..MAX_CAM_POINTS] OF MC_MotionFunctionPoint;
    _mcWrite   : MC_WriteMotionFunction;
    _mcRead    : MC_ReadMotionFunctionPoint;
    _mcWritePt : MC_WriteMotionFunctionPoint;   // online single-point edit
    _mcOnline  : MC_SetCamOnlineChangeMode;

    _file : fb_CamTeachFile;               // isolated async persistence sub-FB
END_VAR
```

Config (via `FB_Init` + `cfg*` properties, per the fb_Axis pattern):
`cfgTableId : UDINT`, `cfgSlaveName : STRING`, `cfgPeriodic : BOOL`, `cfgInterp : E_CamTeach_Interp`,
`cfgSmoothWindow : INT` (0 = off), `cfgFileDir : STRING := 'C:\_MachineConfig\CamTables'`.

---

## Data Model (DUTs)

```iecst
TYPE ST_CamTeach_Point :          // one raw taught pair for THIS slave
STRUCT
    rMaster : LREAL;              // [master unit] strictly increasing after post-process
    rSlave  : LREAL;              // [slave unit]
END_STRUCT
END_TYPE

TYPE E_CamTeach_State  : (eIdle, eTeaching, eConverted, eSaving, eLoading,
                          eDownloading, eVerifying, eVerified, eError) INT; END_TYPE
TYPE E_CamTeach_Interp : (eLinear, eCubicC1) INT; END_TYPE   // eCubicC1 = POLYNOM3 + central-diff slope
```

Global constants (GVL):
```iecst
MAX_CAM_POINTS : INT := 4096;   // ≥ 360/rev × 7 revs + margin
```

HMI: `ST_CamTeach_HMI_Interface` with nested `Commands` / `Status` / `Config`, self-clearing one-shot
command flags — same convention as `fb_Axis`.

---

## Command Methods

| Method | Purpose |
|--------|---------|
| `cmdRecord(rMaster, rSlave)` | Append a raw point (caller supplies both values) |
| `cmdDeleteLast()` | Remove the last recorded point |
| `cmdEditPoint(idx, rMaster, rSlave)` | Spot-edit one point |
| `cmdClear()` | Empty the working set |
| `cmdSave(fileName)` | Persist raw points → file (async) |
| `cmdLoad(fileName)` | File → raw points (async), validate on completion |
| `cmdConvert()` | Post-process + build `_mfPoints[]` → `eConverted` |
| `cmdDownload()` | `MC_WriteMotionFunction` then read-back verify → `eVerified` (sets `stsVerified`) |
| `cmdEditOnline(idx, rSlave)` | Live point edit while cammed (see "Online Edit") |

---

## State Machine

```
  eIdle ─cmdRecord─► eTeaching ─cmdConvert─► eConverted ─cmdDownload─► eDownloading ─► eVerifying ─► eVerified
    ▲                    │ cmdSave/cmdLoad                                                              │
    └──── inpReset ──────┴──────────────► eSaving/eLoading ──► (back to prior)      stsVerified := TRUE ┘
  any step error → eError (stsErrorText); inpReset → eIdle. Editing points clears stsVerified
  (a re-convert + re-download is required before coupling is allowed again).
```

---

## Post-Processing (inside cmdConvert)

Applied to the raw points before building the motion function:
1. **Zero the master offset** — subtract `_points[1].rMaster` so the table starts at master = 0.
2. **Strict-monotonic master** — delete any point whose master is ≤ its predecessor (removes reversal
   points from teach jitter / tension transients). Fault if fewer than 2 points survive.
3. **Optional smoothing** — `cfgSmoothWindow` (3–5) point moving average applied to the **master**
   column to remove hand-teach noise; 0 disables.

---

## Conversion (raw points → motion function)

Build `_mfPoints[i]` per point (Tc2_MC2_Camming), yielding a C1-smooth cubic cam **from
position-only teaching** — no need to teach velocities:

| Field | Value |
|-------|-------|
| `PointIndex` | `i` |
| `Function` | `MOTIONFUNCTYPE_POLYNOM3` (verify enum name against installed lib version) |
| `RelIndexNextPoint` | `1` |
| `MasterPos` | `rMaster` |
| `SlavePos` | `rSlave` |
| `SlaveVelo` | central-difference slope `(rSlave[i+1]−rSlave[i−1]) / (rMaster[i+1]−rMaster[i−1])` |
| `SlaveAcc` | `0` |
| last point `Function` | `MOTIONFUNCTYPE_NONE` (terminator) |

Endpoints use one-sided differences. For a **periodic** table (`cfgPeriodic`), wrap the
central-difference at the seam so velocity is continuous across the period boundary.

---

## Download & Verify (inside cmdDownload)

Tables are **pre-created** under Motion → Tables as motion-function type, sized ≥ `MAX_CAM_POINTS`.
1. `MC_WriteMotionFunction` — bulk-write `_mfPoints[1.._n]` to `_tableId`.
2. Read back a sample (or all) with `MC_ReadMotionFunctionPoint`; compare to `_mfPoints`.
3. On match → `eVerified`, `stsVerified := TRUE`. On mismatch/error → `eError`, `stsVerified := FALSE`.

`stsVerified` is the interlock: `fb_CamCoupler` must not `MC_CamIn` until it is TRUE.

---

## Online Edit (while cammed)

For live cam trimming without decoupling:
1. Update the working `_points[]` (and **re-save the file** so it stays truthful).
2. `MC_WriteMotionFunctionPoint` for the changed point(s).
3. `MC_SetCamOnlineChangeMode` configured to apply the change **at a defined master position**, not
   immediately — so an edit to a segment the slave is currently traversing doesn't step the setpoint
   mid-winding.

(Distinct from `fb_CamCoupler`'s offset/superimpose trim, which shifts the whole coupling rather
than editing table points.)

---

## Persistence — ownership & location

**`fb_CamTeach` owns file I/O**, implemented in the isolated async sub-FB `fb_CamTeachFile`
(`Tc2_Utilities`: `FB_FileOpen → FB_FilePuts/FB_FileGets → FB_FileClose`, non-blocking).

- **Default directory: `C:\_MachineConfig\CamTables`** (`cfgFileDir`; caller passes just a file name).
- One file per slave table, e.g. `<pattern>_<slaveName>.csv`.
- **Format — CSV:** a small `#` metadata header (`slaveName`, `created`, `numPoints`, `periodic`,
  `interp`, `masterUnit`) then `master,slave` rows of **raw** taught values.
- The file is the source of truth; NC tables are recreated from it at startup. The machine layer's
  teach policy decides *when* to save (e.g. after every revolution).

```
# slaveName=mold_z  created=2026-07-14T09:12:00  numPoints=812  periodic=0  interp=cubicC1  masterUnit=mm
0.000, 10.000
1.021, 10.014
...
```

---

## Caveats / Open Items

- **Enum/API names are TF5050-version-dependent** — verify `MOTIONFUNCTYPE_POLYNOM3` /
  `MOTIONFUNCTYPE_NONE`, `MC_WriteMotionFunction`, `MC_ReadMotionFunctionPoint`,
  `MC_WriteMotionFunctionPoint`, `MC_SetCamOnlineChangeMode` against the installed library.
- **Pre-created tables** — the NC table IDs must exist and be sized before download; `cfgTableId`
  binds this instance to one.
- **Timestamps** — `created` is stamped by the caller/HMI (`Date`/wall-clock unavailable in scan).
- **Memory** — `MAX_CAM_POINTS` × 2 LREAL per instance; the file (not PERSISTENT memory) provides
  durability, avoiding retain-size limits at 2500+ points.

---

## Build Phasing

1. **Teach & persist:** points array, record/delete/edit/clear, CSV save/load via `fb_CamTeachFile`,
   post-processing. Verify captured curves offline (Scope / spreadsheet).
2. **Convert, download, verify:** motion-function build, `MC_WriteMotionFunction` + read-back,
   `stsVerified`.
3. **Online edit:** `MC_WriteMotionFunctionPoint` + `MC_SetCamOnlineChangeMode`.
