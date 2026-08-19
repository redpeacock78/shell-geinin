---
name: shell-geinin
description: "Use proactively for repository investigation, file/code search, Git/GitHub inspection, logs, JSON/CSV/text transformation, inventories, diagnostics, batch operations, and ad-hoc data reduction. Apply shell-gei reasoning across POSIX/Linux/macOS/BSD/BusyBox, PowerShell, Windows cmd.exe, WSL/MSYS2/Cygwin, and mixed environments. Prefer purpose-built CLIs and native pipeline operators over throwaway Python/Node/Ruby/Perl scripts; minimize generated command text, repeated work, and tool output returned to model context."
license: MIT
compatibility: Cross-platform. Designed for AI Agent in POSIX-style shells, GNU/Linux, macOS/BSD, BusyBox/embedded systems, PowerShell 5.1+/7+, cmd.exe, and Windows Unix-compatibility layers such as WSL, MSYS2, Cygwin, and Git Bash. Optional tools are capability-probed rather than assumed.
---

# Shell Geinin — Cross-Platform Edition

Use the active command environment as a small algebra for investigation and data reduction.

The goal is not code golf and not Unix cosplay. Borrow the reasoning of shell-gei: change the representation of the problem until the shell's existing primitives solve local pieces naturally, compose those pieces through the environment's native pipeline model, and return only the information the model needs.

The central optimization target is:

**few generated tokens + few subprocesses/cmdlets + little repeated I/O + little returned output + clear semantics**

## Prime directive

Before writing disposable glue in a general-purpose language runtime, attempt to compile the task into:

**discover → choose data model → acquire narrowly → normalize → expose keys/structure → filter/project → relate/group/order → reduce → reshape/render**

Not every task needs every stage. Remove stages that a producer can already perform.

## 1. Detect semantics before choosing syntax

Do not choose commands from the host OS name alone. Determine the active shell and the actual commands that resolve in it.

A Windows host may be running PowerShell, cmd.exe, WSL, MSYS2, Cygwin, Git Bash, or an SSH session into Unix. PowerShell itself can run on Windows, Linux, or macOS. A BSD/macOS machine may have GNU tools installed under prefixed names. A Linux container may expose only BusyBox applets.

Use capability discovery when it matters:

### POSIX-style shell

```sh
command -v rg jq git awk sed find xargs 2>/dev/null
command -V sort 2>/dev/null
uname -s 2>/dev/null || :
```

Use `type`/`type -a` when available to detect aliases/functions/shadowing.

### PowerShell

```powershell
$PSVersionTable
Get-Command rg,jq,git,Select-String,Get-ChildItem -ErrorAction SilentlyContinue
Get-Command sort -All
```

Use `Get-Command`, `Get-Help`, and `Get-Member` as discovery tools. Prefer full cmdlet names in generated commands; aliases such as `cat`, `sort`, `%`, and `?` can hide semantic differences.

### cmd.exe

```bat
ver
where rg 2>nul
where jq 2>nul
where git 2>nul
where findstr 2>nul
```

Never infer a custom helper's behavior from its name. Inspect help/source/type first.

## 2. Choose the native pipeline data model

### Text-stream shells

For POSIX sh, bash, zsh, ksh, dash, ash/BusyBox, BSD shells used in sh-like mode, WSL, MSYS2, Cygwin, and Git Bash, think primarily in:

- byte/text streams;
- records and fields;
- sorted sets;
- keyed relations;
- exit status;
- file descriptors and standard streams.

### PowerShell

PowerShell pipelines carry objects between cmdlets. Preserve objects as long as possible.

Think in:

- objects and properties;
- collections;
- typed comparisons;
- provider-side filtering;
- grouping, projection, measurement, and comparison;
- output streams.

Do **not** stringify objects early merely to imitate Unix text pipelines. Do not parse formatted tables when properties are still available.

### cmd.exe

cmd.exe is a constrained text shell. Think in:

- lines of text;
- `%ERRORLEVEL%`;
- `for`/`for /f` iteration and tokenization;
- redirection and pipes;
- native Windows commands.

Keep cmd solutions simple. If a task becomes quote-heavy, stateful, or structurally complex and PowerShell is already available, promoting from cmd to PowerShell is a shell-level escalation, not a failure.

## 3. Shell-gei mindset

### Solve the concrete instance first

Do not build a parser framework, helper package, or reusable program for a one-off investigation unless the task actually needs reuse.

### Change representation before adding control flow

When a problem looks procedural, ask whether a representation change makes it local.

Useful transformations include:

- file → record stream;
- line → tokens/fields/characters;
- objects → selected properties;
- nested input → keyed records;
- rows ↔ columns;
- value → `key<TAB>value`;
- unordered observations → sorted set;
- ragged shape → padded rectangle → transform → unpad;
- repeated structure → generate once → mirror/reuse;
- two time offsets → aligned streams;
- binary → only the header/magic bytes needed for the predicate.

### Prefer dataflow to hand-written orchestration

Ask first:

- Can filtering move into the producer?
- Can grouping replace a loop?
- Can a set difference replace candidate-by-candidate testing?
- Can a join replace nested lookup logic?
- Can alignment replace index arithmetic?
- Can one producer feed several analyses?
- Can exit status answer the predicate without text parsing?

### Use the strongest native option first

An option is often an algorithm.

Examples of the mindset:

- Unix files without a match → `grep -L` when supported;
- first few matches → producer `-m`/limit option;
- names only → `rg -l` or equivalent;
- Git history bound → `git log -n N`;
- filesystem predicates → `find` predicates instead of post-filtering names;
- PowerShell filesystem filtering → `Get-ChildItem -Filter` when suitable, before `Where-Object`;
- cmd recursive bare paths → `dir /s /b` rather than parsing decorated `dir` output;
- cmd files containing a match → `findstr /m`.

### Normalize once at the boundary

Canonicalize separators, timestamps, encodings, case policy, keys, or record layout once. Downstream stages should operate on one boring representation.

### Expose identity before destructive transforms

If ordering or identity matters, tag/index first and remove the tag later.

Typical forms:

- Unix: `nl`, `awk '{print NR, ...}'`, `path<TAB>value`;
- PowerShell: preserve source object properties or add a calculated property;
- cmd: carry `%~fF`, `%~zF`, or an explicit token alongside the value.

### Generate the universe, then compare

For finite missing-value or validation tasks, consider:

**candidate universe → normalized observed set → difference/intersection**

instead of a procedural probe for every candidate.

### Align rather than index

If records have a fixed positional relation, align streams/collections rather than manually managing indices when the environment has a natural primitive.

### Exploit symmetry and reuse

Compute the irreducible part once. Reverse, reflect, transpose, duplicate, or branch instead of recomputing equivalent work.

### Parse only as deeply as the question requires

Do not load a full parser for a shallow predicate. Inspect file type, header bytes, selected JSON properties, archive members, object properties, or metadata first.

### Treat ordering semantics as correctness

Know whether the task needs:

- numeric vs lexical sorting;
- stable order;
- locale/culture behavior;
- case sensitivity;
- original order after grouping;
- adjacent duplicate removal vs global uniqueness.

## 4. Universal tool-selection policy

Use this order unless the task gives a better domain-specific route:

1. **Purpose-built cross-platform CLI already modeling the domain**
2. **Active shell's native structured/filtering primitives**
3. **Portable/common stream utilities**
4. **Optional richer utilities discovered at runtime**
5. **A maintained script/program only when the shell solution stops being clear or robust**

Examples of high-value purpose-built tools when installed:

- repository/history: `git`;
- GitHub: `gh`, especially `gh --json` / `gh api` with structured projection;
- code/text search: `rg`, then native `grep`/`Select-String`/`findstr`;
- JSON: `jq`, or PowerShell `ConvertFrom-Json` when staying object-native;
- HTTP: `curl`, or PowerShell `Invoke-RestMethod` when object conversion is useful;
- SQL-shaped ad-hoc work: `sqlite3` when available and clearer than a long pipeline.

Do not install a new tool just to avoid a short native solution unless installation is part of the task.

## 5. General-purpose runtime avoidance

Do not invoke `python`, `python3`, `node`, `ruby`, `perl`, `deno`, `bun`, `php`, `go run`, `rustc`, `dotnet-script`, `jshell`, or similar general-purpose language runtimes merely to:

- walk a directory tree;
- search/filter/map/reduce lines;
- count, rank, group, sort, or deduplicate records;
- select fields/properties;
- parse/project ordinary JSON/CSV;
- batch commands;
- add simple parallelism;
- perform simple integer/text transformations;
- inspect shallow binary signatures;
- compare sets of names/IDs;
- make one HTTP request already covered by a CLI.

`awk` and `sed` are intentionally allowed as stream-processing DSLs in Unix-style environments. PowerShell is allowed when it is the active shell or the appropriate Windows shell. Project toolchains are allowed when the task itself requires them: tests, compilers, formatters, package managers, generators, build tools, migration tools, or project scripts.

Before escalating to a runtime, ask:

> Can the operation be expressed clearly as a purpose-built command or a short composition of native pipeline operators without fragile quoting or hidden state?

If yes, stay in the shell.

## 6. Context-budget discipline

A command is not efficient if it emits a novel into model context.

### Filter and project before output

Prefer:

```sh
rg -n 'pattern' src | head -50
jq -c '.items[] | {id,status}' data.json | head -50
git log -n 30 --oneline
```

PowerShell equivalent mindset:

```powershell
Get-ChildItem -File -Recurse -Filter '*.log' |
  Select-Object -First 50 FullName, Length
```

cmd mindset:

```bat
dir /s /b *.log | findstr /i error
```

Do not `cat`, `type`, `Get-Content`, `git log`, `gh api`, `dir /s`, or serialize whole objects when a narrower producer query answers the question.

### Narrow first, widen only if necessary

Start with counts, names, summaries, first matches, bounded history, selected fields/properties, or one relevant range. Expand only when the result is insufficient.

### Do not repeat expensive producers

If multiple analyses consume the same expensive source:

- Unix: consider `tee`, `pee`, one `awk`, one `jq`, a temporary cache, or a single producer with several projections;
- PowerShell: retain objects in the pipeline, use `Tee-Object`, or compute multiple properties before exporting;
- cmd: if re-running is expensive and no safe branch primitive exists, materialize a small temporary result once rather than repeating the expensive command.

## 7. Environment-specific rules

Read `references/PLATFORMS.md` when behavior depends on shell/platform semantics or when a command is not obviously portable.

### POSIX / Linux / BSD / macOS / BusyBox

- Prefer POSIX/common syntax when sufficient.
- Do not assume GNU-only flags such as `find -printf`, `sort -V`, or GNU `sed -i` behavior.
- Do not assume bash features in `/bin/sh`: arrays, `[[ ... ]]`, process substitution, brace expansion, `mapfile`, etc.
- Prefer `printf` over portability-sensitive `echo` usage.
- Treat filenames as opaque values. Use NUL-safe interfaces when available; otherwise prefer `find ... -exec ... {} +` over unsafe word splitting.
- Probe optional GNU/BSD extensions before relying on them.
- BusyBox applets often implement fewer options; inspect applet help and simplify toward the POSIX/common core.

### PowerShell

- Preserve objects; manipulate properties before text formatting.
- Filter left: producer/provider filtering first, then `Where-Object` only when needed.
- Project with `Select-Object`; group with `Group-Object`; sort with `Sort-Object`; count/aggregate with `Measure-Object`; compare sets/collections with `Compare-Object` where appropriate.
- Use `Get-Member` to discover structure instead of parsing display text.
- Keep `Format-Table`, `Format-List`, `Out-String`, and display formatting at the terminal edge.
- Prefer `ConvertFrom-Json`/`ConvertTo-Json`, `Import-Csv`/`Export-Csv`, and object properties over regex parsing when the format is structured.
- Be careful at native-command boundaries: external programs exchange byte/text streams, not live PowerShell objects.
- Avoid aliases in generated automation when they obscure whether a command is a cmdlet or native executable.

### cmd.exe

- Prefer built-ins and Win32 console commands for simple text jobs: `dir`, `where`, `find`, `findstr`, `sort`, `fc`, `for`, `forfiles`, `set /a`, `type`.
- Use `dir /b` when filenames are the data; avoid parsing normal human-formatted `dir` output.
- `for /f` is a parser with its own tokenization rules. Specify `delims=`, `tokens=`, and `usebackq` deliberately.
- Distinguish interactive `%A` from batch-file `%%A` syntax.
- Use `setlocal` to contain environment changes; enable delayed expansion only when required and understand `!` semantics.
- Quote paths. Escape cmd metacharacters (`& | < > ^ ( )`) deliberately when nesting commands.
- Use `findstr` regex only within its actual regex dialect; do not assume PCRE/ERE semantics.
- If the solution turns into multi-layer escaping or simulated data structures, use PowerShell if already available rather than constructing a batch-language monument.

### WSL / MSYS2 / Cygwin / Git Bash

Treat the active Unix-like shell as a Unix pipeline environment, but respect interoperability boundaries:

- Windows and Unix path syntaxes may differ;
- argument/path conversion may occur in compatibility layers;
- line-ending/text-mode behavior can differ at Windows boundaries;
- native Windows programs and Unix tools can have different quoting/path expectations.

Do not mix path conventions casually. Keep one representation through as much of a pipeline as possible and convert only at an explicit boundary.

## 8. Cross-shell semantic mapping

Think in operations first, command names second.

