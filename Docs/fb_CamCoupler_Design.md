# fb_CamCoupler — Design Document

## Purpose

A **reusable, machine-agnostic** function block that safely couples **one** slave axis to a master
axis using a verified NC cam table. It wraps `MC_CamTableSelect` / `MC_CamIn` / `MC_CamOut`, gates
engagement on the [`fb_CamTeach`](fb_CamTeach_Design.md) `stsVerified` flag,
coordinates the [`fb_Axis`](fb_Axis_Design.md) yield hook so the slave stops issuing its
own motion while coupled, and decouples cleanly on fault.

Instantiate **one per slave**. The machine layer (e.g. a winder's `fb_Winder`) owns N of these plus the
run sequence. This block contains no machine-specific logic and does not move the master — the
master is driven independently (typically by its own `fb_Axis` velocity move).

---

## Layering — where this sits

```
  fb_CamTeach   → produces a verified table + stsVerified   (per slave)
  fb_CamCoupler → consumes stsVerified, couples slave↔master (per slave)   ← THIS BLOCK
  fb_Axis       → owns each axis; yields to the coupler while cammed
  Machine layer → owns run sequence, master motion, interlocks
```

---

## Scope

**In scope**
- `MC_CamTableSelect` (register the table) and `MC_CamIn` (engage) for one slave.
- Safe engagement: gated on `verified`, axis enabled, interlocks OK; RampIn / offset so the slave
  need not be exactly at the cam's start position.
- `MC_CamOut` (disengage) followed by a controlled stop of the freed slave.
- Drive the `fb_Axis` yield state around couple/decouple (mutual exclusion — one motion owner).
- Decouple-on-fault / on interlock loss.
- Live trim via cam offset / superimposed correction (without decoupling).

**Out of scope**
- Building/verifying the table → `fb_CamTeach`.
- Moving the master, teach policy, multi-axis sequencing → machine layer.

---

## Design Principles

1. **One instance, one slave.** Coupling is per slave-master pair. Phase lock across slaves comes
   from them sharing the same master axis, not from the coupler.
2. **Never engage an unverified table.** `MC_CamIn` is hard-gated on the incoming `verified` input
   (sourced from that slave's `fb_CamTeach.stsVerified`).
3. **One motion owner at a time.** While coupled, the slave's `fb_Axis` must be yielded (issues no
   jog/move/home). The coupler drives that state via `fb_Axis.cmdSetExternalControl(enable)`;
   without it a coupled axis and its `fb_Axis` would fight (NC error). **Implemented** — while
   yielded, `fb_Axis` keeps power/reset/override/status but suppresses all its own motion and
   rejects motion commands quietly (no fault, so `MC_Power` stays up and the coupling holds).
4. **Clean decouple.** After `MC_CamOut` a slave retains its last velocity — the coupler immediately
   commands a stop (or hands back to `fb_Axis`, which does).
5. **Async, non-blocking.** Couple/decouple are multi-scan state machines.

---

## Interface (skeleton)

```iecst
FUNCTION_BLOCK fb_CamCoupler
VAR_IN_OUT
    Master : AXIS_REF;                 // shared master (driven elsewhere)
    Slave  : AXIS_REF;                 // this slave
END_VAR
VAR_INPUT
    verified    : BOOL;                // from fb_CamTeach.stsVerified for this slave — REQUIRED to couple
    tableId     : UDINT;               // NC cam table id (same one fb_CamTeach downloaded)
    periodic    : BOOL;                // MC_CamTableSelect periodicity for this slave
    masterOffset: LREAL;               // engage offsets to match current pose at master start
    slaveOffset : LREAL;
    masterScale : LREAL := 1.0;
    slaveScale  : LREAL := 1.0;
    interlockOK : BOOL;                // machine/axis permissive
    slaveFB     : POINTER TO fb_Axis;  // yield coordination (optional but recommended)
    cmdCouple   : BOOL;
    cmdDecouple : BOOL;
    inpReset    : BOOL;
END_VAR
VAR_OUTPUT
    stsCoupled : BOOL;                 // MC_CamIn.InSync
    stsBusy    : BOOL;
    stsState   : E_CamCoupler_State;
    stsError   : BOOL;
    stsErrorText : STRING;
END_VAR
VAR
    _camSelect : MC_CamTableSelect;
    _camIn     : MC_CamIn;
    _camOut    : MC_CamOut;
    _mcHalt    : MC_Halt;              // stop the freed slave after CamOut
    _state     : E_CamCoupler_State;
END_VAR

TYPE E_CamCoupler_State : (eIdle, eSelecting, eYielding, eCoupling, eCoupled,
                           eDecoupling, eStopping, eError) INT; END_TYPE
```

---

## Couple Sequence (cmdCouple)

```
1. Preconditions: verified = TRUE, Slave enabled, interlockOK, not already coupled.
                  (Reject with stsError if any fail.)
2. Yield:   slaveFB^.cmdSetExternalControl(enable := TRUE)  — fb_Axis halts, then stops issuing its own motion.
3. Select:  MC_CamTableSelect(Master, Slave, tableId, Periodic := periodic, MasterAbsolute, SlaveAbsolute).
4. Engage:  MC_CamIn(Master, Slave, CamTableID, StartMode := <RampIn>,
                     MasterOffset, SlaveOffset, MasterScaling := masterScale, SlaveScaling := slaveScale).
5. stsCoupled := _camIn.InSync.  → eCoupled.
```

Set offsets so the cam's slave value at the master start equals the slave's **current** pose, and/or
use a **RampIn** start mode, so engagement doesn't jump the axis. The machine layer moves each slave
to the pattern-zero pose before couple and starts the master only once **all** couplers report
`stsCoupled`.

## Decouple Sequence (cmdDecouple / fault)

```
1. MC_CamOut(Slave).
2. Slave keeps last velocity → MC_Halt(Slave) (or hand to fb_Axis, which stops it).
3. slaveFB^.cmdSetExternalControl(enable := FALSE) — fb_Axis resumes control (stays at rest until a
   new explicit command).  → eIdle (or eError if fault-triggered).
```

---

## Safety

- **Fault / interlock loss / STO while coupled** → immediate decouple sequence (CamOut → stop),
  release yield, latch fault; no auto re-couple (deliberate re-couple required).
- **verified drops FALSE** (e.g. table edited) while coupled → treat as decouple condition; the
  online-edit path in `fb_CamTeach` is the supported way to change a live table.
- The coupler adds no independent safety authority — the machine STO/interlocks and each `fb_Axis`'s
  gating remain in force through the shared `AXIS_REF`.

---

## Live Trim (without decoupling)

Two mechanisms, both keeping the coupling engaged:
- **Cam offset:** adjust `masterOffset` / `slaveOffset` (or re-issue `MC_CamIn` with new offsets in
  the modes that support it) to shift the whole relationship.
- **Superimposed correction:** `MC_MoveSuperImposed` on the slave to add a bounded corrective move on
  top of the cam.

For editing individual table **points** live, use `fb_CamTeach.cmdEditOnline` (which uses
`MC_SetCamOnlineChangeMode` to apply at a defined master position).

---

## fb_Axis yield hook (implemented)

The dependency is satisfied. `fb_Axis.cmdSetExternalControl(enable)` sets `_externalControl`; on
enable it runs an interruptible `cmdHalt()` to bring the axis to rest, then the scan body suppresses
**all** of the FB's own motion while yielded:
- jog / pendant jog, torque-jog, the move state machine, and home are gated off;
- `cmdStart` / `cmdHome` / `cmdJog*` reject **quietly** — deliberately **no** `setFault`, because a
  fault gates `MC_Power` and would drop the axis and break the coupling;
- `MC_Power` / `MC_Reset` / `MC_SetOverride` / status keep running, so the axis stays powered and
  reportable while the coordinator drives the shared `AXIS_REF`.

Coupler note: after enabling external control, wait for the slave to reach standstill (the entry
`cmdHalt` completes) before `MC_CamIn`, so the halt and the coupling don't overlap.

---

## Caveats

- Verify `MC_CamTableSelect` / `MC_CamIn` / `MC_CamOut` start-mode enums and offset/scaling semantics
  against the installed Tc2_MC2 / TF5050 version.
- Periodic slaves (e.g. a modulo axis whose 1-rev table accumulates over multiple master periods)
  need the correct periodic + slave-continuation settings — the machine layer supplies `periodic`
  per slave.
