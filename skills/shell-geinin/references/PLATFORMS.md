# Platform and Shell Adapter Guide

Use this file when a shell-gei idea is clear but the concrete syntax depends on the execution environment.

The guiding rule is **capability first, platform second**. The executable that actually resolves determines semantics more reliably than the host OS label.

## 1. Environment classification

### POSIX-ish text pipeline

Typical shells:

- `/bin/sh`
- bash
- dash
- ksh
- zsh used in sh-like command style
- BusyBox `ash` / `hush`
- WSL bash/sh
- MSYS2 bash
- Cygwin bash
- Git Bash

Baseline mental model: newline/byte streams, file descriptors, exit codes, external filters.

### PowerShell object pipeline

Typical shells:

- Windows PowerShell 5.1
- PowerShell 7+ on Windows
- PowerShell 7+ on Linux/macOS

Baseline mental model: .NET objects, properties, typed collections, success/error/information streams, native-command text boundaries.

### cmd.exe text shell

Typical shell:

- `cmd.exe`
- `.bat` / `.cmd`

Baseline mental model: text, environment variables, `for`, token parsing, ERRORLEVEL, pipes/redirection.

## 2. Portable Unix baseline

When the exact Unix family is unknown, start from POSIX/common utility behavior.

High-value baseline tools include:

- shell builtins: `printf`, `test`/`[`, `read`, `cd`, `command`, `getopts`;
- text: `awk`, `sed`, `grep`, `cut`, `paste`, `join`, `comm`, `sort`, `uniq`, `tr`, `wc`, `head`, `tail`, `fold`;
- files: `find`, `ls`, `cp`, `mv`, `rm`, `mkdir`, `dirname`, `basename`;
- process/system: `ps`, `kill`, `uname`;
- archive/basic: `tar` where available.

Prefer `printf` for generated output because `echo` option/backslash behavior varies across implementations and historical modes.

### Portable shell syntax to prefer

```sh
if command -v rg >/dev/null 2>&1; then
  rg 'pattern' .
else
  grep -R 'pattern' .
fi
```

```sh
find . -type f -exec grep -l 'pattern' {} +
```

The second pattern avoids filename word-splitting without depending on `xargs -0`.

### Syntax not safe to assume in `/bin/sh`

Do not assume:

- `[[ ... ]]`;
- arrays;
- process substitution `<(...)` / `>(...)`;
- brace expansion `{1..10}`;
- `mapfile` / `readarray`;
- `shopt`;
- `PIPESTATUS`;
- bash regex operator details;
- zsh glob qualifiers.

Use them when the active shell is known and they materially improve clarity.

## 3. GNU/Linux extensions

GNU tools are often feature-rich and excellent for shell-gei, but extensions should not leak into supposedly portable commands accidentally.

Common GNU-only or GNU-shaped features to probe before relying on cross-Unix:

- `find -printf`;
- GNU-specific `find` predicates/actions;
- `sort -V`, `sort -h`, some long options;
- `sed -i` no-backup syntax;
- `grep -P`;
- `xargs -r` behavior/options;
- `readlink -f`;
- `date -d`;
- `stat -c`;
- `tac` availability.

If a GNU feature drastically shortens the command and GNU behavior is confirmed, use it. Otherwise choose the portable form.

Useful probes:

```sh
find --version 2>/dev/null | head -1
sort --version 2>/dev/null | head -1
sed --version 2>/dev/null | head -1
```

Do not depend on `--version` itself in a generic probe; absence simply means “not confirmed GNU”.

## 4. BSD and macOS

BSD utilities frequently share names with GNU tools but differ in options.

Important recurring portability traps:

### `sed -i`

GNU and BSD/macOS command-line forms differ. For portable automation, prefer a temporary file plus atomic-ish replacement, or `sponge` if it is explicitly available and the semantics are acceptable.

### `stat`

GNU commonly uses `stat -c`; BSD/macOS uses `stat -f` for format strings. Do not guess. Probe help or avoid `stat` formatting if `find`/shell metadata already answers the question.

### `date`

GNU `date -d` and BSD/macOS date manipulation syntax differ substantially. Keep date transformations simple or use the environment-native tool after probing.

### reverse-lines

GNU commonly provides `tac`; BSD/macOS environments may instead have other mechanisms. If reverse-lines is essential and no native command is known, a small `awk` accumulation can be more portable than assuming `tac`.

### GNU tools installed on BSD/macOS

Homebrew/pkg environments may provide `gawk`, `gsed`, `gfind`, `gsort`, etc. Use prefixed GNU tools only after `command -v` confirms them and only when their extension adds real value.

