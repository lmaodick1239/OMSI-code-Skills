---
name: omsi-debugging-performance
description: Use when isolating a fault in OMSI script behavior, adding temporary diagnostics, or evaluating a low-risk performance optimization, before proposing a fix based on assumption rather than evidence
---

# OMSI Debugging and Performance

## Overview

OMSI script has no debugger, breakpoints, or exception stack traces. Fault isolation
relies on static re-reading of the script (stack tracing, block matching, call-order
checks) combined with a small set of runtime diagnostics OMSI exposes: debug display
variables, a string debug line, and a stack-dump command. This skill covers systematic
fault isolation and safe, narrow performance changes; it does not replace running OMSI
itself, which this repository cannot do.

**REQUIRED BACKGROUND:** Load `omsi/foundations` first. Combine with the specialized
skill for the subsystem under investigation (`omsi/vehicle-systems`,
`omsi/events-and-variables`, `omsi/ai-timetable`, `omsi/displays-and-assets`,
`omsi/script-architecture`) once the fault is localized.

## Scope

- Applies to isolating incorrect or unexpected script behavior, adding/removing temporary
  diagnostic instrumentation, and making narrow, low-risk performance changes to existing
  script logic.
- Does not cover writing new subsystem behavior from scratch (see the subsystem-specific
  skills) or configuration registration (`omsi/configuration-and-reuse`).

## Evidence to inspect before making a change

1. The exact macro/trigger/block where the symptom manifests, read in full alongside every
   macro it calls and every variable it reads.
2. Every producer of a variable implicated in the symptom (search for every
   `(S.L.name)`/`(S.$.name)` write site), not just the consumer showing the symptom.
3. The subsystem's declared dependencies (a `'Needs:'`-style header comment, if the family
   uses one) to rule out an uninitialized or not-yet-updated upstream value.
4. Whether a performance concern is about per-frame cost (something running every
   `{frame}`/`{frame_ai}`) versus a one-time `{init}` cost; these need different treatment.

## Operating rules

### Fault isolation workflow

- Reproduce the state transition as a sequence of variable values, not just a single
  symptom description: "what should X be at frame N, what is it actually, and what wrote
  it last."
- Work backward from the incorrect value to its last writer, then to that writer's own
  inputs, one hop at a time. Do not jump straight to "fix" a suspected line without first
  confirming it is actually reached and actually wrong via this trace.
- Identify the owning macro of the faulty state before editing anything -- if the value is
  owned by a different subsystem than the one exhibiting the symptom, the fix belongs in
  the owning subsystem, not a workaround in the consumer.

### Diagnostics available inside OMSI script

