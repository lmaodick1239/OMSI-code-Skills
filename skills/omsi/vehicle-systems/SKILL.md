---
name: omsi-vehicle-systems
description: Use when modifying vehicle behavior scripts for doors, electrical supply, engine, drivetrain, brakes, or cockpit state, before changing state-machine interlocks or continuous simulation values
---

# OMSI Vehicle Systems

## Overview

OMSI vehicle behavior is commonly implemented as a set of cooperating subsystem scripts
(door, electrical, engine, drivetrain, brakes, cockpit) that share state through local
variables and enforce prerequisites on each other: electrical supply before doors move,
brake/door interlocks before a vehicle is released to AI, and so on. This skill covers how
to change that behavior without breaking a prerequisite another subsystem relies on.

**REQUIRED BACKGROUND:** Load `omsi/foundations` first for RPN/stack mechanics, and
`omsi/script-architecture` for how subsystem `.osc` files are wired into a main script.

## Scope

- Applies to vehicle-behavior subsystems such as door control, electrical supply, engine,
  drivetrain/gearbox, brakes, and cockpit state, and their matching varlist/constfile
  declarations.
- Does not cover display/IBIS scripts (`omsi/displays-and-assets`), AI-only frame logic
  (`omsi/ai-timetable`), or configuration file registration (`omsi/configuration-and-reuse`).

## Evidence to inspect before making a change

1. The target subsystem's own script file in full, plus its variable list and constant
   file. Many subsystem scripts declare their cross-subsystem dependencies in a header
   comment (for example a "Needs:" list naming the subsystems they depend on) -- read this
   before editing.
