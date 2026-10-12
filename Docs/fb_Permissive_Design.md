# fb_Permissive — design

A set of up to 10 conditions that must **all** be true for something to be **allowed to proceed**.
All-true means satisfied; unwired slots default TRUE so you wire only what applies.

## Why it is not fb_Interlock

The two look nearly identical and mean different things:

| | `fb_Interlock` | `fb_Permissive` |
|---|---|---|
| question | may this device **keep running**? | may this **start / advance**? |
| unmet is | **abnormal** — a fault to raise and clear | **normal** — the machine is waiting |
| raises a fault | yes (via the owning device) | **no** — deliberately none |
| extends | used by `abs_fb_Device` | nothing; standalone |

"Chamber not yet at vacuum" is not a fault, it is a Tuesday. Modelling it as an interlock puts a
fault on the HMI that the operator cannot clear and should not have to, and trains people to ignore
the fault channel. That is the whole reason this FB exists separately, and why it has no TcAlarm
plumbing, no `setFault`, and does not extend `abs_fb_Device`.

## Interface

```
inpConfig : ST_Permissive     // conditions[1..10] (default TRUE) + descriptions[1..10] (default 'N/A')

stsPermissiveOK : BOOL        // TRUE when every condition is met
stsUnmetCount   : INT         // how many are unmet

HMI : ST_Permissive_HMI_Interface   // bind <instance>.HMI.* — shared lhmi PermissiveList renders it
cfgName : STRING                    // property; the heading the operator sees
setCondition(slot, met, description) // bounds-checked convenience setter
```

That is the whole interface: one input, two outputs, one HMI struct. **Self-contained** per
CONVENTIONS §5 — no GVL, no global state, nothing fetched from the consuming project.

## Decisions worth keeping

**Conditions default TRUE.** A permissive with three of ten slots wired must be satisfied by those
three. Defaulting FALSE would block every partially-wired permissive forever, and the failure would
look like a plant problem rather than a config one. Same default as `ST_Interlock`.

**No first-out — and this is the sharpest line between the two FBs.** An interlock **trips**: one
condition drops, the device faults instantly, and "which one went first" names the *cause* (the later
ones are usually consequences of it). A permissive never trips. It is evaluated at the moment
something asks to proceed, and at that instant every unmet condition is equally and simultaneously
the reason. There is no first, so nominating one would be inventing a causal story the data does not
support. The FB therefore publishes the **set** — `stsConditions` plus `stsUnmetCount` — and the HMI
shows all of them.

(An earlier draft of this FB carried a `stsFirstUnmet`, copied over from `fb_Interlock` by analogy.
It was removed: the analogy is exactly what does not hold.)

**Related fix in `fb_Interlock`, 2026-07-26.** Its `stsFirstOut` assigned on *every* failing
iteration with no empty-guard, so despite the name it reported the **last** failing condition — which
for a trip means the operator was pointed at a symptom rather than the root cause. Now guarded, so it
is genuinely first. Behaviour change for any HMI already reading it: it will now show a different
(correct) condition.

**No bypass.** An earlier draft carried `inpBypass` + `stsBypassed`, argued for on the grounds that a
permissive is not a safety function so forcing one risks a wrong *sequence* rather than an unsafe
machine. Rejected: a bypass is exactly the control that makes an unsatisfied gate report itself
satisfied, and a sequence advanced on a forced permissive fails downstream, somewhere that explains
nothing about why. The conditions hold or they do not — there is no third answer worth publishing.
A condition that should not apply simply goes unwired; slots default TRUE.

`fb_Interlock.inpGlobalBypass` is a genuinely different case and stays: an interlock **stops a
running machine**, so commissioning needs a documented way past it.

**No GVL — the FB is self-contained (CONVENTIONS §5).** An earlier draft put `cMaxConditions` and
`cDescLen` in a `GVL_Permissive`, so one edit could resize the arrays and their loops together. That
GVL is deleted: a library FB dropped into a machine project we do not control must not reach for
anything that project also has to have, and sizing constants are no exception — a GVL is a second
place the truth lives, and two machines on the same library could disagree about a struct's layout
while both compile. Ten is now a literal in `ST_Permissive`, in the HMI struct and in the loops, the
way `fb_Interlock` has always done it. Raising the capacity means editing those together, which is
visible in review.

The removed GVL also took the branch's one real compile risk with it: the array defaults were
written `[GVL_Permissive.cMaxConditions(TRUE)]`, the only use of the array-repetition initializer
anywhere in this library, with a namespace-qualified constant as the repetition factor. The literal
form `[1,1,1,1,1,1,1,1,1,1]` is what `ST_Interlock` has compiled and shipped for years.

**Not a safety function.** Display-and-sequencing only. Real safety lives in the safety PLC and hard
wiring; a condition that must stop a *running* device is an interlock.

## Consumers

- **`abs_fb_MachineFlow` transition gates** — the machine ANDs `stsPermissiveOK` into
  `setPermissive(t, ...)` from the step body. MachineFlow holds **no reference** to this FB; it knows
  only the resulting BOOL. See `fb_MachineFlow_Design.md` → Transition gates for why the earlier
  `POINTER TO fb_Permissive` was removed.
- **`abs_fb_stateDevice.permStartPermissive`** — every sequenced device now carries one, gating entry
  to Running. Wire it with `cmdSetStartPermissive(slot, description, condition)`. This is the reason
  the all-TRUE default is load-bearing rather than merely convenient: seven FBs already extend that
  base in production, and an unwired instance has to mean "satisfied" or adding it would have stopped
  all of them starting. Gates the START only — a dropped permissive never aborts a running device,
  which is what `Interlock` is for.
- **Device start conditions** — a device that should refuse to start rather than fault.
- **lhmi** — the shared `PermissiveList` component binds `ST_Permissive_HMI_Interface` directly, so
  every permissive in a machine gets the same operator view with no per-site UI work.

## Status

**COMPILES.** First TwinCAT build 2026-07-26, against a consuming machine project with a concrete
`abs_fb_MachineFlow` subclass instantiating this FB. No changes to fb_Permissive were needed.

Compiled is not RUN: nothing has yet executed on hardware, so the behaviour — first-out absence,
the count while partially wired, the HMI struct as PermissiveList reads it — is still unexercised.

Both constructs previously flagged here as compile risks were **removed** before that build, not
merely reviewed: the `[GVL_Permissive.cMaxConditions(TRUE)]` array-initialiser became the literal
`[1,1,1,…]` form `ST_Interlock` has shipped for years, and `POINTER TO fb_Permissive` in
`ST_MachineFlow_Transition` went with the pointer gate.
