---
name: omsi-events-and-variables
description: Use when declaring or using OMSI local variables, reading system or predefined local variables, wiring keyboard/mouse triggers, or handling OMSI system triggers such as collision, malfunction, or scheduled-display callbacks
---

# OMSI Events and Variables

## Overview

OMSI scripts communicate with the simulation core through three variable classes (system
variables, predefined local variables, and user-declared local variables) and through
triggers (keyboard/mouse-driven, and system-driven). This skill covers declaring and using
those correctly, and handling the documented system trigger callbacks.

**REQUIRED BACKGROUND:** Load `omsi/foundations` first for the underlying access syntax
(`(L.S.)`, `(L.L.)`, `(S.L.)`, `(L.$.)`, `(S.$.)`) and case-sensitivity rules.

## Scope

- Applies to declaring numeric and string local variables in varlist/stringvarlist files,
  reading system variables and predefined local variables, defining `{trigger:name}` blocks
  for keyboard/mouse events, and handling OMSI system triggers.
- Does not cover multi-file script organization (`omsi/script-architecture`) or the
  configuration-file mechanics of registering a varlist file (`omsi/configuration-and-reuse`).

## Evidence to inspect before making a change

1. The authoritative variable reference for the exact spelling, unit, and read/write
   direction of any system variable or predefined local variable before using it -- these
   names are case sensitive and some are documented as read-only from script ("no write
   access") or bidirectional (OMSI writes a request, the script writes an acknowledgment).
2. The authoritative system trigger reference for the exact trigger name and its data
   contract before adding a `{trigger:name}` block for a system-driven event -- system
   triggers are not named or invoked by the developer; they fire when OMSI decides to.
3. Every existing varlist/stringvarlist file in the target vehicle/object's family, to avoid
   re-declaring a name already declared elsewhere (which is usually a defect, not a valid
   override) and to match the family's existing naming convention.
4. Existing `{trigger:name}` blocks for the same or a related key/mouse event in the target
   subsystem file, to confirm the press/drag/off naming convention already used before
   adding a new one.

## Operating rules

### Three variable classes

- **System variables** (`(L.S.varname)`) are OMSI-wide, read-only from script in the sense
  that a script cannot declare or redefine them; they simply exist. Examples include time-
  of-day, weather, and per-frame timing values. Consult the authoritative reference for the
  exact name and unit before use.
- **Predefined local variables** are vehicle/object-scoped variables that OMSI already
  associates with an internal engine state, but in OMSI 2 some of these still require the
  script to declare them in a varlist/stringvarlist file before they become usable ("on-
  demand predefined variables" per the authoritative reference). Do not assume a predefined
  name is automatically available without declaration; check the reference's "on demand"
  column (or equivalent notation) for the specific variable.
- **User-declared local variables** are freely named by the developer in a varlist/
  stringvarlist file. Prefer names that do not collide with any predefined or system name
  documented in the authoritative reference; a collision with an on-demand predefined name
  silently binds the user variable to internal engine behavior instead of behaving as a
  plain user variable.
- Some predefined local variables are documented as write-restricted ("no write access")
  from script; writing to one of these has no defined effect and should not be relied upon.
  Some are documented as bidirectional for AI vehicles specifically: OMSI writes an intent
  and the script both reads that intent and writes back its own status (see
  `omsi/ai-timetable` for the scheduled-stop handshake as the primary example).

### Trigger classes

- **Keyboard triggers**: for a key combination named `key_combo`, OMSI calls
  `{trigger:key_combo}` when the key is pressed and `{trigger:key_combo_off}` when it is
  released. Both blocks are optional independently, but a press-only handler without an
  `_off` handler will not react to key release.
- **Mouse triggers**: for a mesh with mouse-event designation `mouse_ev`, OMSI calls
  `{trigger:mouse_ev}` on press, `{trigger:mouse_ev_drag}` while held, and
  `{trigger:mouse_ev_off}` on release. A drag handler without a press handler is unusual;
  confirm the intended interaction model before omitting the press case.
- **System triggers** are called directly by OMSI, not by a user input, at specific
  simulation events (for example a collision, or when OMSI requests a display change on an
  AI-driven vehicle). The exact set of system trigger names and their data contracts is
  fixed by OMSI and documented in the authoritative reference; do not invent a system
  trigger name that is not documented there, and do not assume an undocumented trigger name
  will ever be called.

### Delegation to subsystem macros

- When a subsystem already owns a piece of state (for example a door subsystem owning
  door-open state), a new trigger for that same state should delegate to the subsystem's
  existing macro rather than reimplement the state transition inline. This preserves any
  prerequisite check the subsystem's macro already performs (see `omsi/vehicle-systems`).
- A thin trigger wrapper that only records a UI-facing flag (for example marking a physical
  button/switch position for a cockpit indicator) separately from delegating to the
  behavioral macro is an acceptable and common split -- but confirm both halves are present
  when adding a new trigger for an existing behavior, since forgetting the delegation half
  is a common defect.

## Workflow

1. Determine which variable class the new/changed identifier belongs to: system (read-
   only, no declaration), predefined local (may need declaration; check "on demand" status
   in the reference), or plain user-declared local (needs declaration, must not collide
   with any documented name).
2. If declaring a new local variable, add it to the correct list file (numeric vs. string)
   using the naming convention already established in the target family, and confirm no
   existing declaration of the same name exists elsewhere in that family.
3. If adding a keyboard/mouse trigger, confirm whether a press/drag/off counterpart already
   exists or is needed, and match the existing naming pattern for that key/mouse event.
4. If handling a system trigger, confirm the exact name and stack/variable contract against
   the authoritative reference before writing the block; do not guess the trigger's data
   contract from its name alone.
5. If the new logic duplicates state a subsystem already owns, delegate to that subsystem's
   macro instead of writing the state directly.

## Validation

Static checks (possible without running OMSI):

- Every new variable name is checked against the authoritative system/predefined variable
  reference for a collision before being treated as a safe user-declared name.
- Every trigger name used in a `{trigger:name}` block is either a documented keyboard/mouse
  pattern (`name`, `name_drag`, `name_off`) or a name that appears in the authoritative
  system trigger reference; an undocumented, unexplained trigger name is flagged rather than
  assumed to work.
- Read/write direction is checked against the reference's documented restrictions
  (read-only, on-demand, bidirectional) before adding a write to a predefined name.
- Declaration presence: every referenced local variable/string variable resolves to exactly
  one declaration in a varlist/stringvarlist file for the target family, or to a documented
  predefined name.

Runtime checks that require OMSI (not verifiable through static inspection alone):

- Whether a mouse-event trigger actually fires, since that depends on a `[mouseevent]`
  designation on a mesh in the target 3D model, which is outside script files entirely.
- Whether a system trigger fires at the expected simulation moment for the target vehicle's
  actual configuration and hof-file data.
- Whether an on-demand predefined variable behaves as documented once declared, for a
  specific OMSI version.

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| New variable silently behaves like an engine-controlled value | Chosen name collides with a documented on-demand predefined variable | Check the new name against the authoritative predefined-variable reference before declaring |
| Trigger never fires | Trigger name does not match the documented keyboard/mouse pattern, or (for mouse) no mesh declares the matching `[mouseevent]` designation | Compare trigger name against the reference pattern; confirm the model declares the mouse event if applicable |
| Key release has no effect | `_off` counterpart block missing for a press-only trigger | Search for both `{trigger:name}` and `{trigger:name_off}` |
| Write to a predefined variable has no visible effect | Variable is documented as read-only/no-write-access from script | Check the reference's write-access column for that variable |
| AI vehicle never completes a scheduled handshake | Bidirectional predefined variable read but never acknowledged back by the script | Confirm the script writes the expected acknowledgment value, not just reads the request |
| Duplicate/conflicting state | New trigger writes state directly instead of delegating to the subsystem macro that already owns it | Search for other writers of the same variable before adding a new writer |

## Repository starting points

- The authoritative OMSI system-variable and predefined-local-variable reference is the
  primary source for exact names, units, and read/write direction; consult it before
  declaring or using any variable this skill did not already confirm.
- The authoritative OMSI system-trigger reference is the primary source for system trigger
  names and their data contracts; consult it before writing any `{trigger:name}` block that
  is not a plain keyboard/mouse pattern.
- Existing subsystem files in the target vehicle/object family are the best source for the
  family's own local-variable naming convention and trigger-delegation pattern; read a
  working example in the same family before introducing a new one.

## Uncertainty and escalation

- If a variable name is not found in the authoritative reference and not declared anywhere
  in the target family's varlist/stringvarlist files, say so explicitly rather than assuming
  it is safe to use as a new user-declared name.
- If a system trigger's exact data contract (which variables it reads/expects the script to
  set) is not fully specified by the authoritative reference, flag the gap rather than
  inferring the contract from the trigger's name.
- If a mouse trigger's corresponding `[mouseevent]` mesh designation cannot be confirmed
  because the target 3D model/configuration is not available, say so and request the model
  configuration rather than assuming the trigger will fire.
