# Shell-Gei Technique Atlas — Cross-Platform

This file catalogs reusable reasoning patterns. Translate each pattern into the active shell's data model rather than mechanically copying Unix syntax.

## A. Producer-side projection

**Idea:** ask the source for less data before adding another stage.

Unix examples:

```sh
rg -l 'TODO' .
git log -n 20 --oneline
find . -type f -name '*.md'
```

PowerShell examples:

```powershell
Get-ChildItem -File -Filter '*.md'
Get-Process -Name 'pwsh'
Select-String -Path *.log -Pattern 'ERROR' -List
```

cmd examples:

```bat
dir /s /b *.md
findstr /s /m /i "ERROR" *.log
```

**Reasoning:** every byte/object removed at the producer is work that downstream stages and the model never have to perform.

## B. Normalize at the edge

**Idea:** irregular input should become one canonical intermediate representation once.

Examples:

- normalize timestamps to sortable ISO-ish form;
- normalize path separators only when crossing environments;
- normalize case only if the task is explicitly case-insensitive;
- turn semi-structured lines into `key<TAB>value` records;
- in PowerShell, convert JSON/CSV into objects once, then work on properties.

Do not re-parse the same irregularity in every stage.

## C. Explode → generic operator → implode

**Idea:** expose the unit that a standard operator understands.

Classic Unix shape:

```sh
# conceptual
string -> one symbol/record per line -> sort/uniq/awk -> fold back
```

PowerShell analogue:

```powershell
$text.ToCharArray() |
  Group-Object |
  Sort-Object Count -Descending
```

The point is not characters specifically. Explode can mean files → rows, JSON array → objects, archive → members, path → components, or log block → records.

## D. Tag → destructive transform → untag

**Idea:** attach identity/order before sorting, grouping, flattening, or deduplicating.

Unix:

```sh
awk '{print NR "\t" $0}' input |
  sort -k2,2 |
  ...
```

PowerShell usually already carries identity as object properties. Add one only if needed:

```powershell
$i = 0
Get-Content file.txt | ForEach-Object {
  [pscustomobject]@{ Index = ++$i; Text = $_ }
}
```

cmd can carry `%~fF` or a counter when needed, but avoid elaborate tagging if PowerShell is a clearer shell for the task.

## E. Sort/group as an algorithm

`sort` is not merely display formatting.

It enables:

- duplicate adjacency;
- grouping;
- merge joins;
- set comparison;
- rank/quantile approximations;
- restoring deterministic order.

Unix frequency pattern:

```sh
producer | sort | uniq -c | sort -nr
```

PowerShell:

```powershell
producer |
  Group-Object Property |
  Sort-Object Count -Descending
```

Check lexical/numeric/culture/stability semantics.

## F. Universe vs observed set

**Idea:** generate all valid candidates, normalize observations, compare the sets.

Unix:

```sh
# both streams sorted uniquely
comm -23 expected observed
```

PowerShell:

```powershell
Compare-Object -ReferenceObject $expected -DifferenceObject $observed
```

For cmd, if set logic becomes nontrivial, PowerShell is usually the clearer native Windows shell.

This pattern replaces repeated “does candidate X exist?” probes.

## G. Keyed relation instead of nested lookup

Unix relational primitives:

- `join` for equality join on sorted keys;
- `comm` for set relation on sorted lines;
- `paste` for positional zip;
- associative `awk` for small in-memory keyed state.

PowerShell analogues:

- `Group-Object -AsHashTable` where appropriate;
- hashtable keyed lookup;
- `Compare-Object` for difference;
- object properties for identity.

Do not force a text `join` idiom into PowerShell if objects already expose keys.

## H. Align shifted views instead of manual indexing

Unix:

```sh
paste file <(tail -n +2 file)
```

when the active shell supports process substitution.

Portable Unix alternatives may use temporary streams/files or `awk` previous-record state.

PowerShell can keep previous state in `ForEach-Object` when that is the natural object-stream formulation.

The key idea: transform an index problem into an adjacency/alignment problem.

## I. Reshape → operate → restore

If the geometry/layout is awkward, reshape it into something generic tools already understand.

Examples:

- ragged triangle → pad to rectangle → transpose/rotate → strip padding;
- multi-line block → sentinel-separated single record → regex → restore lines;
- hierarchical names → path-component records → group/join → reconstruct.

This is one of the deepest shell-gei habits: solve an easier isomorphic problem.

## J. Symmetry and mirror reuse

Generate one portion and derive the rest through reversal/reflection/duplication.

Unix tools may include `rev`, `tac` if present, `paste`, `awk`, `tee`, `pee`.

PowerShell may use `[array]::Reverse()`, collection slicing, or object reuse when that is clearer—but do not invoke .NET gymnastics merely for golf.

## K. Native option as a pipeline stage eliminator

Before adding a filter, ask whether the producer can perform it.

Examples:

### Search

