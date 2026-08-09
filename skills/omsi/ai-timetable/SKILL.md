---
name: omsi-ai-timetable
description: Use when writing or changing behavior for a vehicle that may be driven by AI or is off the player's focus, deciding {frame} vs {frame_ai} placement, handling scheduled bus-stop/target system triggers, or integrating a reusable AI-state/time/timetable/delay script library module
---

# OMSI AI and Timetable Behavior

## Overview

An OMSI vehicle can be driven by the player, by AI while still "belonging" to the player
(e.g. autopilot/AI takeover), or as a fully AI-controlled bus elsewhere in the simulation.
Scripts must decide which entry point runs in each case (`{frame}` vs `{frame_ai}`), honor
AI-facing system variables and system triggers, and (if a schedule is active) drive display
and door behavior from timetable data OMSI provides. This skill covers that behavior layer.

**REQUIRED BACKGROUND:** Load `omsi/foundations` first for RPN/block mechanics, and
`omsi/vehicle-systems` for the door/electrical/engine interlocks AI logic must also respect.

## Scope

- Applies to any macro or trigger whose behavior differs when the vehicle is AI-driven or
  off-focus: `{frame_ai}` blocks, AI-facing system variables (`AI`, `AI_Engine`,
  `AI_Scheduled_AtStation`, `AI_Scheduled_AtStation_Side`, `AI_target_index`,
  `AI_Blinker_L`/`_R`, `AI_Light`, `AI_Interiorlight`, `AI_Brakelight`), and the
  `ai_scheduled_settarget`/`ai_scheduled_setbusstop` system triggers.
- Applies to integrating a reusable script library's AI State, Time, Timetable, and Delay
  modules (where such a library is present in the project) per their documented public
  contract.
- Does not cover display rendering mechanics (`omsi/displays-and-assets`) or the underlying
  door/electrical/engine state machines themselves (`omsi/vehicle-systems`) beyond how AI
  drives them.

## Evidence to inspect before making a change

1. Whether the target vehicle/object family has a separate AI-focus main script (a
   `{frame_ai}` block placed in its own file) or relies solely on `{frame_ai}` blocks inside
   individual subsystem files -- read whichever pattern the family actually uses in full
   before adding new AI logic, rather than assuming one style.
2. Every subsystem the new AI logic must drive (engine, brakes, doors, drivetrain) to
   confirm their `{frame}`-side interlocks (see `omsi/vehicle-systems`) are also satisfied
   by the `{frame_ai}`-side code path -- AI logic is not exempt from those prerequisites.
2b. Whether a separate AI-focus script and an individual subsystem's own `{frame_ai}` block
   both exist for the same family; if so, confirm they do not both write the same variable
   in the same frame (double-application risk).
3. The reusable library's documented module contracts (declarations required, call sites
   required, and which variables are read-only outputs) before wiring in any AI State, Time,
   Timetable, or Delay module macro.
4. The OMSI system-trigger and predefined-variable references for the exact semantics of
   `AI_Scheduled_AtStation`, `ai_scheduled_settarget`, and `ai_scheduled_setbusstop` before
   writing new trigger-handling code -- do not infer semantics from a single script sample
   alone.

## Operating rules

### Deciding {frame} vs {frame_ai}

- `{frame_ai}` is called instead of `{frame}` for vehicles not in the player's focus (e.g.
  AI buses). If a script defines only `{frame}` and no `{frame_ai}`, OMSI calls `{frame}` in
  both cases. Scenery objects never get a `{frame_ai}` entry point.
- Put logic in `{frame_ai}` (or an AI-focus main script's equivalent block) only for
  behavior that must differ when the vehicle is AI-driven: taking pedal/throttle input from
  `AI_Engine`/`Throttle`/`Brake` instead of the player's controls, driving displays from
  `AI_target_index`/`SetLineTo` instead of a player menu, and running the scheduled
  bus-stop door sequence from `AI_Scheduled_AtStation` instead of player key presses.
- Logic that should run identically regardless of AI/player state (most subsystem physics,
  most display refresh logic) belongs in `{frame}` only, or in a macro called from both
  entry points, not duplicated separately in `{frame_ai}`.

### AI engine/brake handoff

- `AI_Engine` is a request from OMSI to the script when the vehicle is AI-driven: -1 means
  "switch the engine off", 0 means "not applicable / not AI", 1 means "switch the engine
  on". The script must translate this into its own engine-start/engine-stop procedure
  (setting `engine_on`, `engine_injection_on`, gear/neutral state, parking brake) rather than
  writing engine state directly from `AI_Engine` without going through the normal
  start/stop sequence -- otherwise the engine subsystem's own interlocks (see
  `omsi/vehicle-systems`) are bypassed.