- `Debug_0` through `Debug_5` (or a similar family of debug-display local variables, if
  present in the target vehicle's variable list) can be written from any macro and read in
  OMSI's debug-mode information bar. Use them to expose an intermediate stack/variable
  value without altering program logic.
- `$msg` writes the top string-stack value to OMSI's debug line; use it for a one-off
  string/condition check, then remove it.
- `%stackdump%` opens a dialog with the current float stack contents; treat it as a
  last-resort, single-use probe (it is intrusive/blocking), not something to leave in
  committed script.
- All three are debug-only affordances: they must be removed (or clearly marked temporary)
  before considering the change complete, since they add per-frame overhead and can leak
  into the shipped debug display.

### Static checks (apply `omsi/foundations` rules exhaustively)

- Case-exact match of every macro/trigger/variable name against its declaration.
- Block balance: one `{end}` per entry point, one `{endif}` per `{if}`.
- Call-order: every macro call site precedes its `{macro:...}` definition in load order.
- Stack-effect re-derivation for every changed expression, on both the float and string
  stacks independently.
- Constant/curve binding: confirm a constant or curve name used in the faulty expression
  is defined in the constfile actually registered for the target vehicle/object, not a
  sibling variant file.
- Asset-path references (sound files, textures, fonts) resolve relative to the correct
  base directory for their access form; do not assume a path that "looks right" without
  checking the convention used elsewhere in the same file.

### Runtime test matrix (requires OMSI; plan it even if it cannot run here)

When a fix is ready to validate in OMSI, exercise it across the combinations relevant to
the subsystem, for example:

| Axis | States to cover |
|---|---|
| Script lifecycle | `{init}` cold start, first few `{frame}` calls, long-running session |
| Control mode | User-driven vs. AI-driven (`{frame}` vs `{frame_ai}`, if the family has both) |
| Electrical state | Supply off, supply on, mid-transition |
| Motion state | Stationary, accelerating, at speed, braking to stop |
| Trigger timing | Key/mouse press, hold/drag (if applicable), release/off |
| Display/timetable state | No active timetable, active timetable, at a scheduled stop |

State explicitly which of these combinations could not be exercised because this
repository lacks the target vehicle/object package or an OMSI executable.

### Low-risk performance discipline

- Scale any continuous accumulator by the system variable representing elapsed frame time,
  not a fixed per-frame constant, so behavior stays consistent across framerates (see
  `omsi/vehicle-systems` for the accumulator-clamping pattern this implies).
- For script-texture-heavy display logic, gate expensive redraw work behind a "did the
  underlying value actually change" check rather than redrawing every frame regardless of
  change (see `omsi/displays-and-assets` for the lock/modify/unlock sequencing this
  interacts with).
- Preserve texture lock/unlock pairing exactly; do not add an early return or branch that
  could skip an unlock after a lock, since that leaves the texture in a locked state.
- Do not restructure unrelated code while chasing a performance issue -- a performance
  fix should be as narrow as a bug fix.

## Workflow

1. Restate the symptom as an expected-vs-actual variable value at a specific point in
   script execution.
2. Trace backward from the wrong value to its last writer; repeat until reaching a root
   cause (an incorrect condition, a missing prerequisite check, an ordering problem, or a
   genuinely wrong constant/curve value).
3. Apply the static checks above to the root-cause location and its immediate neighbors.
4. If the root cause is not yet certain, add the minimum debug instrumentation
   (`Debug_#`/`$msg`) needed to confirm it, note that it is temporary, and plan its removal.
5. Make the narrowest fix that addresses the confirmed root cause.
6. Remove all temporary instrumentation added during isolation.
7. Re-run the full static checklist on the final diff.
8. Describe the runtime test matrix relevant to the fix, and state plainly which parts of
   it this repository can and cannot exercise.

## Validation

Static (possible from this repository alone): the full checklist in "Static checks" above,
plus confirming no temporary debug instrumentation remains in the final diff unless the
user explicitly asked for it to stay.

Runtime checks that require OMSI (not verifiable from this repository):

- Whether the fix actually resolves the symptom in-sim, across the test matrix above.
- Actual per-frame cost/framerate impact of a performance change -- this repository has no
  profiler or OMSI runtime to measure it.
- Whether removing debug instrumentation changes timing-sensitive behavior (it should not,
  but this can only be confirmed by running OMSI).

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| Fix "should" work but symptom persists | Root cause traced to the wrong writer; another producer overwrites the value later in the same frame | Search for every writer of the variable again, including subsystems not yet considered |
| Debug variable shows correct value but display/behavior still wrong | The debug probe was placed after the point where the actual consumer already read a stale value | Move the probe to just before the consumer reads the value, not just after the suspected writer |
| Fix works for user-driven case but not AI case | Change made only in a `{frame}` block; the AI path uses a separate `{frame_ai}` block or separate AI main script | Confirm whether the family has a separate AI-focus path and apply the fix there too |
| Performance change alters visible behavior | Redraw/update gating skipped a legitimate state change, not just a redundant one | Confirm the gating condition covers every variable the redraw actually depends on |
| Leftover `$msg`/`Debug_#` writes in committed script | Temporary instrumentation not removed after isolation | Diff review specifically for debug-only lines before finishing |
| Texture left inconsistent after an edit | An early exit/branch introduced between a lock and its matching unlock | Trace every code path between a lock call and its unlock call for early returns |

## Repository starting points

Consult the following skills for the subsystem-specific evidence and file families to
inspect once a fault is localized:

- `omsi/foundations` for the language-level static checks this skill assumes.
- `omsi/vehicle-systems` for door/electrical/engine/drivetrain/brake/cockpit fault patterns
  and Timegap-scaled accumulator discipline.
- `omsi/events-and-variables` for variable ownership and trigger lifecycle checks.
- `omsi/ai-timetable` for `{frame}`/`{frame_ai}` duplication and AI-handoff faults.
- `omsi/displays-and-assets` for script-texture lock/unlock and stack-contract faults.
- `omsi/script-architecture` for call-order and macro-not-found faults.
- `omsi/configuration-and-reuse` for constant/curve/asset registration mismatches.

## Uncertainty and escalation

- Do not report a fix as "resolved" based on static inspection alone; state explicitly that
  it is internally consistent by the static checklist and that in-sim confirmation requires
  OMSI, which this repository cannot run.
- If the root cause cannot be isolated without seeing the target vehicle/object's actual
  registered configuration (which constfile, which script variant, which model asset), ask
  for that file rather than guessing which sibling variant is in play.
- If a performance concern's actual framerate impact matters for the decision, say so and
  note that only an in-sim profiling session can measure it.
