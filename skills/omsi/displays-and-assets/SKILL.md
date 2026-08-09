---
name: omsi-displays-and-assets
description: Use when working on IBIS, matrix/rollband, DDU, script textures, freetex texture substitution, text textures, dynamic text/fonts, or other in-cab or exterior display scripts in an OMSI vehicle or scenery object, before calling script texture system macros, wiring a freetex/texttexture entry, or changing display refresh logic
---

# OMSI Displays and Assets

## Overview

OMSI vehicle and scenery-object displays (IBIS controllers, matrix/rollband
destination signs, DDU driver displays, in-cab instruments) are built from three
distinct texture mechanisms, and a single display panel often combines more than one:

1. **Script textures** -- a dynamically rendered texture that the script draws onto
   directly, pixel by pixel or via text-drawing macros, through a lock/draw/unlock
   macro sequence.
2. **Freetex substitution** -- a whole-image swap. The object's configuration file
   declares a texture as replaceable and binds it to a "friendly name"; the script
   sets a declared string variable to a relative image path, and OMSI substitutes the
   entire texture file at that path in place of the original. No per-pixel drawing is
   involved.
3. **Text textures** -- a texture region the configuration file declares with a font,
   size, and color; the script writes display text into an associated string
   variable and OMSI renders it directly, without any script-texture lock/draw/unlock
   macro calls.

These three mechanisms are easy to conflate because the same display family
(destination sign, driver console, passenger information panel) frequently uses more
than one side by side -- for example a route-sign implementation that draws a live
countdown with script-texture macros while swapping a static background image with
freetex, or a driver display that mixes a `[texttexture]` numeric readout with a
freetex-substituted warning icon. This skill covers the lock/draw/unlock discipline
and stack contracts for script textures, the configuration/variable wiring for
freetex substitution, and the configuration wiring for text textures, plus the
string-vs-float stack discipline all three depend on.

**REQUIRED BACKGROUND:** Load `omsi/foundations` first (RPN, stacks, string vs float
access forms) and `omsi/script-architecture` for how a display subsystem's init/frame
macros are wired into a main script.

## Scope

- Applies to script-texture-driven subsystems: IBIS controllers, matrix/rollband
  destination displays, DDU (driver display unit) panels, and any macro sequence that
  calls script-texture system macros to draw text, pixels, or rectangles.
- Applies to freetex texture substitution: the configuration-file `[matl]`/
  `[matl_freetex]` declarations that bind a replaceable texture to a friendly name,
  and the script-side string variable that holds the substitute image's relative path.
- Applies to text textures: the configuration-file `[texttexture]`/`[texttexture_enh]`
  declarations and the script-side string variable that supplies the rendered text.