- A repeated pattern for this handoff sets the parking brake and clears gear/injection state
  together with the engine-on/off transition, rather than changing engine state alone --
  confirm the target subsystem's own start/stop macro is reused or mirrored, not
  reimplemented ad hoc.

### Scheduled bus-stop door handoff

- `AI_Scheduled_AtStation` is bidirectional: OMSI sets it to `1` when the AI vehicle reaches
  a scheduled stop (signal: open doors) and to `-1` when the bus should prepare to depart
  (signal: close doors and get ready). The script must acknowledge readiness by setting it
  back to `0` once doors are closed and the vehicle is genuinely ready to move off; OMSI does
  not clear this variable for the script.
- `AI_Scheduled_AtStation_Side` (where declared) tells the script which side to open,
  relevant for rail vehicles or vehicles with doors on both sides.
- Any custom door-opening logic driven by this handoff must still pass through the same
  speed/brake interlocks used by the player-facing door trigger path (see
  `omsi/vehicle-systems`) -- do not open doors unconditionally just because
  `AI_Scheduled_AtStation` is 1.
- After doors are confirmed closed (all door position variables below the closed threshold),
  release the stop brake before writing `0` back to `AI_Scheduled_AtStation`, matching the
  order the interlock chain expects.

### Scheduled destination/target triggers

