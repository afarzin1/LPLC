# abs_fb_MachineFlow — design

An SFC-style **token engine** for parallel machine sequences, extending `abs_fb_stateDevice`. Steps
and transitions are declared as data in `defineFlow()`; step bodies live in `stepActions()`. The
graph is authored **once, in the PLC**, and exported as a static JSON string that lhmi's
`shared/v2/flow` renders — so the HMI never re-declares a sequence and the two cannot drift.

## The one primitive

A transition fires when **all** its source steps are active-and-done **and** its gate is satisfied.
Firing consumes the source tokens and activates every target. That single rule gives:

| shape | how |
|---|---|
| AND-split (divergence) | one transition, several targets |
| AND-join (convergence) | one transition, several sources |
| OR-choice | two transitions from one step with opposing gates |
| loop | a target whose rank is at or above the source |

## Transition gates

**One layer: `permissive : BOOL`** — sticky, set from a step body via `setPermissive(t, cond)`, reset
to TRUE on reset. Whatever the gate really is, the machine ANDs it together and pushes the result in:

```pascal
VAR CONSTANT                    // ids must be CONSTANT — they are CASE labels below
    cStepEvac : INT := 1;       // in addStep() call order
    cStepHeat : INT := 2;
END_VAR
VAR
    _transReady : INT;          // transition ids are never CASE labels, so plain VAR
END_VAR

defineFlow():
    addStep('Evacuate', T#10M, 'pull below 5 Torr');   // -> cStepEvac
    addStep('Heat',     T#0S,  'ramp to setpoint');    // -> cStepHeat
    setInitialStep(cStepEvac);
    _transReady := flow(cStepEvac, cStepHeat, 'chamber ready');

stepActions():   // beside the stepDone() the body already calls
    CASE inpStepId OF
    cStepEvac:
        setPermissive(_transReady, PermChamberReady.stsPermissiveOK AND recipeLoaded);
        IF Pressure < 5.0 THEN stepDone(cStepEvac); END_IF
    END_CASE
```

Note `addStep(name, timeout, description)` — it takes the NAME and **returns** the id; it does not take
one. Ids are assigned in call order, which is why the constants above are hand-numbered to match. See
the FB header for the `IF addStep(…) <> cStepEvac THEN setCfgError(…)` guard against drift.

**Why not a pointer to the fb_Permissive (removed 2026-07-26).** An earlier revision put a
`refPermissive : POINTER TO fb_Permissive` in `ST_MachineFlow_Transition` and dereferenced it every
scan, ANDing `stsPermissiveOK` in inside the engine. It bought nothing:

- The **boolean already expresses the effective gate.** The machine can AND the permissive in at the
  `setPermissive()` call in one line — the same shape as `stepDone(id)`, and the same direction as
  every other thing a step body tells the engine.
- The **HMI never needed the relay.** lhmi maps transition → permissive itself (`gates={{1: "…"}}`)
  and binds that instance's `ST_Permissive_HMI_Interface` directly, so the condition list comes from
  the permissive, not through this FB. The flow chart only needs *whether* the gate is met — one bit,
  which the permissive mask already carries.
- It was **incongruent with steps.** Steps are values the engine owns, told about through methods; the
  permissive was a borrowed object pulled out by dereference. Opposite ownership, opposite direction.
- A stored pointer **can go stale** — reassigned, re-instantiated, or out of scope — and the `<> 0`
  guards catch NULL, not freed. A step struct has no such failure mode. On a CX7000 that is a crash,
  and the kind that surfaces months in.
- It made a plain data type depend on an FB: `ST_MachineFlow_Transition` could not compile without
  `fb_Permissive`. See CONVENTIONS §5.

The memory argument that motivated the pointer — 64 embedded `fb_Permissive` instances (10 BOOL +
10 × STRING(80)) is roughly 110 kB, real money on a CX7000 — is sound, and it still rules out
embedding. It just never implied a *pointer*: a BOOL fed from an instance the machine owns costs
nothing and keeps `fb_Permissive` the independent component it was always meant to be.

