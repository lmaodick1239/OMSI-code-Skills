# OMSI Agent Skills

This directory contains eight portable, evidence-grounded AI skills for working with the
OMSI 2 scripting system. Each skill follows the
[SKILL.md](https://agentskills.io/specification) convention: a single Markdown file with
YAML frontmatter (`name`, `description`) that a compatible AI agent can load on demand.

These skills are written to be portable: they describe patterns, file naming conventions,
and language rules by name and content rather than by linking to specific files in any one
repository. When you apply a skill, locate the equivalent files in the OMSI project you are
actually working in (main script, subsystem `.osc` files, `*_varlist.txt`,
`*_constfile.txt`, configuration file) using the naming conventions and search guidance each
skill provides, rather than expecting the exact paths named in these skills to exist.

## Evidence model

Every skill in this package makes OMSI-specific claims only from these kinds of sources, in
this priority order:

1. Authoritative OMSI language and integration references (the OMSIWiki "Scripting System",
   "System and predefined local variables", "System Macros", and "System Trigger" articles,
   or an equivalent authoritative reference available to the agent).
2. A modern restatement of the same scripting rules, when available, cited only to
   corroborate the authoritative reference, never to introduce a new unverified claim.
3. A documented reusable script library's public contract (for example a library's own
   README describing required variable/script list registrations and call order), when the
   target project includes such a library.
4. Representative scripts and data files found in the target OMSI project, used as examples
   of a pattern -- never asserted as an OMSI-wide rule on their own.

Anything drawn only from example scripts is labeled a **repository pattern** or **example
pattern**, not an OMSI-wide fact. Files whose names indicate copies, backups, or variants
(for example a `.bak` suffix, a trailing timestamp, or parallel implementations like
`_ok`/`_ok2`/`_ok3`) are never treated as canonical without comparison against the file
actually registered in the target vehicle/object configuration.

## Skill index

| Skill | Use for |
|---|---|
| [`omsi/foundations`](omsi/foundations/SKILL.md) | RPN expressions, stacks/registers, case sensitivity, comments, `{if}`/`{else}`/`{endif}`, macro/`{end}` rules -- load first for any script edit. |
| [`omsi/script-architecture`](omsi/script-architecture/SKILL.md) | Multi-file vehicle/object script layout, `{init}`/`{frame}`/`{frame_ai}` orchestration, subsystem macro naming, script registration order. |
| [`omsi/vehicle-systems`](omsi/vehicle-systems/SKILL.md) | Doors, electrical supply, engine, drivetrain, brakes, cockpit state machines and their interlocks. |
| [`omsi/events-and-variables`](omsi/events-and-variables/SKILL.md) | Declaring/using numeric and string local variables, system/predefined variables, key/mouse triggers, OMSI system triggers. |
| [`omsi/ai-timetable`](omsi/ai-timetable/SKILL.md) | Behavior for AI-driven or off-focus vehicles, `{frame}` vs `{frame_ai}` placement, scheduled bus stop/target triggers, reusable AI-state/time/timetable/delay patterns. |
| [`omsi/displays-and-assets`](omsi/displays-and-assets/SKILL.md) | IBIS, matrix/rollband, DDU, script textures, dynamic text/fonts, reusable draw-macro patterns. |
| [`omsi/configuration-and-reuse`](omsi/configuration-and-reuse/SKILL.md) | `.bus`/`.sco`/`.cfg` `[script]`/`[varnamelist]`/`[stringvarnamelist]`/`[constfile]` registration, constant/curve files, integrating reusable script libraries, filenames and licensing. |
| [`omsi/debugging-performance`](omsi/debugging-performance/SKILL.md) | Fault isolation using `Debug_#`, `$msg`, `%stackdump%`, static validation checklist, runtime test matrix, low-risk performance discipline. |

## How an agent should use this package

1. Pick the narrowest skill for the task at hand (see the table above).
2. For any edit touching OMSI script syntax, load `omsi/foundations` first, then the
   specialized skill.
3. Cross-cutting tasks typically combine two skills: adding a script subsystem uses
   `script-architecture` + `configuration-and-reuse`; vehicle state behavior uses
   `vehicle-systems` + `events-and-variables`; AI-facing displays use `ai-timetable` +
   `displays-and-assets`.
4. Before editing, each skill asks the agent to map: the target entry point/trigger/macro,
   its input variable/constant/system-macro-stack contract, its output variables and
   downstream consumers, and the list/configuration registrations it needs.
5. Validation in every skill is split between static checks that are possible by reading the
   project's own files, and runtime checks that require running OMSI itself with a complete
   vehicle/object package and target `.bus`/`.sco` configuration files.

## Known limits

No skill in this package should claim proof of a runtime behavior that cannot be exercised
by static reading alone. In particular:

- These skills do not assume an OMSI executable or automated test harness is available.
- These skills do not assume every script family has a paired target `.bus`/`.sco`
  configuration in the current project; some libraries or subsystem files may be supplied
  without the model/`.cfg` registration that would prove end-to-end integration.
- Where a skill cannot verify a claim from the evidence actually available, it says so
  explicitly and asks for the missing file (typically the target `.bus`, `.sco`, or
  `model*.cfg`) rather than asserting behavior.


## Sources
https://github.com/Road-hog123/OMSI-RHLib
http://wiki.omnibussimulator.de/omsiwikineu.de/index.php?title=Scriptsystem
http://wiki.omnibussimulator.de/omsiwikineu.de/index.php?title=Fahrzeugleistung_anpassen
http://wiki.omnibussimulator.de/omsiwikineu.de/index.php?title=System-Trigger
http://wiki.omnibussimulator.de/omsiwikineu.de/index.php?title=System-Makros
http://wiki.omnibussimulator.de/omsiwikineu.de/index.php?title=System-_und_vordefinierte_lokalen_Variablen
https://reboot.omsi-webdisk.de/wiki/entry/120-script-system/
https://fellowsfilm.com/threads/omsi-2-osc-programming-language-for-custom-sound-script.17034/
https://cdlbt.co/tutorials/freetex
