---
name: omsi-foundations
description: Use when reading or writing any OMSI .osc script expression, before touching stack-based RPN code, {if}/{else}/{endif} blocks, macro calls, comments, or variable/constant access syntax
---

# OMSI Scripting Foundations

## Overview

OMSI's scripting language (`*.osc` files) is a stack-based, Reverse Polish Notation (RPN)
language with separate float and string stacks, eight numeric registers, case-sensitive
identifiers, and a call-before-definition rule for macros. This skill covers the language
mechanics every other OMSI skill in this package depends on. Load it first whenever an
edit touches script syntax, then load the specialized skill for the task.
For exact signatures, operators, and variable dictionaries, load `omsi/system-reference`.

Source authority: the OMSIWiki "Scripting System" reference article is the primary source
for the rules below. A modern restatement of the same rules (checked for agreement) is used
only where it adds detail, for example on filename recommendations for new script assets.
Where sources agree, this skill states the rule once rather than duplicating citations.

## Scope

- Applies to every `*.osc` file, every constant/curve definition file, and expressions
  embedded in variable-list/string-variable-list declarations, in any OMSI vehicle or
  scenery object script package.
- Does not cover multi-file orchestration (`omsi/script-architecture`), configuration file
  registration (`omsi/configuration-and-reuse`), or subsystem-specific behavior
  (`omsi/vehicle-systems`, `omsi/ai-timetable`, `omsi/displays-and-assets`).

## Evidence to inspect before making a change

1. The exact target file and the 10-20 lines around the change, read in full (do not infer
   stack state from a fragment).
2. The macro or trigger's call site(s) via a repository search, to confirm call-before-
   definition ordering will still hold after the edit.
3. Any variable-list/string-variable-list/constant file that declares identifiers the
   expression reads or writes.
4. A working, unmodified example of the same construct elsewhere in the same codebase (for
   example an existing trigger block, or a short, well-commented macro body) to confirm
   local convention before introducing a new pattern.

## Operating rules

### Reverse Polish Notation and stacks

- OMSI uses RPN: operators follow operands (`1 2 +` computes `1 + 2`; `(1+2)*(4+5)` is
  `1 2 + 4 5 + *`).
- There are two independent stacks -- a float stack and a string stack -- each with 8
  addressable slots (0-7). A push places the new value at slot 0 and shifts existing values
  up; a pop/read removes or inspects from slot 0 outward.
- There are 8 float registers (`l0`-`l7` load, `s0`-`s7` store); registers persist across
  stack operations within the same script pass and are the only way to stash a float value
  without a named local variable. There is no register for strings; use `d`/`$d` to
  duplicate top-of-stack instead.
- Comparison/subtraction operators are order-sensitive to push order: `-` computes
  `stack1 - stack0`; `4 2 >` is true because it evaluates `4 > 2` (the earlier-pushed value
  is on the left of the comparison).
- Before editing an RPN expression, trace the stack depth and slot contents line by line
  (mentally or in a scratch note) for every value pushed, popped, and duplicated. An
  off-by-one in ordering silently reads the wrong stack slot; OMSI does not raise a runtime
  type or bounds error for this.

### Case sensitivity and comments

- OMSI script text is case sensitive throughout: keywords, macro names, variable names, and
  trigger names must match their declaration/definition exactly.
- A comment line starts with an apostrophe (`'`) at the very first character of the line.
  Indentation before the apostrophe breaks the comment and it will be interpreted as code
  (the wiki states plainly: `'I am a comment line` / `   'I am NOT a comment line!`). Most
  real-world script files consistently place `'` at column 1 for comments; preserve that
  convention in new lines.

### Blocks: entry points, `{end}`, and `{if}`/`{else}`/`{endif}`

- Every block of commands sits between one entry-point keyword (`{init}`, `{frame}`,
  `{frame_ai}`, `{macro:name}`, `{trigger:name}`) and the universal exit keyword `{end}`.
  Every entry point requires exactly one matching `{end}`.
- `{if}` follows the condition expression, not precedes it: push the condition result first
  (0 = false, anything else = true), then write `{if}`. `{else}` is optional; `{endif}`
  always closes the nearest open `{if}`.
- Nested `{if}` is the only substitute for "else if"; each nested `{if}` needs its own
  `{endif}`, so a two-level nested conditional needs two consecutive `{endif}` lines.
  Miscounting these is a common source of silent logic bugs since OMSI does not report a
  block-mismatch error message an agent can rely on for detection -- static line-by-line
  matching is required (see Validation).

### Macros and the call-before-definition rule

- A macro is invoked with `(M.L.name)` and must be called *before* the `{macro:name}` block
  that defines it appears in the loaded script text (across all files, in script list
  order). Calling a macro after or without a matching later definition causes OMSI to
  report the macro could not be found.
- This rule is why a main script places trigger/macro *calls* early and subsystem files
  loaded afterward place the macro *definitions* later -- see `omsi/script-architecture`
  and `omsi/configuration-and-reuse` for the multi-file consequences.

### Data types and variable access

- OMSI has exactly two data types: float (single precision, signed) and string. There are
  no booleans; 0/1 floats are used for false/true.
- Access forms (all case sensitive):
  - `(L.S.varname)` load system variable to float stack.
  - `(L.L.varname)` load local float variable to float stack.
  - `(S.L.varname)` store float stack top into local float variable.
  - `(L.$.varname)` load local string variable to string stack.
  - `(S.$.varname)` store string stack top into local string variable.