2. Every other subsystem file that reads or writes the same variable name (search by exact,
   case-matched identifier across the target vehicle's registered scripts).
3. The vehicle's authoritative performance/physics documentation for the target simulator if
   the change affects wheel torque, braking force, or engine speed/torque curves.
4. Sibling variants of the same subsystem (multiple door, drivetrain, or gearbox
   implementations are common in a bus mod project) to confirm which variant is actually
   registered by the target vehicle's configuration file before copying from it -- see
   `omsi/configuration-and-reuse` for registration checks. Do not assume a file is canonical
   just because it has the shortest or most common-looking name.

## Operating rules

### Identify the owning subsystem before touching a shared variable

- Variables that gate cross-subsystem behavior (electrical bus-bar availability, stop-brake
  state, engine-on state, per-door open state) are written by one subsystem and read by
  several others. Before writing to one of these, confirm from the writing subsystem's own
  file which conditions gate the write, and preserve those conditions in the new code path
  rather than writing to the variable directly from an unrelated subsystem.
- A common pattern for an externally-triggered action (for example a door open request that
  can come from outside the normal driver control) is to explicitly re-check the
  prerequisite (such as electrical availability) before delegating to the shared macro that
  performs the action, rather than assuming the prerequisite was already satisfied by the
  caller.

### Preserve state-machine prerequisites

- Electrical supply commonly gates doors and other electrically-dependent behavior: a
  subsystem's frame macro should check bus-bar/battery availability before acting on a
  door, light, or accessory trigger.
- Brake/door interlocks: a stop-brake or "at station" brake variable is typically set only
  when vehicle speed is below a threshold constant and the vehicle is intentionally
  stationary. Do not set door-open state without routing through the same speed/brake check
  the surrounding subsystem already uses.
- Timers: any per-frame timer variable should be incremented by the frame's elapsed time
  (see Timegap-scaled changes below), not by a fixed per-frame constant.
- AI handoff: a bidirectional "at station" style variable is often used so the simulator's
  AI can signal intent (open doors / prepare to depart) and the script acknowledges back
  once ready. Preserve this handshake in both directions; do not just read the request, also
  write the acknowledgment back once the vehicle-side condition is satisfied (see
  `omsi/ai-timetable` for the full contract and `omsi/events-and-variables` for the
  authoritative variable semantics).

### Timegap-scaled continuous changes

- Any value that should change smoothly over real time (fuel level, timers, gradual
  approach to a target) must be scaled by the system variable for elapsed frame time
  (commonly named `Timegap`), not incremented by a fixed per-frame amount. A typical pattern:
  ```
  {trigger:refuel}
  	(L.L.fuel_content) (L.S.Timegap) 3 * + 250 min (S.L.fuel_content)
  {end}
  ```
  Note the clamp (`min`/`max`) after the Timegap-scaled increment -- always bound a
  Timegap-scaled accumulator to a sane range, since frame time can vary.
- A reusable "approach a target exponentially" macro (register-based: target, current
  value, positive approach constant, negative approach constant) is a common utility
  pattern for smoothing continuous values (needle sweeps, gradual light dimming, temperature
  drift). Prefer reusing an existing equivalent macro in the target codebase over writing a
  new ad hoc smoothing formula, if one already exists for the same purpose.

### Separate constant/curve tuning from code-level changes

- In OMSI's documented physics model, the simulator core computes tire/road interaction and
  vehicle motion; the script computes wheel torque and braking force, which it writes back
  to the simulator via system variables. Torque and speed-related tuning (engine power curve
  shape, idle/max RPM, gear ratios) belongs in the relevant constant file (`[const]`,
  `[newcurve]`/`[pnt]`), not hardcoded into the script macro body, unless the change is
  structural (e.g. adding a gear) and cannot be expressed as a curve/constant edit.
- When editing a curve, keep `[pnt]` entries in strictly ascending x-order (see
  `omsi/foundations`); OMSI clamps flat outside the defined range rather than extrapolating.

### Units and variable relationships for wheel torque/braking

Per OMSI's documented predefined vehicle variables:

| Variable | Meaning | Unit |
|---|---|---|
| `M_Wheel` | Torque applied to driven axles (script writes; sum across all driven axles) | kNm |
| `n_Wheel` | Average wheel rotational speed (read-only from script) | rpm |
| `Brakeforce` | Total braking force for all wheels together (do not combine with `Axle_Brakeforce_*` for the same wheel set) | N |
| `Axle_Brakeforce_#_L`/`_R` | Per-wheel braking force | N |
| `Throttle` / `Brake` / `Clutch` | Pedal positions (read-only from script) | 0..1 |
| `Velocity` | Speedometer-derived speed | km/h |

Driving force (engine, retarder, hydraulic/electric brakes) can act in any direction;
braking force (friction brakes, rolling/air resistance) can only remove kinetic energy from
the vehicle. Do not model a "negative driving force" as a substitute for a proper
braking-force variable, since this is physically wrong at the moment the vehicle comes to
rest (braking force cannot add energy back into the vehicle, but a driving-force sign flip
implicitly could).

## Workflow

1. Read the target subsystem file end to end; note any "Needs:" style header comment to
   identify cross-subsystem dependencies before editing.
2. Search for every other subsystem reading/writing the variable(s) the edit touches.
3. Confirm whether the change is a constant/curve tuning change (goes in the constant file)
   or a structural behavior change (goes in the macro body).
4. Apply `omsi/foundations` stack-tracing and block-balance discipline to the edited
   expression.
5. If the change affects a Timegap-scaled accumulator, confirm a `min`/`max` clamp is
   present and bounded to a value consistent with the surrounding code's own conventions.
6. Confirm the edit does not remove or bypass an existing prerequisite check
   (electrical/brake/speed) found in the unmodified surrounding code.

## Validation

Static checks (possible without running OMSI):

- Confirm every prerequisite check present before the edit is still present after it.
- Confirm Timegap-scaled values have a bound (`min`/`max`) matching the surrounding
  convention.
- Confirm curve `[pnt]` ordering after any constant-file edit (see `omsi/foundations`).
- Confirm variable read/write access forms match declared types across all consumer files.

Runtime checks that require OMSI (not verifiable statically):

- Actual clutch engagement feel, gear-change timing, and engine sound response to a tuning
  change -- engine/gearbox tuning is inherently an iterative, in-sim process.
- Whether a new door/brake interlock path behaves correctly across the full range of
  vehicle speeds and AI states, which requires the vehicle's complete configuration and the
  OMSI executable to exercise.

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| Doors respond even with electrical supply off | New door trigger/path skips the electrical-availability check present in the reference pattern | Compare the new trigger against the subsystem's existing externally-triggered door pattern |
| Vehicle creeps or jitters when stationary | Stop-brake interlock bypassed or speed threshold constant not honored | Check the stop-brake speed-threshold constant usage and velocity comparison before setting brake state |
| Fuel/timer value jumps unpredictably at low framerate | Accumulator incremented by a fixed amount instead of Timegap-scaled, or missing clamp | Confirm `Timegap` multiplication and a `min`/`max` clamp |
| Torque/brake force tuning has no effect | Edited the script macro body instead of the constant file the vehicle's configuration actually registers | Confirm which constant file is registered by the target vehicle configuration before editing |
| AI vehicle never releases from a stop | Bidirectional "at station" acknowledgment (setting the request variable back to its neutral value) omitted after doors close | Confirm the door-closed branch writes the acknowledgment back per the bidirectional contract |

## Repository starting points

- The door subsystem script family -- trigger/macro pattern with electrical and brake
  interlocks.
- The engine subsystem script family -- start/stop sequencing, Timegap-scaled fuel content,
  electrical-success dependency.
- The collision/malfunction subsystem -- failure-state pattern shared across electrical,
  engine, and drivetrain subsystems.
- The drivetrain/gearbox script family and its constant files -- compare any sibling
  variants against the one actually registered by a target vehicle before reusing.
- The brake subsystem script and its constant files -- pressure circuits and interlocks.
- The simulator's authoritative vehicle-performance documentation -- theory for wheel
  torque, braking force, torque curves, and the split between core-engine and script
  responsibilities.

## Uncertainty and escalation

- If a tuning or behavior change's in-sim effect (clutch feel, gear-change smoothness,
  engine sound response) cannot be confirmed without OMSI, say so explicitly rather than
  asserting the change "improves" performance.
- If multiple sibling variants of a subsystem exist and it is unclear which one a target
  vehicle registers, ask for the target vehicle configuration file rather than guessing
  which variant to edit.
- Do not assume an undocumented variable's unit or sign convention; cross-check against
  `omsi/events-and-variables` and the authoritative predefined-variable reference, or flag
  it as unverified.