- Does not cover the vehicle physics/state subsystems that feed display inputs
  (`omsi/vehicle-systems`) or AI-driven display updates specifically
  (`omsi/ai-timetable`, which this skill's triggers often interact with).
- Does not cover the general configuration-file registration mechanics for scripts,
  variable lists, and constants (`omsi/configuration-and-reuse`); this skill covers
  only the display-texture-specific configuration entries (`[matl]`, `[matl_freetex]`,
  `[texttexture]`, `[texttexture_enh]`, `[scripttexture]`) and how each pairs with a
  script-side variable.

## Evidence to inspect before making a change

1. The target display subsystem's `.osc` file in full, including its `_varlist.txt`/
   `_stringvarlist.txt`/`_constfile.txt`.
2. The authoritative system macro reference for script textures (the OMSI wiki's
   "System Macros" page) for the exact input/output stack contract of every macro the
   change touches -- input stack ordering for these macros is significant and easy to
   get wrong without the reference.
3. Any nearby init macro in the same family, to see the established order of texture
   index setup, color/font initialization, and initial draw-then-unlock sequence before
   writing a new one.
4. If the display responds to an AI-facing system trigger (destination/target changes,
   scheduled bus stop updates), also read `omsi/ai-timetable` and `omsi/events-and-
   variables` for that trigger's data-handoff variables.
5. For freetex work: the target object's configuration file for the existing `[matl]`/
   `[matl_freetex]` pair for the texture being substituted, and the registered
   `[stringvarnamelist]` file(s) for the exact friendly-name spelling the
   `[matl_freetex]` entry expects -- the two must match exactly, character for
   character, including case.
6. For text-texture work: the target object's configuration file for the existing
   `[texttexture]`/`[texttexture_enh]` declaration (font, size, color, position) and
   the script for which string variable it is paired with.

## Operating rules

### Script texture lock/draw/unlock sequencing

- A script texture must be initialized once (typically in an `_init` macro), then
  locked before any draw call, then unlocked when drawing for that pass is complete.
  Never issue a draw macro (pixel, rectangle, or text) without an active lock on that
  texture index, and never leave a texture locked across frames -- lock/draw/unlock
  should complete within the same macro invocation.
- Mipmap regeneration (a "filter" step) should happen after unlocking, never between
  lock and unlock. Skipping or reordering this step causes visible aliasing/flicker
  artifacts at a distance from the model; it does not affect readability up close, so
  the defect is easy to miss without a runtime check at multiple camera distances.
- Set the drawing color explicitly before every draw call that depends on a specific
  color; do not assume a previous macro left the desired color selected, since color
  state is a shared, mutable property of the texture context.

### Stack contracts for script texture macros

- Script-texture system macros consume a fixed, ordered sequence of float-stack and
  (for text) string-stack inputs (texture index, then coordinates, then
  color/font/formatting parameters, then finally the text on the string stack for a
  text-drawing macro). Get the authoritative parameter order and count from the system
  macro reference before writing or modifying a call -- do not infer it from a single
  example elsewhere in the codebase, since different macros in the same family (pixel
  vs. rectangle vs. text) take different argument counts.
- Font and text-length lookups are typically separate macros from the text-drawing
  macro itself (look up a font index by name first, measure text length if
  right/center alignment is needed, then draw). Confirm both steps are present when
  adding right- or center-aligned text; a missing length lookup is a common defect that
  causes text drawn flush-left when centered/right alignment was intended.

### Freetex texture substitution

- Freetex is a whole-image replacement mechanism configured entirely through the
  object's configuration file, activated by writing a path into a script-declared
  string variable -- it is not a script-texture drawing operation and does not involve
  lock/draw/unlock macros.
- The configuration file declares the substitutable texture with `[matl]` (naming the
  original texture and its zero-based index within the model, needed only when the
  same texture file is reused across multiple surfaces in that model) followed
  immediately by `[matl_freetex]` (repeating the original texture name, then giving it
  a "friendly name" used to correlate it with a script string variable).
- The friendly name declared in `[matl_freetex]` must have a matching entry, spelled
  identically including case, in a `[stringvarnamelist]`-registered string variable
  list file. Without that matching declaration, OMSI has no way to associate the
  friendly name with a variable the script can write to.
- The script substitutes the texture by writing a relative path (relative to the
  object's texture folder, using backslash path separators) into that string variable.
  An empty or unset value typically leaves the original texture in place; do not assume
  a specific "no substitution" sentinel value without checking the object's own
  documented default for that entry.
- Because freetex is a full-image swap, the substitute image must match the expected
  dimensions and format conventions of the original texture (or the model's UV mapping
  for that surface) to display correctly -- a mismatched aspect ratio or resolution
  will visibly distort rather than error out.
- Freetex substitution is commonly combined with script-texture drawing in the same
  display: a script may swap a full background/frame image via freetex while drawing
  dynamic text or icons on top with script-texture macros. When both are present on
  the same visual surface, confirm draw order and layering intent explicitly rather
  than assuming one mechanism's output does not interfere with the other.
- Group related freetex-substitutable textures under subfolders of the object's
  texture directory for organization; when doing so, reference the subfolder in the
  path written to the string variable (backslash-separated), not in the
  `[matl_freetex]` declaration itself.

### Text textures

- A `[texttexture]`/`[texttexture_enh]` configuration entry declares a texture region
  with a font, size, color, and position; the script supplies the text to render by
  writing to the paired string variable. Unlike a script texture, no lock/draw/unlock
  macro sequence is involved -- OMSI renders the string directly whenever it changes.
- Confirm the exact string variable name paired with a `[texttexture]`/
  `[texttexture_enh]` entry from the configuration file before writing to it; a
  mismatched or misspelled variable name silently produces no visible text rather than
  an error.
- Because rendering is driven purely by the string variable's content, prefer writing
  the finished, fully formatted string in one operation per update rather than partial
  writes across multiple frames, to avoid a visibly incomplete render on an
  intermediate frame.

### Float vs. string stack discipline in display code

- Display code mixes float-stack values (coordinates, colors, indices) with
  string-stack values (destination text, line numbers, formatted output, freetex/text-
  texture path and content strings) more heavily than most other subsystems. Before
  editing a display macro, explicitly separate which values are float-stack and which
  are string-stack in your trace, since a mismatched access form (`(L.L. ...)` for a
  string, or vice versa) silently produces garbage rather than an error (see
  `omsi/foundations`).
- String formatting operations for display code (padding, truncation, numeric-to-string
  conversion) leave the numeric operand used for the operation on the float stack even
  when the primary result is a string; do not assume the float stack is unaffected by a
  string-length/padding operation.

### Update gating and dynamic asset paths

- Only redraw a display when its underlying content actually changed (a destination
  index, a delay value, a scheduled target) rather than redrawing every frame
  unconditionally; an unconditional per-frame full-texture redraw is a common source of
  needless per-frame cost in display scripts (see `omsi/debugging-performance` for the
  broader performance discipline). This applies to script-texture redraws specifically;
  freetex and text-texture updates are driven by variable writes, but the same
  "only write when the value actually changed" discipline avoids needless string
  operations every frame.
- Any dynamic asset path (a font name, a loaded sub-texture, a freetex substitute
  image) used by a display macro or variable is resolved relative to the object's own
  content, not to a shared/global location. Validate a new dynamic asset reference
  against the actual target content package (fonts, texture files) rather than
  assuming a path used elsewhere is present in every vehicle/object package -- if the
  target content is not available to inspect, say so explicitly rather than assuming
  the asset exists.

## Workflow

1. Identify which display family is being changed and which texture mechanism(s) it
   uses (script texture, freetex, text texture, or a combination), then read its full
   `.osc` file plus variable lists and the relevant configuration-file entries.
2. For script-texture changes: confirm the lock/draw/[filter]/unlock sequence for the
   macro being changed matches the established pattern in that same file; do not
   introduce a draw call outside a lock/unlock pair. For any new or changed macro call
   to a script-texture system macro, look up its exact stack contract in the
   authoritative system macro reference before writing the call.
3. For freetex changes: confirm the `[matl_freetex]` friendly name and its paired
   string variable declaration match exactly, and that the path being written is
   relative to the object's texture folder using backslash separators.
4. For text-texture changes: confirm the paired string variable name against the
   configuration file's `[texttexture]`/`[texttexture_enh]` entry before writing to it.
5. If the display reacts to a trigger fed by AI/timetable data, confirm the receiving
   variable(s) match `omsi/ai-timetable`'s documented data-handoff contract for that
   trigger.
6. Add or confirm an update-gate (a "did this change" check) before an expensive
   redraw or string write, unless the display is known to be cheap to update
   unconditionally.
7. Re-check float-vs-string access forms across the whole changed block.

## Validation

Static checks (possible from this repository alone):

- Every script-texture draw macro call site is preceded by a lock call and followed
  (within the same macro) by an unlock call for the same texture index.
- Mipmap/filter regeneration, if present, occurs after unlock, never between lock and
  unlock.
- Every script-texture macro call's argument count and stack order matches the
  authoritative system macro reference exactly.
- Every `[matl_freetex]` friendly name has a matching entry in a registered
  `[stringvarnamelist]` file, spelled identically including case.
- Every `[texttexture]`/`[texttexture_enh]` entry's paired string variable name matches
  what the script actually writes to.
- Float vs. string access forms are correct for every variable touched in the changed
  block.
- A length/measurement lookup precedes any right- or center-aligned script-texture text
  draw that depends on measured width.

Runtime checks that require OMSI (not verifiable from this repository):

- Whether mipmap/filter timing actually produces a visible aliasing artifact at
  distance -- this requires observing the rendered texture in OMSI at multiple camera
  distances.
- Whether a dynamic asset path (font, sub-texture, freetex substitute image) resolves
  correctly for a specific target vehicle/object package not present in this
  repository.
- Whether a freetex substitute image's dimensions/aspect ratio actually match the
  original surface's UV mapping without visible distortion.
- Actual frame-time cost of a display's redraw logic under a real vehicle load.

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| Display flickers or looks corrupted while updating | Script-texture draw call issued without an active lock, or texture left locked across frames | Confirm every draw call is inside a matching lock/unlock pair scoped to one macro invocation |
| Distant display looks oversharpened/aliased | Mipmap/filter step missing or issued between lock and unlock instead of after unlock | Confirm filter call placement relative to lock/unlock |
| Text silently wrong or garbled | Float/string access form mismatch, or wrong stack argument order for a text-drawing macro | Re-check access forms and argument order against the system macro reference |
| Right- or center-aligned text renders flush-left | Missing length/measurement lookup before the aligned draw call | Confirm a text-length macro call precedes the aligned draw and its result feeds the position calculation |
| Freetex substitution never takes effect | Friendly name in `[matl_freetex]` does not exactly match the declared string variable name, or the variable is never registered under `[stringvarnamelist]` | Compare the configuration file's friendly name against the string variable list entry character for character |
| Freetex image displays stretched or distorted | Substitute image dimensions/aspect ratio do not match the original texture's UV mapping | Compare substitute image dimensions against the original texture being replaced |
| Text texture never shows or shows the wrong content | Script writes to a string variable name that does not match the `[texttexture]`/`[texttexture_enh]` entry's paired variable | Compare the configuration entry's variable name against every write site in the script |
| Display never updates after a destination/target change | Update-gate condition never becomes true, or the AI/timetable trigger's data-handoff variable is not being read | Cross-check against `omsi/ai-timetable`'s documented trigger contract for that data source |
| New dynamic asset (font/texture) fails to load for a specific vehicle | Asset path assumed to exist based on another package's layout | Confirm the asset is present in the specific target content package before relying on the path |

## Repository starting points

- Search for `.osc` files whose names match display-related subsystem families (IBIS,
  matrix, rollband, DDU, driver information/GPS-style controllers) and inspect their
  init/frame macros for the lock/draw/unlock pattern, plus their configuration files
  for `[matl_freetex]` and `[texttexture]`/`[texttexture_enh]` entries.
- The OMSI wiki's "System Macros" reference page is the authoritative source for script
  texture macro names, stack contracts, and font/text-length lookup macros -- consult
  it before writing any new script-texture call.
- A freetex authoring tutorial, if available, is a useful reference for the exact
  `[matl]`/`[matl_freetex]`/`[stringvarnamelist]` sequence and common pitfalls (index
  numbering, friendly-name-to-variable correlation, subfolder path conventions).
- A library's reusable "draw" module, if present in the repository, is a useful
  reference for a clean, minimal wrapping of the lock/unlock and text-alignment pattern;
  confirm its own declared prerequisites (variable lists, init call) before reusing its
  macros.

## Uncertainty and escalation

- If a display subsystem is supplied without the model/texture registration that would
  prove its script-texture index, freetex entry, or text-texture entry is actually
  wired to a visible surface, say so explicitly rather than asserting the display works
  end-to-end.
- If the exact stack contract for an unfamiliar script-texture macro is not covered by
  the authoritative system macro reference available in this repository, flag the gap
  rather than guessing argument order or count.
- If a dynamic asset (font, sub-texture, freetex substitute image) referenced by a
  display macro or variable cannot be found in the target content package, ask for
  that package rather than assuming the asset is shared across all vehicles/objects.
- If a `[matl_freetex]` friendly name has no discoverable matching string variable
  declaration anywhere in the registered variable lists, say so explicitly rather than
  assuming the wiring is complete.