`actionsReset` resets `.permissive` (and `.fired`) only, never the whole transition record, so the
declared topology survives a reset.

## HMI contract

`HMIflow : ST_MachineFlow_HMI_Status`, consumed by `shared/v2/flow`. Full key table in that
subsystem's README. Two properties of it that matter:

**Masks are UDINT halves, never one LWORD.** A JSON/JS transport is exact only to 2^53, so a 64-bit
mask must not travel as a single number — hence `stsActiveLo`/`Hi`, `stsDoneLo`/`Hi`,
`stsPermissiveLo`/`Hi`.

**`stsPermissiveLo`/`Hi` mirror the EFFECTIVE gate** — the sticky boolean AND any attached
`fb_Permissive`. Reporting a gate as met while its permissive blocks would draw a ready gate on a
sequence that cannot advance, which is precisely the symptom the operator opened the chart to
diagnose. (This was fixed when permissive references were added; the mask previously read the
boolean alone.)

**`stsFlowDef` is static after startup** — read it once, on demand, never cyclically. It is capped at
**4095** bytes. Overflow is not silent: `appendDef` sets `_defOverflow`, and `actionsRun` raises
`sendWarning('… definition JSON truncated - shorten step/transition names or widen stsFlowDef')`.
Names and step descriptions are what consume it.

This contract is **ADS-scoped**. The def string and the mask set are not designed for the MQTT
transaction-group path used by RTOS-class controllers.

## Limits

64 steps · 64 transitions · 4 sources/targets per transition · 4095-byte `stsFlowDef`.

**Literals, not a GVL** (CONVENTIONS §5 — `GVL_MachineFlow` was deleted 2026-07-26): each appears in
its array bound and in every loop that walks it, so raising one is a paired edit. The FB header lists
where. **64 is the one number not free to change** — the HMI publishes its masks as 2 × UDINT
(lo = 1..32, hi = 33..64), so widening means new mask members and a new HMI contract, not a bigger
bound.

## Status — open threads

- [x] **COMPILES, as of 2026-07-26.** Built in TwinCAT against a consuming machine project via
  `fb_SEQ_LibTest`, the first concrete subclass this FB has ever had — an ABSTRACT FB alone does
  not make the compiler resolve its overrides and call graph. **One fix was required:** the local
  `s` in `buildDefJson`/`finalizeFlow`/`fireTransitions` was rejected as an identifier and is now
  `sid`. Every other error in that build cascaded from it.
- **NOT YET RUN.** Compiling proved the syntax and type resolution; no token has ever moved. The
  engine's actual behaviour — firing order, parallel joins, loop-backs, timeout faults, retry
  resume, the def JSON's content — is all still unverified. That is now the gate.
- **No concrete implementation SHIPS.** The subclass that proved the compile (`fb_SEQ_LibTest`) lived
  on a throwaway machine-project branch and was deleted with it, so nothing extends
  `abs_fb_MachineFlow` in this library or in any machine project again. The compile result stands,
  but **reproducing it means re-authoring a subclass** — and a machine project is the wrong home for
  one anyway now that CONVENTIONS §12 allows only a single PROGRAM. A demo `fb_SEQ_*` belongs HERE,
  in the library, as both the reference implementation and the permanent compile target.
- **`HMIflow.stsActiveNames` was dead until 2026-07-26.** The name is declared twice — in
  `ST_MachineFlow_HMI_Status` and as an FB-level member — and `publishHMI` wrote the unqualified FB
  member, leaving the struct field an empty string forever. It compiled cleanly, which is what made
  it a trap: the struct documents the field as *"' + '-joined names of the active steps"*, so a
  future binding of `flow.activeNames` would have silently read `''`. Now populated. lhmi was
  unaffected — it reads the same text via the inherited `stsStepDescription`.