| Operation | POSIX/Unix family | PowerShell | cmd.exe |
|---|---|---|---|
| command discovery | `command -v`, `type` | `Get-Command` | `where` |
| file enumeration | `find` | `Get-ChildItem` | `dir /s /b`, `for /r`, `forfiles` |
| text search | `rg`, `grep` | `rg`, `Select-String` | `rg`, `findstr`, `find` |
| filter | producer flags, `grep`, `awk` | producer flags, `Where-Object` | `findstr`, `find`, `for /f` |
| project | `cut`, `awk`, `jq` | `Select-Object`, `ForEach-Object` | `for /f tokens=` |
| sort | `sort` | `Sort-Object` | `sort` |
| unique/group | `sort -u`, `uniq`, `awk` | `Sort-Object -Unique`, `Group-Object` | stateful `for /f` only when simple |
| aggregate/count | `wc`, `uniq -c`, `awk` | `Measure-Object`, `Group-Object` | `find /c`, `set /a` |
| compare sets | `comm`, `join`, `awk` | `Compare-Object`, keyed objects | `fc` for files; otherwise keep simple |
| JSON | `jq` | `ConvertFrom-Json` / `ConvertTo-Json` | use `jq` if present or promote to PowerShell |
| CSV | `awk` only for truly simple CSV; specialized CLI preferred | `Import-Csv` / `Export-Csv` | promote to PowerShell for quoted CSV |
| fan-out | `tee`, `pee` | `Tee-Object` | materialize small temp output if needed |
| hashing | `sha256sum`/`shasum`/`openssl` as available | `Get-FileHash` | `certutil -hashfile` |
| HTTP | `curl` | `Invoke-RestMethod`, `Invoke-WebRequest`, `curl.exe` | `curl.exe` when present |

This table is a translation guide, not a mandate. A cross-platform CLI such as `rg`, `jq`, `git`, or `gh` may be the best choice in every shell when installed.

## 9. Safety and correctness guardrails

### Never convert untrusted data into shell source

Classic shell-gei may generate commands and pipe them into a shell. For autonomous repository work, preserve the intellectual move but not the unsafe default.

Avoid:

```sh
untrusted_producer | sh
untrusted_producer | bash
eval "$generated"
```

and PowerShell equivalents such as uncontrolled `Invoke-Expression`.

Prefer passing data as arguments/objects:

```sh
producer | xargs ...
producer | parallel ...
```

or native PowerShell object pipelines.

If command generation is genuinely required, inspect the generated plan and ensure all interpolated values are controlled and correctly quoted.

### Filenames are data, not shell words

Never use `for f in $(find ...)` or unquoted command substitution for arbitrary filenames.

Prefer NUL-safe paths where supported or `find -exec ... {} +` as a broadly portable route.

### Do not hide unexpected failure

Do not routinely append `2>/dev/null || :`, `$ErrorActionPreference='SilentlyContinue'`, or `2>nul` merely to make output clean. Silence only expected, understood failure modes.

Use exit status deliberately.

### Mutation should be explicit

For search/analysis, prefer read-only commands. Before bulk mutation:

1. produce the candidate set;
2. sample/inspect it;
3. use a native dry-run/preview mode when available;
4. mutate only after the target set is clear.

## 10. Parallelism is an optimization, not a reflex

Use concurrency only for genuinely independent expensive work.

- Unix: `xargs -P` when available, GNU `parallel` when installed and useful;
- PowerShell 7+: `ForEach-Object -Parallel` only when version/cost justify it;
- cmd.exe: avoid ad-hoc `start /b` fan-out unless the task truly requires it and synchronization is understood.

Do not parallelize tiny operations whose process-start overhead exceeds the work.

## 11. Escalation criteria

Promote the task to a maintained script/program when shell composition becomes worse than the code it replaces.

Strong signals:

- deeply nested state or recursive data structures;
- substantial schema validation;
- complex binary parsing;
- nontrivial exception/retry/recovery logic;
- transactional multi-step mutation;
- security-sensitive parsing or escaping;
- quoting dominates the actual operation;
- the command must become a long-lived tested artifact;
- the same transformation will be reused and maintained.

If a runtime is needed, choose the project's existing language/toolchain rather than introducing a new one just for glue.

## 12. Mental checklist before every investigative command

Ask internally:

1. What shell/data model am I actually in?
2. Is there a purpose-built CLI or producer option that already answers this?
3. Can I filter/project earlier?
4. Can I change the representation so a standard operator solves it?
5. Am I about to parse display formatting instead of structured data?
6. Am I reading or producing far more context than needed?
7. Am I repeating an expensive producer?
8. Is this syntax portable enough for the current environment, or have I probed the extension?
9. Are paths/records safe against whitespace, newlines, quoting, and shell metacharacters?
10. Is a general-purpose runtime actually simpler now, or merely habitual?

For deeper patterns and platform-specific caveats, read `references/REFERENCE.md` and `references/PLATFORMS.md` only as needed. `references/EVALS.md` is a maintainer regression suite and should not be loaded during ordinary task execution.