## 5. BusyBox and embedded Unix

BusyBox intentionally provides small implementations of many Unix utilities and can be configured so applets/options vary by build.

Assume less, not more.

Good strategy:

1. identify BusyBox if relevant;
2. inspect the specific applet help;
3. reduce the command toward POSIX/common options;
4. avoid requiring optional utilities that may not exist at all.

Useful discovery examples:

```sh
busybox 2>/dev/null | head
busybox --list 2>/dev/null | head
busybox find --help 2>&1 | head -40
```

Do not assume GNU long options or full GNU regex/tool behavior.

## 6. PowerShell — object-shell shell-gei

PowerShell is not “Unix with longer command names”. Its pipeline transmits objects between PowerShell commands.

The PowerShell version of shell-gei is therefore **property algebra** rather than premature text munging.

### Discovery trio

```powershell
Get-Command Get-ChildItem
Get-Help Get-ChildItem -Parameter Filter
Get-Process | Get-Member
```

### Filter as far left as possible

Best shape:

```powershell
Get-ChildItem -Path . -File -Recurse -Filter '*.log' |
  Select-Object -First 50 FullName, Length, LastWriteTime
```

Less desirable when the provider can already filter:

```powershell
Get-ChildItem -Path . -File -Recurse |
  Where-Object Name -Like '*.log'
```

Use `Where-Object` for predicates the producer cannot express.

### Preserve objects until the display edge

Good:

```powershell
Get-Process |
  Where-Object WorkingSet64 -GT 500MB |
  Sort-Object WorkingSet64 -Descending |
  Select-Object -First 10 Name, Id, WorkingSet64
```

Bad mental model:

```powershell
Get-Process | Format-Table | Select-String ...
```

`Format-*` is for presentation. Once formatted, downstream processing no longer has the original domain objects in the useful form you wanted.

### Core operator mapping

- filter: `Where-Object`
- projection/limit/unique: `Select-Object`
- transform/property/method: `ForEach-Object`
- ordering: `Sort-Object`
- grouping/frequency: `Group-Object`
- aggregation/count: `Measure-Object`
- set-ish comparison: `Compare-Object`
- fan-out/materialization: `Tee-Object`
- regex/text search: `Select-String`
- structure discovery: `Get-Member`

### Structured formats

Prefer:

```powershell
Get-Content data.json -Raw |
  ConvertFrom-Json |
  Select-Object -ExpandProperty items
```

or, when a native cross-platform `jq` is already present and the operation is naturally JSON-stream-oriented, use `jq` directly.

For CSV, prefer `Import-Csv` / `Export-Csv` over splitting lines manually because quoting and embedded delimiters make CSV nontrivial.

### Native executable boundary

External/native commands do not emit live PowerShell objects. Their stdout enters the PowerShell pipeline as native command output. Decide explicitly whether to:

- keep using a purpose-built native CLI (`rg`, `jq`, `git`, `gh`);
- convert structured text at the boundary;
- return to object-native cmdlets.

Avoid needless ping-pong between object and formatted-text representations.

### Native command name collisions

PowerShell aliases can shadow familiar Unix names.

Use:

```powershell
Get-Command sort -All
Get-Command curl -All
```

When an actual executable matters, use its executable name/path such as `sort.exe` or `curl.exe` if that is the intended tool.

### Version-sensitive features

Before relying on newer syntax or parallelism:

```powershell
$PSVersionTable.PSVersion
```

`ForEach-Object -Parallel` is a PowerShell 7-era feature; do not assume it in Windows PowerShell 5.1.

## 7. Windows cmd.exe — constrained shell-gei

cmd can still do useful dataflow work when the problem fits its text model.

### Command discovery

```bat
where git
where rg
where jq
```

`where` also respects executable extension behavior on Windows.

### Enumerate paths without decoration

```bat
dir /s /b *.log
```

Prefer `/b` when downstream commands need paths. Parsing normal `dir` output is fragile and locale-dependent.

### Search text

Literal/simple:

```bat
find /i "error" app.log
```

Regex-ish/multi-file:

```bat
findstr /s /n /i /r "error warning" *.log
```

Files containing a match:

```bat
findstr /s /i /m "error" *.log
```

`findstr` regex is its own limited dialect. Do not transplant PCRE or GNU grep assumptions into it.

### Parse lines with `for /f`

Entire line:

```bat
for /f "usebackq delims=" %L in ("file.txt") do @echo(%L
```

Selected comma-separated fields:

