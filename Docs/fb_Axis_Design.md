# fb_Axis — Design Document

## Purpose

Define a single, reusable axis wrapper function block (fb_Axis) that serves as the standard motion building block across multiple applications. The block must be lean and general enough to work cleanly in simple single-axis applications, while exposing the right hooks to support more complex applications (coordinated/cammed motion, queued and blended moves) through optional higher-level layers.

This document specifies *what* to build and *which* hooks to reserve. It does not prescribe implementation details, naming, or formatting — those follow the existing library style guide and reuse existing library blocks where available.

---

## Design Principles

1. **One FB for all single-axis modes.** A single fb_Axis handles position, velocity, and torque control rather than separate FBs per mode. The common foundation (power, fault handling, reset, status, safety gating, drive interface) is shared; mode-specific behavior is selected at runtime.

2. **Internal persistent move table.** The move definitions live inside each fb_Axis instance as a persistent structured table, rather than being supplied by the caller as scalar inputs or an external array. The FB face stays minimal; HMI edits are staged in a config buffer and copied into the live persistent table only when accepted.

3. **fb_Axis stays single-axis and lean.** Coordination (camming, gearing) and multi-move sequencing (queuing) live in optional higher-level layers, not in fb_Axis. fb_Axis exposes the minimal hooks those layers need to take control, but contains none of their logic.

4. **Forward-compatible interface, incremental implementation.** The interface (command struct, status struct, mode enumeration) should be designed up front to be capable of the full feature set. Bodies for advanced features may be stubbed and implemented later without breaking the interface.

5. **Works for virtual axes.** fb_Axis must function for a virtual (driveless) axis as well as a physical one, since coordinated applications use a virtual master axis instantiated from the same FB.

---

## Architecture / Layering

The design is layered. fb_Axis is the bottom layer; the upper layers are optional and application-specific.

- **fb_Axis (core, this document):** single axis, one active move at a time (with provision for a pending move to support blending), runtime mode selection, power/fault/reset lifecycle, safety gating, status reporting. Yields control to an external owner when commanded (for coordination).

- **Sequencer layer (optional, future):** owns a deeper move queue and feeds fb_Axis one move at a time, using fb_Axis status to know when the next move can be accepted. Applications that do not need queuing use fb_Axis directly.

- **Coordination layer (optional):** owns master/slave relationships (camming, gearing, phasing). It commands coupling on the relevant axes and sets the appropriate fb_Axis instances into a yielded/external-control state so they do not issue independent motion while coupled. Coordinated applications instantiate one fb_Axis per physical axis plus one per virtual master, and add this layer on top.

This layering keeps fb_Axis general: simple applications use it bare; complex applications add the layers they need.

---

## Core Motion Functions (build into fb_Axis now)

These are the everyday single-axis behaviors and should be implemented in the core block:

- **Power / enable control** — enabling and disabling the axis.
- **Reset / fault clearing.**
- **Absolute positioning.**
- **Relative positioning.**
- **Velocity control.**
- **Torque/force control.**
- **Controlled stop (interruptible / resumable).**
- **Controlled stop (forced standstill / locked until reset).** Both stop behaviors are required and are distinct; the interlock/fault handling must select the correct one (forced/locked stop for safety events, interruptible stop for normal sequencing pauses).
- **Velocity/dynamics override** — runtime scaling of the axis's commanded velocity/acceleration. Required for global speed control of coordinated motion when applied to a master axis.
- **Status decoding / reporting** — translate the underlying axis state into the block's status output for HMI and higher-level logic.

---

## Drive / NC Requirements

Torque mode requires the EtherCAT drive terminal and NC axis to support cyclic torque operation. For an ELM7231-style CiA 402 axis, configure the terminal/NC so `MC_TorqueControl` can switch the drive into continuous torque mode (CST, modes-of-operation value 10) and stream the target torque cyclically. The EtherCAT PDO/CoE setup must expose the required control/status objects for NC operation, including mode switching, target torque, torque ramp/slope where supported, and actual torque feedback.

This is a hardware/NC configuration requirement, not FB logic. fb_Axis supplies `ContinuousUpdate := TRUE`, `Torque`, `TorqueRamp`, and `Direction` to `MC_TorqueControl`; the NC/drive configuration must map those commands to the terminal's supported CST PDO/CoE objects.

---

## Reserved Functions / Hooks (interface now, implement as needed)

Reserve room in the command struct, mode enumeration, and/or methods for the following, so they can be implemented later without interface changes:

- **Modulo positioning** — move to a rotary position with shortest-path or forced-direction options. Relevant to rotary/wrap axes.
- **Homing / referencing** — required for incremental and virtual axes even though physical axes use absolute feedback.
- **Set actual position / re-zero** — set the axis position without motion. Needed for resetting a virtual master between cycles and for re-referencing.
- **Runtime parameter read/write** — read and write axis parameters at runtime (for live tuning, recipes, adjustable limits) without offline configuration changes.
- **Buffer mode / move blending** — see dedicated section below.
- **Touch-probe / position latching** — capture axis position on a trigger (e.g. registration). Reserve only if registration-type features are on the application roadmap.
- **Time-driven profile following** — following a custom position/velocity/acceleration profile table. Reserve only if a use case beyond camming is anticipated.

---

## Move Blending / Buffering (special note)

Move blending is the one advanced feature whose minimal mechanism must live inside fb_Axis rather than a higher layer, because the underlying motion library implements blending as a per-axis mechanism requiring the next move to be armed (with its buffer mode) while the current move is still active.

Implications for fb_Axis:

- The command struct must carry a **buffer-mode** selector (aborting vs buffered vs the blending variants).
- fb_Axis must be able to hold a **current move plus one pending move**, accepting the pending command while the current move is still active and arming it with its buffer mode so the underlying motion engine performs the transition.
- The status output must signal when fb_Axis is **ready to accept the next move**, so a sequencer (or application code) knows when to push the next blended command.

This may be **stubbed initially** (single-move behavior, buffer-mode field present but only "aborting" honored) and completed later, provided the interface (buffer-mode field, "accepting next" status) is in place from the start.

---

## Coordination Hooks (for the optional coordination layer)

To support camming/gearing via a higher layer without embedding coordination logic in fb_Axis, the core block must provide:

- **An external-control / yielded state** — an input (or equivalent mechanism) that tells fb_Axis a coordinator now owns the axis. While yielded, fb_Axis maintains power, safety gating, and status reporting but does not issue its own motion commands, so it neither fights nor interferes with externally commanded coupled motion.
- **Shared access to the underlying axis reference** — so the coordination layer can command coupling on the same axis. Only one source commands motion at a time; the yielded state enforces this.
- **A "coupled / externally controlled" status indication** — so HMI and higher logic know the axis is following rather than self-directed.

Note that gearing may be the simpler coupling mechanism for fixed-ratio master/slave relationships, while camming covers profiled relationships; both live in the coordination layer.

---

## Interlock / Safety Model

Two distinct interlock scopes must be supported, each placed according to its responsibility:

- **Axis-level interlock** — gates whether the axis may operate *at all*, independent of any specific move (e.g. safety state, drive readiness, referenced state). This belongs as an input to fb_Axis. Prefer a **structured input** that carries the individual contributing conditions rather than a single boolean, so the specific blocking cause can be surfaced in status/diagnostics. Loss of an axis-level permissive during motion must trigger the appropriate forced/controlled stop, and — when the axis is under external/coordinated control — must be able to command a controlled decouple via the coordination layer.

- **Move-level interlock** — gates whether a *specific* commanded move is permitted (application-specific permissive conditions). This belongs in the move command struct, checked at execution and optionally monitored continuously.

A move executes only when the axis-level permissive, the move-level permissive, and the fault state all allow it.

---

## Interface Elements to Define

The following interface elements should be defined up front (per the style guide and reusing existing library types where they exist):

- **Mode enumeration** covering the core modes and reserved modes (position absolute/relative, velocity, torque, modulo, homing, stop variants, and an external/yielded indication as appropriate to the chosen mechanism).
- **Move definition struct** carrying mode selection, mode-specific setpoints (target, velocity, dynamics, torque, ramp, etc.), buffer-mode selector, and move-level interlock. fb_Axis stores an internal persistent array of these definitions and executes a selected index.
- **Axis-level safety/permissive struct** carrying the individual contributing conditions.
- **Status struct** carrying operational state (busy/active/done/in-target/in-torque), readiness to accept the next move (for blending/sequencing), coupled/external-control indication, error state and error identifier, and the decoded information needed by HMI faceplates.
- **The underlying axis reference** as a shared in/out element.

---

## Build Phasing (suggested)

1. **Phase 1 (core):** power, reset, absolute/relative/velocity/torque, both stop variants, override, status decoding, two-scope interlock gating. Single move at a time. Interface includes all reserved fields and the blend/coordination hooks, stubbed.
2. **Phase 2:** blending (current + pending move, buffer mode honored, accepting-next status), set-position/home, runtime parameter access, modulo.
3. **Phase 3 (application layers, separate blocks):** sequencer (queue), coordination (cam/gear/phasing) using the yielded-control hook.

---

## Out of Scope for fb_Axis

The following are explicitly *not* part of fb_Axis and belong to higher layers or separate diagnostic blocks:

- Camming, gearing, phasing, and cam scaling/adjustment logic (coordination layer).
- Multi-move queue management (sequencer layer).
- Drive-level diagnostic acquisition beyond standard status (e.g. temperature, thermal-load reads) — separate diagnostic concern.