- `ai_scheduled_settarget` instructs the script to reset the displayed line/destination.
  The line string arrives via the string variable `SetLineTo`; the destination index (into
  the vehicle's yard/hof data) arrives via `AI_target_index`. Handle both together in the
  same trigger handler -- setting only one leaves the display in an inconsistent state.
- `ai_scheduled_setbusstop` instructs the script to update the current-stop-dependent display
  (e.g. an interior "next stop" indicator). The new stop name/identifier arrives via the
  string variable `act_busstop`.
- `target_index_int` is the variable the script itself should set to reflect which
  destination sign is actually showing, since it also drives passenger behavior; keep it
  synchronized with whatever the display subsystem renders, not just with `AI_target_index`
  (which is OMSI's request, not necessarily the script's final resolved state).

### Reusable AI-state and timetable library modules

- Before calling any reusable AI-state/time/timetable/delay macro from a script library,
  confirm its required varlist entries are registered and its required init/frame call
  sites exist in the target's `{init}`/`{frame}`/`{frame_ai}` blocks, per the library's own
  documented requirements -- these modules intentionally do nothing useful if their
  declaration or call site is missing, and OMSI will not raise a distinct error for a
  silently-absent module.
- Treat every variable these modules expose as read-only from consuming code; the modules'
  own documentation states their outputs should not be written to by other scripts. If a
  consuming subsystem needs a derived value, compute it into a new local variable rather than
  overwriting the module's output.
- The AI-state module's tri-state output (player driving their own bus / AI driving the
  player's bus / AI driving another AI bus) is only ever in the third state if the script
  defines a `{frame_ai}` entry point; a script with only `{frame}` will never see that state
  from this module.
- The timetable and delay modules derive their values from the same underlying system-macro
  calls a script could call directly; prefer the library module once it is already wired in,
  to avoid two independent read paths for the same timetable state that could disagree after
  a schedule change.

## Workflow

1. Determine whether the target vehicle/object family already has an AI-focus code path
   (separate main script, or per-subsystem `{frame_ai}` blocks) and read it in full.
2. For the specific behavior being added or changed, decide whether it truly differs between
   player and AI operation; if not, put it in `{frame}`/a shared macro instead of
   duplicating into `{frame_ai}`.
3. For engine/brake/door handoffs, trace the full request/acknowledge cycle
   (`AI_Engine`/`AI_Scheduled_AtStation`/`AI_Scheduled_AtStation_Side`) end to end, confirming
   the script writes back every acknowledgment OMSI expects.
4. For display/destination changes, confirm both halves of a paired trigger
   (`SetLineTo`+`AI_target_index`, or the stop-name string+the trigger it accompanies) are
   handled together.
5. If introducing a reusable AI/timetable library module, confirm its documented
   declarations and call sites are all present before relying on its output variables
   anywhere else.
6. Re-check that no existing `{frame}`-side interlock (electrical, brake, speed) from
   `omsi/vehicle-systems` was bypassed by the new AI-side code path.

## Validation

Static checks (possible without OMSI):

- Every AI-facing bidirectional variable this code reads is also written back with its
  documented acknowledgment value on the appropriate branch (`AI_Scheduled_AtStation` back to
  `0`, engine-off handoff clearing injection/gear state, etc.).
- `{frame_ai}` logic does not duplicate a state write already made by `{frame}` for the same
  variable in the same pass, and vice versa for a separate AI-focus script versus a
  subsystem's own `{frame_ai}` block.
- Paired trigger data (`SetLineTo` + `AI_target_index`; `act_busstop` + its trigger) is always
  set together, never one without the other in the same handler.
- Any reusable AI/timetable/delay library module referenced has its required varlist
  entries, script file, and init/frame call sites all present in the target configuration --
  flag any that are missing rather than assuming the module is active.

Runtime checks that require OMSI (not verifiable without it):

- Whether an AI-driven vehicle actually reaches, stops at, and departs from a scheduled stop
  correctly end to end, since this depends on OMSI's own AI driving logic and a live
  timetable/route configuration.
- Whether `{frame_ai}` is actually invoked for a given vehicle in a given scenario (depends
  on OMSI's focus-tracking, not statically checkable from script text alone).

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| AI bus never departs from a stop | Script never writes `0` back to `AI_Scheduled_AtStation` after closing doors | Trace the door-closed branch for the acknowledgment write |
| AI bus's destination sign and passenger routing disagree | `AI_target_index` handled but `target_index_int` never updated to match the resolved display state | Confirm both variables are set together after a target-set trigger |
| Engine behaves oddly under AI control only | `AI_Engine` written directly into engine state instead of routed through the normal start/stop macro | Compare AI-side engine handling against the player-side start/stop trigger logic |
| Duplicate or conflicting AI logic | Both a separate AI-focus script and a subsystem's own `{frame_ai}` block write the same variable | Search for the same variable being written in both locations |
| Library AI-state/timetable value never changes | Required module declaration, script file entry, or init/frame call site missing from configuration | Re-check the module's documented requirements against the target configuration |
| Doors open for AI bus regardless of speed | Scheduled door-open path skips the speed/brake interlock the player-facing trigger enforces | Compare the AI door-open branch against the corresponding player trigger's guard conditions |

## Repository starting points

- OMSI's authoritative predefined-variable and system-trigger references, for exact
  semantics of `AI`, `AI_Engine`, `AI_Scheduled_AtStation`, `AI_Scheduled_AtStation_Side`,
  `AI_target_index`, `target_index_int`, `SetLineTo`, `act_busstop`,
  `ai_scheduled_settarget`, and `ai_scheduled_setbusstop`.
- A library's documented AI-state, time, timetable, and delay module contracts, where such a
  reusable library is available in the project, for their exact declaration and call-site
  requirements before wiring in a new module.
- Existing AI-focus main scripts or `{frame_ai}` blocks in the target vehicle/object family,
  as the concrete pattern to extend for engine handoff, door handoff, and display updates.

## Uncertainty and escalation

- If a family has no AI-focus script or `{frame_ai}` block at all, do not assume one is
  required -- confirm whether the family is ever used as an AI vehicle before adding AI-only
  logic.
- If the exact acknowledgment timing for a bidirectional variable is ambiguous from the
  variable reference alone, cross-check an existing working AI-handoff sequence in the same
  family before writing a new one from scratch, and flag remaining ambiguity rather than
  guessing.
- Do not claim an AI/timetable behavior change "works" in-sim; only that it is internally
  consistent by the static checks above. Confirming actual AI driving/stop/departure behavior
  requires OMSI and a live route/timetable configuration outside static script review.
