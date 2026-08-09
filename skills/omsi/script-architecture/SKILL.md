---
name: omsi-script-architecture
description: Use when organizing or extending a multi-file OMSI vehicle or scenery object script, adding a new subsystem file, deciding {init}/{frame}/{frame_ai} orchestration, or reordering script load order in a configuration file
---

# OMSI Script Architecture

## Overview

A complete OMSI vehicle script is split across a main script and per-subsystem `*.osc`
files. The main script's `{init}`/`{frame}`/`{frame_ai}` blocks call subsystem macros; the
subsystems define those macros later in the load order. This skill covers that
orchestration layer. Load `omsi/foundations` first for the underlying language rules
(especially the call-before-definition rule this entire architecture depends on).

## Scope

- Applies to main-script structure (`{init}`, `{frame}`, `{frame_ai}`, top-level
  `{trigger:...}` blocks that dispatch across subsystems) and to naming/ordering of
  subsystem `*_init`/`*_frame` macros.
- Does not cover the configuration-file registration mechanics themselves (script list,
  variable-list entries, etc.) -- see `omsi/configuration-and-reuse` for that.
- Does not cover the internal behavior of any one subsystem -- see `omsi/vehicle-systems`,
  `omsi/ai-timetable`, `omsi/displays-and-assets`.

## Evidence to inspect before making a change

1. The main script for the target vehicle/object family, read in full. If more than one
   main-script variant exists for the family (a common pattern in bus mods -- for example a
   base variant plus a variant with extra display/controller modules wired in), compare
   them before assuming either is canonical.
2. The AI-focus counterpart, if the vehicle family has one -- a main script or subsystem
   file dedicated to `{frame_ai}` logic (see `omsi/ai-timetable` for its content).
3. The subsystem file(s) whose macros the main script calls, to confirm the macro name and
   its expected call position exist and match exactly (case sensitive).
4. The target vehicle's script list in its vehicle/object configuration file, if present,
   to confirm actual load order. If that configuration file is not available for review,
   state explicitly that load order cannot be confirmed rather than assuming it from the
   main-script call sequence alone.
5. Any reusable script library's documentation, if such a library is involved, for its
   documented call-order requirement.

## Operating rules

### Main script responsibilities

- The main script's `{init}` block should contain only a sequence of `(M.L.subsystem_Init)`
  calls (a widely observed convention in OMSI bus scripts). Do not put raw variable
  initialization directly in the main `{init}` block if the corresponding subsystem file
  exists -- put it in that subsystem's `*_Init` macro instead.
- The main script's `{frame}` block similarly should be a sequence of
  `(M.L.subsystem_Frame)` calls. Some real-world main scripts also contain a few direct
  debug-variable writes after the subsystem calls, or small inline cross-subsystem
  handoff logic (for example writing several related variables together when the AI takes
  over a vehicle) -- treat this as an observed pattern for small cross-cutting logic, not a
  rule to imitate for new subsystem-scale logic.
- Top-level system triggers that fan out to multiple subsystems (for example a collision
  trigger or a malfunction-reset trigger) are typically defined in the main script and call
  into subsystem macros. See `omsi/events-and-variables` for system trigger semantics.

### Subsystem file conventions

- Per the OMSIWiki "Scripting System" reference: each subsystem should have its own
  `*.osc` file and, as needed, its own variable-list/string-variable-list/constant file.
  Key-command triggers belonging to that subsystem are defined in that subsystem's file,
  not the main script.
- Macro naming: the wiki recommends `{macro:subsystem_frame}` and `{macro:subsystem_init}`.
  Real-world OMSI script collections vary in exact capitalization by author and era (for
  example `Door_Init`/`Door_Frame` in one family's door script vs `engine_init`/
  `engine_frame` in an engine script, or a `rhlib_`-prefixed lowercase style in a reusable
  library). Match the exact case already used by the subsystem being extended; do not
  normalize casing across files in the same change.
- A subsystem file often declares a `'Needs:'` header comment listing its dependencies (for
  example a door script stating it needs the electrical subsystem, or a collision script
  stating it needs electrical, engine, and drivetrain subsystems). Read these before
  editing a subsystem that depends on another, and verify the depended-on subsystem's
  `_Init`/`_Frame` macro actually runs earlier in the same frame if execution order matters.

### Call-before-definition consequence for file order

- Because a macro must be called before its `{macro:...}` block is defined (see
  `omsi/foundations`), the main script -- which calls every subsystem macro -- must be
  registered *before* every subsystem file in the configuration's script list. Subsystem
  files must, in turn, be ordered so that any cross-file macro call (subsystem A calling a
  macro defined in subsystem B) still precedes B's definition in the concatenated load
  order.