```sh
grep -L PATTERN files...
grep -m 1 PATTERN file
rg -l PATTERN .
```

```powershell
Select-String -Path *.log -Pattern 'ERROR' -List
```

```bat
findstr /s /m /i "ERROR" *.log
```

### Files

```sh
find . -type f -name '*.log' -size +1k
```

```powershell
Get-ChildItem -File -Recurse -Filter '*.log'
```

```bat
forfiles /s /m *.log /d -7 /c "cmd /c echo @path"
```

## L. One expensive source, many cheap views

Unix:

```sh
expensive_source |
  tee cache.tmp |
  first_analysis
```

or when moreutils exists:

```sh
expensive_source |
  pee 'analysis_one' 'analysis_two'
```

PowerShell:

```powershell
expensive_source |
  Tee-Object -Variable items |
  first_analysis
# then use $items if retaining it is cheaper than rerunning the source
```

Be conscious of memory. Materializing millions of objects just to avoid a cheap producer can be worse.

## M. Exit status is information

Unix:

```sh
if grep -q PATTERN file; then
  ...
fi
```

cmd:

```bat
findstr /i "PATTERN" file.txt >nul
if errorlevel 1 (echo not-found) else (echo found)
```

PowerShell cmdlets often return objects and errors rather than only Unix-style status. Native executable exit status is available through `$LASTEXITCODE`.

Do not parse textual “success” when the command already exposes success/failure structurally.

## N. Shallow binary probe

Ask the minimum question.

Unix-ish:

```sh
file artifact
od -An -tx1 -N16 artifact
xxd -l 16 artifact 2>/dev/null
```

Windows PowerShell:

- `Get-Item` for metadata;
- `Get-FileHash` for hashing;
- `Format-Hex` when available for byte inspection.

cmd:

- `certutil -hashfile` for hashing;
- `fc /b` for binary comparison.

Do not deploy a full parser solely to identify a magic number or equality.

## O. Structure-preserving JSON work

Cross-platform `jq` is often ideal for native JSON text:

```sh
jq -c '.items[] | select(.state == "open") | {id,name}' data.json
```

In PowerShell, if the data naturally participates in object-native processing:

```powershell
$data = Get-Content data.json -Raw | ConvertFrom-Json
$data.items |
  Where-Object state -EQ 'open' |
  Select-Object id, name
```

Do not serialize PowerShell objects to JSON and parse them with text tools unless there is a concrete boundary reason.

## P. CSV is not “split on comma”

Quoted delimiters and embedded newlines make real CSV structured data.

Prefer:

- PowerShell `Import-Csv` / `Export-Csv`;
- a purpose-built CSV CLI if already installed;
- `awk` only when the input contract explicitly guarantees simple delimiter-separated records without CSV quoting complexity.

cmd.exe should generally promote real CSV work to PowerShell if available.

## Q. Filesystem-safe dispatch

Unsafe Unix anti-pattern:

```sh
for f in $(find . -type f); do ...; done
```

Preferred broad route:

```sh
find . -type f -exec command {} +
```

When both sides support NUL delimiters:

```sh
find . -type f -print0 | xargs -0 command
```

In PowerShell, `Get-ChildItem` passes FileInfo objects and avoids shell word splitting:

```powershell
Get-ChildItem -File -Recurse | ForEach-Object {
  # $_ is an object, not a whitespace-split filename
}
```

In cmd, always quote `%~fF` when passing it as a path argument.

## R. Stable-order preservation

If grouping/sorting is temporary and original order matters:

1. attach index/order identity;
2. sort/group;
3. perform operation;
4. sort back by index;
5. remove index.

Unix `sort -s` may help when supported and correctly keyed, but portability and locale should be considered.

PowerShell supports property-based sorting; explicitly carry an index if stable restoration matters rather than assuming incidental order.

## S. Locale/culture is part of the algorithm

Unix sort/character classes can depend on locale.

For byte-oriented deterministic data processing where appropriate:

```sh
LC_ALL=C sort
```

Do not force `C` locale when human-language collation is actually part of the requirement.

PowerShell comparisons/sorting can have culture and case semantics. Specify comparison behavior when correctness depends on it.

cmd output can be locale-sensitive, especially human-formatted system commands. Prefer machine-shaped/bare output or PowerShell objects over parsing localized display text.

## T. Command discovery is itself shell-gei

Do not write code to rediscover installed capabilities.

Unix:

```sh
command -v rg jq gawk gsed gfind parallel sponge 2>/dev/null
```

PowerShell:

```powershell
Get-Command rg,jq,parallel,Get-FileHash -ErrorAction SilentlyContinue
```

cmd:

```bat
where rg 2>nul
where jq 2>nul
```

Then choose the strongest available primitive.

## U. Command-name collision handling

Before using a familiar short name in a mixed environment, ask what it resolves to.

PowerShell examples:

```powershell
Get-Command sort -All
Get-Command cat -All
Get-Command curl -All
```

Unix examples:

```sh
type -a awk 2>/dev/null || command -V awk
type -a sed 2>/dev/null || command -V sed
```

Do not assume semantic identity from spelling.

## V. Moreutils when present

These are optional accelerants, never requirements.

- `sponge`: consume all input before replacing a file;
- `pee`: fan one stream into multiple commands;
- `chronic`: suppress successful command output, show output on failure;
- `ifne`: run a command only when input is non-empty;
- `combine`: boolean/set-like operations on line streams;
- `mispipe`: preserve/return status from an earlier pipeline command;
- `ts`: timestamp lines;
- `isutf8`: UTF-8 validation;
- `errno`: errno lookup.

Use them only after capability discovery.

## W. Parallelism threshold

Parallelism is useful when:

- each item is expensive;
- items are independent;
- ordering is irrelevant or explicitly restored;
- the producer/consumer can tolerate concurrency;
- process startup is small relative to item work.

Unix:

```sh
find ... -print0 | xargs -0 -P4 command
```

or GNU Parallel when available.

PowerShell 7+:

```powershell
$items | ForEach-Object -Parallel { ... } -ThrottleLimit 4
```

Do not assume PowerShell 7 when Windows PowerShell 5.1 may be active.

## X. PowerShell “filter left” hierarchy

From strongest to weakest when applicable:

1. server/source query filter;
2. provider/cmdlet `-Filter` or domain parameter;
3. pipeline `Where-Object`;
4. stringify and regex parse only at a text boundary.

This mirrors Unix producer-side filtering and is a direct object-pipeline analogue of shell-gei economy.

## Y. PowerShell formatting is terminal-stage rendering

Keep:

```powershell
... | Select-Object Name,Length | Format-Table
```

at the end.

For programmatic export, use structured serializers such as `Export-Csv` or `ConvertTo-Json`, not `Out-File` on formatted objects when later parsing is expected.

## Z. cmd `for /f` is not Unix `read`

`for /f`:

- skips blank lines;
- tokenizes by delimiters by default;
- supports `eol=`, `skip=`, `tokens=`, `delims=`, `usebackq`;
- can execute a child command and parse its captured output;
- has interactive `%A` vs batch `%%A` syntax.

Therefore always write the parsing contract explicitly.

For entire nonblank lines:

```bat
for /f "usebackq delims=" %L in ("file.txt") do @echo(%L
```

If blank lines themselves are data, cmd's `for /f` may be the wrong abstraction.

## AA. cmd output minimization

Prefer machine-ish switches:

- `dir /b` over normal `dir`;
- `findstr /m` when only filenames matter;
- `find /c` when only a count matters;
- `where` instead of recursively inspecting PATH by hand;
- `forfiles` metadata variables rather than parsing `dir` date/size columns.

## AB. Cross-boundary minimization

Every shell/OS interop boundary adds quoting, encoding, path, and type conversion cost.

Examples:

- WSL Linux command ↔ Windows `.exe`;
- MSYS2 path ↔ native Windows path;
- Cygwin path ↔ native Windows path;
- PowerShell object ↔ native text CLI;
- JSON object ↔ formatted display text.

Cross once, deliberately, then remain in the new representation until necessary to return.

## AC. Do not shell-golf away observability

A good agent command should make failure diagnosable.

Avoid collapsing meaningful failures through unconditional suppression:

```sh
... 2>/dev/null || :
```

unless both suppressed conditions are known and irrelevant.

Similarly avoid blanket PowerShell `-ErrorAction SilentlyContinue` or cmd `2>nul` when errors would distinguish “not found” from “permission denied” or “tool broken”.

## AD. Command synthesis is advanced and restricted

Shell-gei tradition includes data → command text → shell execution. For autonomous agents, treat command text as a dangerous target representation.

Preferred alternatives:

- Unix: `xargs`, `find -exec`, GNU Parallel arguments;
- PowerShell: pass objects/arguments directly, call operator `&` with separated arguments when needed;
- cmd: use `for` substitution without constructing another parser layer.

Never send untrusted repository/network/user text into `sh`, `bash`, `cmd /c`, `Invoke-Expression`, or equivalent as source code.

## AE. “One-liner” means one dataflow, not one physical line

Readability may require line breaks at pipeline boundaries.

Unix:

```sh
producer |
  normalize |
  filter |
  reduce
```

PowerShell likewise treats a continuous pipeline as a one-liner conceptually even when broken across physical lines.

Optimize the number of conceptual stages, not line count.

## AF. Escalation test

Stay in shell when the task is mostly:

- search;
- selection;
- projection;
- grouping;
- sorting;
- joining;
- counting;
- reshaping;
- batch dispatch;
- shallow format inspection.

Promote to maintained code when most complexity is:

- nested mutable state;
- recursive structures;
- schema/error machinery;
- cryptographic/security-sensitive parsing;
- transactional mutation;
- reusable business logic.

The shell-gei move is knowing how far to push composition—and where to stop.
