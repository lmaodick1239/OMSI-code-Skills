---
name: omsi-configuration-and-reuse
description: Use when registering script/variable/constant files in an OMSI vehicle or scenery object configuration file, defining constants and curves, integrating a reusable script library module, or choosing filenames and encoding for new OMSI script assets
---

# OMSI Configuration and Reuse

## Overview

Every OMSI script file, variable list, and constant file must be explicitly registered in
the owning vehicle or scenery object's configuration file before OMSI will load it.
Registration order interacts directly with the call-before-definition rule for macros, and
reusable script libraries add their own ordering and encoding requirements on top of that.
This skill covers the configuration-file contract and how to integrate a reusable script
module correctly.

**REQUIRED BACKGROUND:** Load `omsi/foundations` first for the call-before-definition rule,
and `omsi/script-architecture` for why file order in the configuration matters.

## Scope

- Applies to the configuration-file commands that register script assets, to constant/curve
  file structure, and to integrating a reusable script library (a self-contained package of
  `.osc`/varlist/constfile modules meant to be dropped into multiple vehicle projects).
- Does not cover the internal behavior of any specific subsystem (`omsi/vehicle-systems`,
  `omsi/ai-timetable`, `omsi/displays-and-assets`) or the RPN/stack mechanics of the script
  bodies themselves (`omsi/foundations`).

## Evidence to inspect before making a change

1. The target object's configuration file in full, focusing on every command block that
   lists script, variable, string-variable, or constant files, in the order they appear.
2. Every file the configuration references, to confirm each one exists at the stated
   relative path and that the path is spelled with the exact case used on disk.
3. The reusable library's own documentation (its README or header comments), if one is being
   integrated, for its exact required file list, call-site requirements, and any variables it
   marks as read-only output.
4. The header comment block of any file being copied or adapted, for encoding, license, and
   attribution notices that must be preserved.

## Operating rules

### Configuration file registration commands

- A vehicle or scenery object's configuration file uses dedicated commands to register the
  files it needs: one for script files, one for numeric variable-list files, one for string
  variable-list files, and one for constant/curve files. Each command's first entry is the
  count of files that follow, then the filenames themselves, given as a path relative to the
  configuration file itself.
- Predefined/system variables are the only exception: they do not require registration and
  can be read or written without appearing in any list.
- File names are otherwise free-form, but the community convention (stated in the supplied
  modern scripting guide) is to avoid special/non-English characters in filenames to prevent
  path issues on non-matching-locale systems -- prefer plain ASCII names for any newly
  created script, varlist, stringvarlist, or constfile.
- When adding a new subsystem file, its varlist/stringvarlist/constfile must be added to the
  *same* configuration file, in the same edit, as the `.osc` file itself -- a script that
  reads or writes a variable not declared anywhere (and not a predefined variable) will not
  behave as intended.

### Order interacts with the call-before-definition rule

- Because a macro must be called before it is defined (see `omsi/foundations`), the order of
  files under the script-registration command determines the effective concatenated script
  the language engine sees. A file whose macros are called by an earlier file must itself
  appear later in the list.
- This is why a main script that calls every subsystem's `_Init`/`_Frame` macros must be
  listed before every subsystem file, and why subsystem files that call each other's macros
  must be ordered so each call still precedes its definition (see `omsi/script-architecture`
  for the full reasoning).
- Reordering an existing registration list is a structural change: re-check every
  cross-file macro call after any reorder, not just the files directly touched.

### Constants and curves

- Constant/curve files use their own small command set: one command defines a single named
  float constant with its value; another begins a new piecewise-linear curve definition;
  a third adds one x-y point to the most recently started curve. Only floats can be defined
  this way -- there is no string-constant form.
- Curve points must be added in strictly ascending x-order. OMSI extends the curve flat
  (using the nearest endpoint's y-value) for any x outside the defined range rather than
  extrapolating the slope; keep this in mind when adding or removing endpoint points.
- Prefer expressing a tunable numeric value or a piecewise-linear response as a constant or
  curve rather than hardcoding it in a macro body, so it can be tuned without touching script
  logic (see `omsi/vehicle-systems` for the drivetrain/engine tuning example).

### Integrating a reusable script library module

- Treat a reusable library as a set of independent modules, each with its own required file
  registrations and call sites; only integrate the modules a feature actually needs, since
  each module's frame-time calls carry a performance cost even when unused output is
  discarded.
- A library module typically documents, per module: which varlist/stringvarlist/script file
  must be registered, which of its init/frame macros must be called and from which entry
  point(s) (`{init}` only, `{frame}` only, or both `{frame}` and `{frame_ai}`), and whether
  later modules in the same library depend on an earlier module's output variables.
- A library's own script files are conventionally registered *after* the calling vehicle's
  own subsystem files under the script-registration command, while the calls into the
  library's init/frame macros happen *early* -- mirroring the same call-before-definition
  ordering used for in-house subsystems (see `omsi/script-architecture`). This ensures the
  library's variables are populated before any other script tries to read them in the same
  frame.
- Variables a library documents as its own output (i.e., set by the library, not meant to be
  written by the integrating script) should be treated as read-only from outside the library
  once integrated; writing to them from vehicle-specific script defeats their purpose and can
  desynchronize the library's internal state tracking (for example, a library-maintained
  "value changed this frame" flag will become wrong if the underlying value is written from
  outside the library).