- A reusable script library's documented rule can follow the same principle in the other
  direction for a library: library script files generally come last in the script list,
  while calls to the library's init/frame macros occur early, so that variables are set
  before any scripts try to access them. This means library module macros are called early
  (in the main script or an early subsystem) but the library's own `.osc` files are listed
  last so their definitions come after all the calls that need them, and after other
  subsystems the library modules may read from.

## Workflow

1. Identify the vehicle/object family's main script and confirm whether it has a separate
   AI-focus main script or relies solely on `{frame_ai}` blocks inside subsystem files.
2. Decide whether the new/changed logic belongs in a new subsystem file, an existing
   subsystem's `_Init`/`_Frame` macro, or (rarely, for narrow cross-cutting reads) directly
   in the main script's block, matching the established pattern in the codebase.
3. If adding a new subsystem file: name its init/frame macros consistently with the
   subsystem's own naming style, add the corresponding `(M.L.NewSubsystem_Init)` /
   `(M.L.NewSubsystem_Frame)` calls to the main script's `{init}`/`{frame}` blocks, and add
   `{frame_ai}` wiring only if the subsystem must behave differently when the vehicle is AI-
   or off-focus-driven (see `omsi/ai-timetable`).
4. Confirm the new file's dependencies (`'Needs:'`) are satisfied by subsystems that already
   run earlier in the same frame, or document that load order cannot be confirmed without
   the target configuration file.
5. Update the target script list order accordingly (see `omsi/configuration-and-reuse`) --
   this skill only covers *why* an order is required, not the configuration syntax.
6. Re-verify call-before-definition holds for every macro touched, across every file order
   change.

## Validation

Static checks (possible from source reading alone):

- Every `(M.L.name)` call site in the main script has a matching `{macro:name}` definition
  somewhere in a subsystem file expected to load after it.
- The main script's `{init}`/`{frame}` blocks contain only subsystem macro calls plus any
  narrow cross-cutting logic that matches an established precedent elsewhere in the family.
- If a separate AI-focus main script exists for the family, confirm its `{frame_ai}` block
  does not duplicate logic already present in a subsystem's own `{frame_ai}` block (would
  cause double-application of a state change).
- If the change spans multiple main-script variants for the same family, decide explicitly
  whether all variants need the same edit, and say so if only one file was updated.

Runtime checks that require OMSI (not verifiable from source reading alone):

- Actual macro/script load order as resolved by OMSI from a target configuration file's
  script list, when that configuration file is not available for review.
- Frame-order interaction between subsystems (for example whether a door subsystem's frame
  macro truly executes after an electrical subsystem's frame macro in a given frame) --
  OMSI's exact intra-frame macro execution order beyond what a static call trace shows is
  not independently confirmable without running OMSI.

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| "Macro not found" for a subsystem macro | New subsystem file listed before the main script, or after a file that calls it, in the script list | Compare call site file position against definition file position in configuration order |
| Subsystem reads a stale/zero value from a dependency | Dependency's `_Init`/`_Frame` macro not called before the dependent subsystem's macro in the same block | Check `'Needs:'` comment and confirm call order in main script's `{init}`/`{frame}` |
| Duplicate or conflicting AI logic | Both a separate AI-focus main script and a subsystem's own `{frame_ai}` block implement the same state transition | Search for the same variable being written in both locations |
| Change applied to one main-script variant only | Vehicle family has multiple main scripts and only one was edited | Search the family's directory for other main-script files before finishing |
| New subsystem never initializes | `(M.L.NewSubsystem_Init)` call omitted from `{init}` even though `_Frame` was wired into `{frame}` | Confirm both `_Init` and `_Frame` calls exist unless the subsystem has no init-time state |

## Repository starting points

- The vehicle/object family's primary main script -- inspect its `{init}`/`{frame}` blocks
  as the orchestration pattern for that codebase (treat as an observed pattern, not a
  universal OMSI requirement).
- Any variant main script for the same family (differing focus mode, extra modules, or
  regional differences) -- compare to see how a family adds subsystems without
  restructuring the base pattern.
- A separate AI-focus main script, if one exists for the family; see `omsi/ai-timetable`.
- Representative subsystem files (door, engine, collision, or similar) with `'Needs:'`
  dependency comments and `_Init`/`_Frame` macro pairs.
- Any reusable script library's documentation -- for its documented integration order
  requirement.
- The OMSIWiki "Scripting System" reference article -- authoritative statement of the
  "Distributing the Script into Several Files" convention.

## Uncertainty and escalation

- Treat any one vehicle family's main-script structure as a pattern to adapt, not a
  universal OMSI requirement, when working on a different vehicle type.
- If the target vehicle's actual script list order is not available for review, say so
  explicitly and ask for that configuration file rather than assuming an order from the
  main-script call sequence alone.
- If a subsystem's dependency chain is ambiguous (multiple `'Needs:'` entries, unclear which
  subsystem must run first within the same frame), flag the ambiguity rather than guessing
  an order.