- System macros are invoked with `(M.V.macroname)`; they read/write both stacks according to
  their own documented contract (see `omsi/displays-and-assets` and `omsi/ai-timetable` for
  specific macros, and the OMSIWiki "System Macros" reference for authoritative stack
  contracts).
- Sound triggers (`(T.L.name)`, `(T.F.name)`) are a distinct mechanism from keyboard/mouse/
  system triggers (`{trigger:name}`); do not conflate the two when reading or writing
  script.

### Constants and curves

- Constant/curve definition files declare `[const]` (name + single float value) and
  piecewise linear `[newcurve]`/`[pnt]` functions. `[pnt]` x-values must be strictly
  ascending within a curve; OMSI extends the first/last point horizontally outside the
  defined range rather than extrapolating the slope.
- `(C.L.name)` loads a constant; `(F.L.name)` calls a curve, consuming the top float stack
  value as x and replacing it with the interpolated/extended y.
- Only floats can be defined in constant files; there is no string constant form in the
  documented language.

## Workflow

1. Identify the exact block (entry point or macro) being changed and read it in full,
   including its `{end}`/`{endif}` closures.
2. Trace stack effect for every line touched: what is pushed, what is consumed, and what
   remains afterward, for both the float and string stacks independently.
3. Confirm every identifier referenced (`(L.L. ...)`, `(S.L. ...)`, `(L.$. ...)`,
   `(S.$. ...)`, `(C.L. ...)`, `(F.L. ...)`, `(M.L. ...)`) is declared/defined somewhere the
   agent can point to (variable list, string-variable list, constant file, or an earlier
   `{macro:...}` block).
4. Make the minimal, narrow edit. Preserve existing indentation, comment style, and
   line-ending/encoding conventions of the file (see `omsi/configuration-and-reuse` for
   encoding notes).
5. Re-read the edited block end-to-end and re-count `{if}`/`{endif}` pairs and the block's
   entry/`{end}` pair.

## Validation

Static checks (possible from source reading alone):

- Block balance: every `{init}`/`{frame}`/`{frame_ai}`/`{macro:...}`/`{trigger:...}` has
  exactly one `{end}`; every `{if}` has exactly one `{endif}`, with `{else}` optional and
  singular.
- Stack-effect check: manually re-derive the float/string stack contents at each line of a
  changed expression; confirm the final consumer (a `(S.L. ...)`, `(S.$. ...)`, comparison,
  or macro call) reads the slot the agent intended.
- Access-form check: confirm float vs. string access forms match the variable's declared
  type (`(L.L.)`/`(S.L.)` for float locals vs `(L.$.)`/`(S.$.)` for string locals); a
  type-form mismatch against a declared name is a common defect class in hand-edited script.
- Case check: confirm every macro/trigger/variable name matches its declaration or the
  wiki-documented system name exactly, including case.
- Call-order check: confirm every macro call site precedes its `{macro:...}` definition in
  the effective concatenated script order implied by the script list (see
  `omsi/script-architecture`).

Runtime checks that require OMSI (not verifiable from source reading alone):

- Whether the edited expression produces the intended in-sim value at the target frame,
  since static reading cannot substitute for an OMSI executable or automated script runner.
- Whether a macro/trigger name collides with an OMSI-internal on-demand predefined variable
  not covered by the reference material available for review.

## Common failure modes

| Symptom | Likely cause | Detection |
|---|---|---|
| "Macro not found" behavior / silently no-op | Macro called after its definition, or definition missing from script load order | Search for `{macro:name}` and confirm its position relative to `(M.L.name)` call sites and file load order |
| Wrong branch taken in `{if}` | Condition value pushed after `{if}` instead of before, or comparison operand order reversed | Re-derive stack state immediately before `{if}`; check `-`/`<`/`>` operand order |
| Value silently wrong with no error | Off-by-one stack slot access, or float/string access form mismatch | Line-by-line stack trace; check access form against declared variable type |
| Nested conditional executes wrong branch | Missing or extra `{endif}` in a nested `{if}` chain | Count `{if}`/`{endif}` pairs top-down; each nested `{if}` needs its own `{endif}` |
| Nothing happens after edit, no visible error | Comment line broken by leading whitespace (line now parsed as code) or vice versa | Check every `'` comment starts at column 1 |
| Curve gives unexpected flat value at range edges | `[pnt]` list not ascending, or value outside range (expected -- OMSI clamps horizontally) | Confirm curve points ascend in x; confirm whether flat extension is the intended behavior |

## Repository starting points

- The OMSIWiki "Scripting System" reference article -- authoritative language reference
  (RPN, stacks, keywords, operators, variable access, constants/curves, sound triggers,
  system macros, conditionals).
- A modern restatement of the same scripting-system rules, useful for a second phrasing of
  an ambiguous point.
- Any existing, unmodified `.osc` file in the same codebase with trigger blocks and macro
  definitions, as a source of typical stack usage.
- Short, well-commented macros from a reusable script library (if one is present in the
  project), useful as a clean pattern reference for stack-heavy arithmetic and nested
  `{if}` structure.

## Uncertainty and escalation

- If a variable or macro name appears nowhere in the project's variable lists,
  string-variable lists, constant files, or script definitions, say so explicitly and ask
  whether it is an OMSI-predefined name (consult `omsi/events-and-variables`) rather than
  assuming it exists.
- If the intended stack effect of an unfamiliar operator is not covered by the available
  reference material, do not guess its behavior; flag the gap and request confirmation
  before relying on it.
- Do not claim a static edit "works" -- only that it is internally consistent by the checks
  above. In-sim confirmation requires OMSI, which is outside the scope of static review.