- If the library is distributed under a permissive open-source license requiring attribution
  and change notices (for example an Apache-2.0-style license), preserve the license header
  and copyright notice at the top of every library file when copying or adapting it, and
  record what was changed if the file is modified, per that license's terms.

### Encoding and file hygiene

- Preserve the existing text encoding and line-ending convention of a file being edited;
  do not silently re-save a file in a different encoding as a side effect of an edit, since
  OMSI script files can be affected by encoding-dependent characters in comments or string
  literals.
- Do not treat a `.bak`, timestamped-backup (for example a name ending in a numeric
  timestamp before its original extension), or "Copy"/"Kopie"-suffixed file as the canonical
  version of a script or constfile; always confirm which file the target configuration
  actually registers before editing or reusing content from a sibling file.

## Workflow

1. Identify every new file the change introduces (script, varlist, stringvarlist,
   constfile) and the configuration file that must register them.
2. Add each new file to the correct registration command, incrementing the file count and
   inserting the filename in a position consistent with the call-before-definition ordering
   established in `omsi/script-architecture`.
3. If integrating a reusable library module, cross-check its documented required
   registrations and call sites one by one; do not assume a module is fully wired in until
   every documented requirement is satisfied.
4. If defining new constants/curves, add them to the correct constfile, keeping curve
   points strictly ascending in x.
5. Re-verify no unrelated registration entry was reordered or duplicated by the edit.
6. Confirm new filenames use plain ASCII characters and do not collide with an existing
   registered file under a different case (which would be a distinct file on a
   case-sensitive filesystem but might be confused with an existing one).

## Validation

Static checks (possible without OMSI):

- Every file referenced in a registration command exists at the stated relative path with
  exactly matching case.
- Every new script/varlist/stringvarlist/constfile introduced by the change has a
  corresponding registration entry, and vice versa (no registered file that does not exist,
  no needed file left unregistered).
- Script registration order still satisfies call-before-definition for every macro call
  touched by the change (cross-check with `omsi/script-architecture`).
- Curve `[pnt]`-equivalent entries are strictly ascending in x within each curve.
- Any copied library file retains its original license header and copyright notice.

Runtime checks that require OMSI (not verifiable without it):

- Whether OMSI successfully loads the modified configuration file without an internal
  parsing error not visible from static inspection.
- Whether a reused library module's frame-time behavior matches its documentation once
  actually driven by OMSI's simulation loop.

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| New variable behaves as always-zero or undeclared | Varlist/stringvarlist entry omitted from the configuration file's registration list | Confirm every new variable's declaring file is registered |
| "Macro not found" after adding a new subsystem file | New file inserted at the wrong position relative to its callers in the script registration list | Recheck ordering against every macro call site, not just the new file's own calls |
| Library module produces stale or zero values | A required init/frame call for that module was never added to an entry point, or was added to the wrong entry point ({frame} vs {frame_ai}) | Cross-check every documented call-site requirement for the module against the actual `{init}`/`{frame}`/`{frame_ai}` blocks |
| Library's internal change-tracking flag is wrong | Vehicle script wrote directly to a variable documented as library output | Search for any `(S.L. ...)`/`(S.$. ...)` write to a library output variable outside the library's own files |
| Curve gives a flat, unintended value at one end | New `[pnt]`-equivalent entry inserted out of ascending x-order, or an endpoint accidentally removed | Re-read the curve's full point list in file order and confirm ascending x |
| File loads the wrong content after an edit | A `.bak`/timestamped-backup/"Copy" sibling file was edited instead of the one actually registered | Confirm which exact filename appears in the configuration file's registration list |

## Repository starting points

None of this skill's rules require repository-specific evidence beyond the language and
library-integration references already cited; when working in a specific repository, locate
its configuration-file format documentation, its constant/curve file convention, and any
bundled reusable script library's own README before applying this skill.

## Uncertainty and escalation

- If the target vehicle's or scenery object's actual configuration file is not available,
  say so explicitly and ask for it rather than inferring registration order from script
  content alone.
- If a reusable library's documentation does not clearly state whether a variable is
  read-only output, treat it as read-only by default and flag the ambiguity rather than
  writing to it.
- Do not assume a filename convention (case, special characters) is safe without confirming
  the target platform; when in doubt, prefer plain ASCII and flag any existing
  non-ASCII filename in the surrounding project as a pre-existing risk rather than a pattern
  to imitate.
