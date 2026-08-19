# Research Notes — Cross-Platform Shell-Gei

This document records the design analysis behind the cross-platform `shell-geinin` skill. It is not required during normal invocation.

## 1. Core thesis

The transferable essence of shell-gei is **not a fixed set of GNU commands**.

It is a problem-solving discipline:

1. observe the active environment;
2. choose the environment's native data model;
3. transform the problem into records/objects/sets/relations;
4. push filtering and projection toward the source;
5. compose small existing operators;
6. avoid repeated work and premature general-purpose coding;
7. return only the minimal result needed;
8. preserve safety, quoting, ordering, and representation boundaries.

This makes shell-gei applicable beyond Bash/Linux.

Unix shell-gei uses text streams. PowerShell can express the same cognitive style over object pipelines. cmd.exe supports a narrower form over text, `for` parsing, pipes, redirection, and Win32 console utilities.

## 2. Shell-gei culture and study-session findings

### Organizer / primary Shell-Gei sources

- Ryuichi Ueda, Shell-Gei top page
  - https://b.ueda.tech/?page=01434
  - Shell-gei is framed around accomplishing investigation, calculation, and text processing directly through command-line composition.

- Shell-Gei study-session archive
  - https://b.ueda.tech/?page=00684
  - Large archive of problems, answer pages, videos, and participant links.

- 17th study session
  - https://b.ueda.tech/?post=06409
  - Explicit concrete-first rule: solve the given problem before trying to generalize it.

- 16th study session
  - https://b.ueda.tech/?post=05644
  - Log-analysis examples normalize timestamp representation before downstream operations.

- 15th study session
  - https://b.ueda.tech/?post=05093
  - Native options such as `grep -L` can eliminate downstream count/filter stages.

- 29th study session
  - https://b.ueda.tech/?post=09870
  - `join`-style relational reasoning and the classic representation-change move of padding/reshaping a triangle into a rectangle before transformation.

- 30th study session
  - https://b.ueda.tech/?post=10134
  - Native producer limits (`grep -m`) and candidate-universe/set-difference reasoning.

### Community analyses

- NTT docomo Business Engineers' Blog — Shell One-Liner 160 knock completion
  - https://engineers.ntt.com/entry/2022/12/04/071122
  - Highlights command knowledge, shell features, options, regex, composition patterns, and an art component.

- papiro — stream-oriented thinking / `xargs`
  - https://papiro.hatenablog.jp/entry/2015/10/26/181252
  - Contrasts procedural thought with stream-processing thought.

- papiro — 61st study session
  - https://papiro.hatenablog.jp/entry/2022/09/25/185705
  - Stable-sort semantics and flatten/restore representation tricks.

- papiro — 62nd study session
  - https://papiro.hatenablog.jp/entry/2022/12/04/225616
  - Shallow binary inspection and extracting only relevant archive-contained content.

- papiro — 65th study session
  - https://papiro.hatenablog.jp/entry/2023/07/23/214432
  - Explode characters → `uniq -c` run-length style transformation.

- Engineering Type event report
  - https://type.jp/et/feature/18048/
  - Solution-sharing culture, compact representations, symmetry, and moreutils `pee` examples.

- ngyuki — moreutils survey
  - https://qiita.com/ngyuki/items/ad7d52186a84cc973438
  - Practical behavior of `chronic`, `combine`, `errno`, `ifne`, `isutf8`, `mispipe`, `pee`, `sponge`, and `ts`.

- Shell-Gei Advent Calendar 2023
  - https://qiita.com/advent-calendar/2023/shellgei
  - Shows the culture extending beyond a frozen GNU/Linux recipe set, including a PowerShell `nl` article and discussions of command behavior/portability.

## 3. Unix/POSIX portability research

### POSIX Shell and Utilities

- The Open Group, POSIX.1 Shell & Utilities, Issue 8 / 2024
  - https://pubs.opengroup.org/onlinepubs/9799919799/utilities/contents.html
  - Defines the portable shell command language and standard utility family including `awk`, `grep`, `join`, `paste`, `sort`, `uniq`, `xargs`, etc.

Design consequence:

- use POSIX/common behavior as the unknown-Unix baseline;
- treat GNU conveniences as optional accelerators;
- distinguish shell syntax extensions from portable `/bin/sh` syntax.

### GNU tool ecosystem

Representative manuals/documentation researched include GNU coreutils/findutils/gawk/sed references.

Design consequence:

- GNU utilities provide powerful operator-level shortcuts, but the skill must probe rather than assume them on BSD/macOS/BusyBox;
- NUL-safe path processing and rich sort/find options are valuable when actually available.

### BusyBox

- BusyBox documentation
  - https://busybox.net/downloads/BusyBox.html
  - BusyBox combines small implementations of many Unix utilities; available options/features can be reduced and builds are configurable.

Design consequence:

- capability probe the applet/build;
- fall back toward the common/POSIX operator set;
- never assume “Linux” implies GNU userland.

## 4. PowerShell research

PowerShell is central to the cross-platform version because it demonstrates that shell-gei is better understood as **pipeline algebra** than “pipe newline text through GNU commands”.

### Object pipeline

- PowerShell overview
  - https://learn.microsoft.com/powershell/scripting/overview

- about_Pipelines
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_pipelines

- PowerShell 101: One-liners and the pipeline
  - https://learn.microsoft.com/powershell/scripting/learn/ps101/04-pipelines

Key finding:

- PowerShell pipelines pass objects between cmdlets;
- Microsoft explicitly teaches pipeline composition and “filtering left”; therefore a shell-gei translation should preserve objects and move predicates toward the producer rather than convert everything into text.

### Discovery as a first-class skill

- Get-Help / Get-Command
  - https://learn.microsoft.com/powershell/scripting/learn/ps101/02-help-system

- Get-Member / object discovery
  - https://learn.microsoft.com/powershell/scripting/learn/ps101/03-discovering-objects

Design consequence:

- `Get-Command`, `Get-Help`, `Get-Member` are the PowerShell analogues of `command -v`, `man`, and inspecting stream structure.

### Filtering and projection

- `Where-Object`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.core/where-object

- `Select-Object`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-object

- `Get-ChildItem -Filter`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem

Microsoft documentation notes that provider filtering is applied while objects are retrieved and can be more efficient than retrieving everything then filtering in PowerShell.

Design consequence:

- source/provider filter → `Where-Object` → text parsing is the PowerShell version of “producer-side projection first”.

### Grouping, sorting, measuring

- `Group-Object`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/group-object

- `Sort-Object`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/sort-object

- `Measure-Object`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/measure-object

These directly map shell-gei operations such as grouping/frequency/order/reduction into object-native operators.

### Structured format handling

- Microsoft.PowerShell.Utility module
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/
  - Includes JSON/CSV conversion, `Compare-Object`, `Group-Object`, `Measure-Object`, `Select-String`, `Sort-Object`, `Tee-Object`, etc.

Design consequence:

- do not regex-parse JSON/CSV when PowerShell already has structured converters.

### Formatting boundary

- `Format-Table`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/format-table

- `Out-File`
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/out-file

`Out-File` uses PowerShell's formatting system; formatted display is not an ideal programmatic interchange representation.

Design consequence:

- keep `Format-*` and display conversion at the terminal edge;
- manipulate/select objects first.

### Native command boundary

- about_Output_Streams
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_output_streams

- about_Parsing
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_parsing

Design consequence:

- native executable stdout/stderr boundaries differ from live object pipelines;
- quoting and argument-passing behavior must be treated explicitly when crossing from PowerShell to native CLIs.

### Cross-platform identity

- about_Automatic_Variables
  - https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_automatic_variables
  - `$PSVersionTable`, `$IsWindows`, `$IsLinux`, `$IsMacOS` support environment discovery in modern PowerShell.

Design consequence:

- the skill should classify by active shell/data model rather than assuming PowerShell means Windows.

## 5. cmd.exe research

Microsoft's current Windows Commands documentation explicitly retains both Command shell and PowerShell, while recommending PowerShell for robust modern automation. The cross-platform skill therefore supports cmd as a constrained but legitimate shell-gei environment rather than pretending it has Unix/PowerShell structure.

### Windows Commands overview

- https://learn.microsoft.com/windows-server/administration/windows-commands/windows-commands

### `for` and `for /f`

- https://learn.microsoft.com/windows-server/administration/windows-commands/for

Important findings:

- `for /f` parses files, strings, or captured child-command output;
- it has explicit `tokens=`, `delims=`, `eol=`, `skip=`, and `usebackq` parsing rules;
- interactive and batch variable syntax differ (`%A` vs `%%A`).

Design consequence:

- treat `for /f` as a parser whose contract must be explicit, not as a transparent Unix `while read` equivalent.

### Search/filter primitives

- `findstr`
  - https://learn.microsoft.com/windows-server/administration/windows-commands/findstr

- `find`
  - https://learn.microsoft.com/windows-server/administration/windows-commands/find

- `sort`
  - https://learn.microsoft.com/windows-server/administration/windows-commands/sort

- `where`
  - https://learn.microsoft.com/windows-server/administration/windows-commands/where

- `forfiles`
  - https://learn.microsoft.com/windows-server/administration/windows-commands/forfiles

- `dir`
  - https://learn.microsoft.com/windows-server/administration/windows-commands/dir

Design consequences:

- `dir /b`, `findstr /m`, `find /c`, `where`, and `forfiles` are cmd-side examples of producer-side reduction;
- avoid parsing human-formatted `dir` output when bare paths or metadata variables are available;
- do not assume `findstr` regex equals GNU grep/PCRE.

### Delayed expansion