```bat
for /f "usebackq tokens=2,3* delims=," %A in ("file.txt") do @echo %A %B %C
```

Inside a batch file, use `%%L`, `%%A`, etc.

Be aware that `for /f` has its own rules for blank lines, comments/eol, tokenization, quoting, and command substitution. Treat it as a parser, not a transparent line reader.

### Parse command output

```bat
for /f "usebackq delims=" %L in (`where git`) do @echo(%L
```

Nested metacharacters often require escaping with `^`.

### Recursive/batch file operations

```bat
for /r %F in (*.log) do @echo %~fF
```

Useful `%~` modifiers include full path, drive/path, extension, size/time metadata combinations.

For file selection by age/name and dispatch, consider `forfiles`:

```bat
forfiles /s /m *.log /d -7 /c "cmd /c echo @path @fsize"
```

### Count lines

A classic cmd idiom:

```bat
type file.txt | find /c /v ""
```

Use only when text semantics are acceptable.

### Integer arithmetic

```bat
set /a n=40+2
```

Do not summon Python for trivial integer arithmetic from cmd.

### Delayed expansion

When a variable is modified and consumed inside a parenthesized block, normal `%VAR%` expansion can happen too early. Use locally-scoped delayed expansion only when needed:

```bat
setlocal EnableDelayedExpansion
set n=0
for %F in (*) do @(
  set /a n+=1
  echo !n! %F
)
endlocal
```

In a batch file, double the loop variable percent sign.

Remember that `!` then becomes syntactically significant. Filenames/data containing exclamation marks can make delayed-expansion code tricky. Avoid enabling it globally without a reason.

### When to leave cmd

Promote to PowerShell if available when you need:

- real structured JSON/CSV manipulation;
- rich object metadata;
- complex grouping/joining;
- nested state;
- robust Unicode-heavy processing;
- multi-layer command escaping that obscures intent.

The objective is shell economy, not proving cmd is Turing-complete.

## 8. WSL

WSL runs a real GNU/Linux user environment on Windows, so inside the distribution use the Linux/POSIX adapter.

The important boundary is Windows ↔ Linux interop.

Guidelines:

- keep Linux paths as Linux paths while using Linux tools;
- keep Windows paths as Windows paths while using native Windows tools;
- convert at an explicit boundary rather than repeatedly;
- remember that invoking `wsl.exe` from PowerShell/cmd and invoking Windows executables from WSL crosses quoting/path/environment models.

If the entire task is naturally Linux-shaped, stay inside WSL for the pipeline instead of bouncing every stage across the boundary.

## 9. MSYS2 and Git Bash

MSYS2 provides a Unix-like shell/tool environment on Windows and may automatically convert Unix-looking paths/arguments when invoking native Windows programs.

Guidelines:

- use Unix-style pipelines internally;
- be cautious when passing `/foo/bar`-shaped arguments to native `.exe` tools;
- avoid unnecessary Unix↔Windows path conversion mid-pipeline;
- inspect `cygpath` or environment-specific path conversion facilities when crossing the boundary.

Git Bash shares much of the same practical concern: the active shell is Unix-like, but many targets are native Windows executables.

## 10. Cygwin

Cygwin provides GNU/Open Source Unix tools plus a POSIX compatibility layer on Windows.

Use Unix shell-gei internally, while remembering:

- Windows and POSIX paths coexist;
- permissions/ownership map onto Windows security;
- text/binary and line-ending behavior can matter at interoperability boundaries;
- native Windows tools do not necessarily share Cygwin path assumptions.

Again, minimize boundary crossings.

## 11. Cross-platform command collision rules

Some names mean different things depending on shell resolution.

Examples:

- PowerShell `sort` may resolve to `Sort-Object`, while Unix `sort` is a text utility;
- PowerShell `cat` may resolve to `Get-Content`;
- `where` in PowerShell is commonly an alias context, while `where.exe` is the Windows path-search command;
- `curl` may resolve differently across Windows PowerShell/Core and installed binaries.

When semantics matter, inspect resolution or use the explicit full cmdlet/executable name.

## 12. Generic portability decision tree

1. **Is there a domain-specific cross-platform CLI?**
   - use it.
2. **Does the active shell preserve useful structure?**
   - PowerShell: stay objects;
   - Unix/cmd: stay text/records.
3. **Can the producer filter/project?**
   - do that first.
4. **Can a common baseline operator solve it?**
   - prefer that.
5. **Would an extension materially simplify it?**
   - probe and use if available.
6. **Is the pipeline becoming more fragile than a maintained program?**
   - escalate.