- `setlocal`
  - https://learn.microsoft.com/windows-server/administration/windows-commands/setlocal

Design consequence:

- delayed expansion solves block-time variable expansion problems but changes `!` semantics, so it should be enabled locally and intentionally.

## 6. Windows Unix-compatibility layers

### WSL

- WSL documentation
  - https://learn.microsoft.com/windows/wsl/

- WSL basic commands
  - https://learn.microsoft.com/windows/wsl/basic-commands

WSL runs GNU/Linux tools directly on Windows.

Design consequence:

- inside WSL, use the Linux adapter;
- crossing WSL ↔ Windows is a representation/quoting/path boundary and should be minimized.

### MSYS2

- MSYS2 overview
  - https://www.msys2.org/

- MSYS2 environments
  - https://www.msys2.org/docs/environments/

- filesystem/path conversion
  - https://www.msys2.org/docs/filesystem-paths/

Design consequence:

- use Unix pipeline reasoning internally;
- recognize automatic path conversion and native-Windows program boundaries.

### Cygwin

- Cygwin overview
  - https://www.cygwin.com/

- text/binary modes
  - https://cygwin.com/cygwin-ug-net/using-textbinary.html

- POSIX/security mapping
  - https://cygwin.com/cygwin-ug-net/ntsec.html

Design consequence:

- treat Cygwin as a Unix-like shell environment with explicit Windows interoperability boundaries.

## 7. Unix philosophy as the broader substrate

Shell-gei intensifies an older Unix composition tradition: small tools, pipelines, and output designed to become another program's input.

Relevant historical/general sources located during research include Unix pipeline/oral-history material and USENIX work on shell pipelines.

The cross-platform skill deliberately translates this from “text is always universal” to:

> Preserve the richest simple representation the active shell already composes well.

For Unix that is usually text/records. For PowerShell it is objects. For cmd it is constrained text.

## 8. Major design changes from the macOS-centric skill

### Before

- Unix/macOS-first command inventory;
- GNU variants and moreutils prominent;
- general-purpose runtimes avoided;
- output/token minimization emphasized.

### Now

1. **Environment classifier first**
   - active shell/data model beats host OS.
2. **Capability-based tool selection**
   - `command -v` / `Get-Command` / `where`.
3. **Three pipeline families**
   - Unix text streams;
   - PowerShell object pipelines;
   - cmd text/token pipelines.
4. **POSIX baseline plus optional extensions**
   - GNU/BSD/BusyBox differences are explicit.
5. **Boundary minimization**
   - PowerShell ↔ native text, WSL ↔ Windows, MSYS/Cygwin ↔ native Windows are treated as representation transitions.
6. **Cross-shell semantic mapping**
   - operations are primary; command spellings are adapters.
7. **No false portability**
   - process substitution, GNU flags, PowerShell 7-only features, cmd delayed expansion, etc. are not silently assumed.
8. **Object-gei for PowerShell**
   - filter left, project properties, group/measure/sort objects, format only at the edge.
9. **cmd is supported without glorifying batch gymnastics**
   - use concise native primitives; promote to PowerShell when structure exceeds cmd's sweet spot.

## 9. Agent-specific adaptation

Historical shell-gei sometimes rewards surprise, extreme one-liners, command synthesis, and assumptions acceptable in contest/study environments.

An autonomous coding agent has different priorities:

1. correctness;
2. safety;
3. minimal context returned;
4. low repeated I/O/tool calls;
5. representation clarity;
6. native/purpose-built operators;
7. portability appropriate to the actual environment;
8. brevity;
9. artistry.

This keeps the shell-gei cognitive style while rejecting dangerous or opaque golf.

## 10. Final distilled definition

For this skill, a “shell geinin” is an agent that:

- sees a problem as a transformable data representation;
- discovers tools before inventing code;
- selects operators by semantics, not habit;
- pushes predicates and projection left;
- prefers relations/sets/groups/alignment over bespoke loops;
- preserves the shell's native structure;
- crosses representation boundaries intentionally;
- pays for output with the model's context budget;
- uses the shortest *clear and robust* path to the answer;
- knows when the shell has stopped being the right abstraction.

## 11. PowerShell shell-gei precedents

Cross-platform shell-gei is not purely hypothetical. Community material includes attempts to solve Shell-Gei study-session problems directly in PowerShell, including reports from the early Shell-Gei study-session era, and later Shell-Gei Advent Calendar entries that deliberately recreate Unix-style operators in PowerShell.

Representative sources:

- PowerShell solution report for the 3rd Shell-Gei study session
  - https://tech.guitarrapc.com/entry/2013/02/18/070226
- Shell-Gei Advent Calendar 2023
  - https://qiita.com/advent-calendar/2023/shellgei

Design consequence:

- carry over the *problem-transformation discipline*, not the Unix command spelling;
- in PowerShell, object/property operators are usually more native than forcing every value through text merely to imitate Unix.
