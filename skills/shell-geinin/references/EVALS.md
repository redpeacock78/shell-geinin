# Shell-Geinin Cross-Platform Evaluation Set

These are behavior-level evaluations for the `shell-geinin` skill. They are not normal runtime instructions. Use them when changing the skill to check whether the agent preserves shell-gei reasoning across environments instead of memorizing one platform's command spellings.

## Evaluation philosophy

A strong answer should generally:

1. detect or respect the stated active shell;
2. use producer-side filtering/projection when available;
3. preserve the native pipeline data model;
4. avoid throwaway general-purpose runtime scripts;
5. avoid assuming GNU-only, Bash-only, or PowerShell-version-specific features without evidence;
6. bound output before returning it to the model;
7. avoid repeated expensive producers;
8. handle filenames/data safely;
9. cross shell/runtime boundaries only when there is a clear semantic win;
10. escalate to maintained code when shell composition becomes opaque.

The exact command is not the test. The reasoning shape is.

---

## E1 — Unknown Unix repository search

**Environment:** unknown POSIX-like shell. GNU extensions are not guaranteed.

**Task:** Find at most 30 files under the current tree containing `FIXME`, excluding `.git`, and show matching line numbers.

**Strong behavior:**

- probes `rg` first;
- if available, uses `rg` with an output bound;
- otherwise uses portable/common `grep`/`find` without Bash-only process substitution;
- does not `cat` all files or invoke Python.

**Failure signatures:** full-tree dump, Python walk, unconditional GNU `find -printf`, `for f in $(find ...)`.

---

## E2 — GNU/Linux producer-side filtering

**Environment:** GNU/Linux with GNU coreutils/findutils and `jq`.

**Task:** List the 20 largest regular `.json` files under `data/` with size and path.

**Strong behavior:** lets `find` select regular `.json` files, emits only required fields, numeric-sorts once, and takes 20 rows. It should use NUL-safe processing if filenames are carried through an unsafe textual stage.

**Failure signatures:** Python `os.walk`, `ls -R`, repeated `du` per file when `find` already exposes size, reading JSON contents.

---

## E3 — BSD/macOS portability

**Environment:** macOS/BSD userland. GNU-prefixed tools may or may not be installed.

**Task:** Replace `foo` with `bar` in a temporary test file and verify the result.

**Strong behavior:** does not blindly assume GNU `sed -i`; probes `gsed` if a GNU-specific solution is useful, or uses a portable temporary-file/writeback pattern. Mutation is verified.

**Failure signatures:** `sed -i` with GNU semantics assumed, accidental zero-byte file, silent mutation without check.

---

## E4 — BusyBox container

**Environment:** minimal container with BusyBox and `/bin/sh`. No Python, `rg`, `jq`, or GNU `parallel` guaranteed.

**Task:** Count distinct first fields in a whitespace-delimited log and print the top 10 counts.

**Strong behavior:** uses common applets such as `awk`, `sort`, `uniq`, `head`; inspects applet help only if an uncertain option is needed; stays within the minimal environment.

**Failure signatures:** long-option assumptions, Bash arrays, process substitution, package installation just to solve the task.

---

## E5 — PowerShell object pipeline

**Environment:** PowerShell 7 on Windows.

**Task:** Show the 15 largest `.log` files under the current directory with full path, length, and last-write time.

**Strong behavior:** uses `Get-ChildItem` producer filtering where possible, keeps filesystem objects in the pipeline, sorts by `Length`, projects properties with `Select-Object`, and formats only at the final display edge if formatting is needed.

**Failure signatures:** `Get-ChildItem | Format-Table | Select-String`, parsing `dir` display text, spawning Python, converting objects to CSV/text mid-pipeline for no reason.

---

## E6 — PowerShell structure discovery

**Environment:** PowerShell 5.1 or newer.

**Task:** From an unfamiliar cmdlet's output, filter by a property but the property names are not known yet.

**Strong behavior:** uses `Get-Command`/`Get-Help`/`Get-Member` to discover structure before writing a parser. Does not assume PowerShell 7-only features.

**Failure signatures:** `Out-String | Select-String` against formatted output, guessing property names repeatedly.

---

## E7 — Native executable inside PowerShell

**Environment:** PowerShell with `git` and `jq` installed.

**Task:** Summarize selected JSON emitted by a native CLI.

**Strong behavior:** recognizes the native boundary. If the CLI emits JSON, either sends it directly to `jq` as text or converts once with `ConvertFrom-Json` and then remains object-native. It does not bounce repeatedly between object and text representations.

**Failure signatures:** `Format-Table` before parsing, repeated JSON serialize/deserialize cycles, shell-hopping without benefit.

---

## E8 — cmd.exe sweet spot

**Environment:** classic `cmd.exe`; PowerShell exists but the task is simple.

**Task:** Recursively list paths of `.log` files that contain `ERROR`, without printing matching lines.

**Strong behavior:** uses concise Windows primitives such as `for /r` plus `findstr /m` or an equivalent simple composition. In a batch file it uses `%%F`; interactively `%F`.

**Failure signatures:** immediately launches Python; constructs a large PowerShell program for a trivial text predicate; forgets the `%F`/`%%F` distinction.

---

## E9 — cmd.exe escalation point

**Environment:** cmd.exe with PowerShell installed.

**Task:** Parse quoted CSV with commas inside quoted fields, group by two columns, sort numerically by an aggregate, and emit JSON.

**Strong behavior:** recognizes that cmd's token model is the wrong abstraction and promotes to PowerShell rather than building a quoting monument. Uses `Import-Csv`, object grouping/projection, and `ConvertTo-Json` or another concise structured route.

**Failure signatures:** nested `for /f` delimiters pretending to parse RFC-style CSV, general-purpose Python when PowerShell is the native structured shell already present.

---

## E10 — WSL boundary

**Environment:** PowerShell on Windows with WSL installed. Repository lives in the Linux filesystem.

**Task:** Search source files using Linux `rg` and return only 20 matches to PowerShell.

**Strong behavior:** performs the filtering inside WSL and crosses the WSL↔Windows boundary once with already-reduced output. It does not enumerate the full Linux tree into PowerShell first.

**Failure signatures:** per-file `wsl.exe` calls, copying the whole tree listing across the boundary, path-converting every row unnecessarily.

---

## E11 — MSYS2 path conversion

**Environment:** MSYS2 shell invoking a native Windows tool.

**Task:** Pass Unix-style paths to a native Windows command whose argument syntax contains leading slash-like tokens.

**Strong behavior:** remembers that MSYS2 may automatically transform arguments/paths, uses `cygpath` or `MSYS2_ARG_CONV_EXCL` only when needed, and avoids assuming that a path-shaped argument survives unchanged.

**Failure signatures:** unexplained mangled arguments, repeated ad-hoc string replacement of `/c/...` paths.

---

## E12 — Cross-platform JSON investigation

**Environment:** unspecified; `git` and either `jq` or PowerShell structured JSON support may exist.

**Task:** From a 100 MB JSON-producing command, answer one count grouped by a single field.

**Strong behavior:** discovers available structured operators, pushes field selection/grouping toward the source, avoids returning 100 MB to the model, and avoids a language-runtime heredoc.

**Failure signatures:** captures the entire JSON in model context, writes a temporary Python parser without checking `jq`/PowerShell capabilities.

---

## E13 — Expensive producer fan-out

**Environment:** Unix-like shell with `tee`, optionally moreutils `pee`.

**Task:** One expensive command produces a stream needed for two independent cheap summaries.

**Strong behavior:** runs the producer once and fans out with `tee`/process substitution only when the shell supports it, `pee` when available, or materializes a bounded temporary artifact when that is clearer. It does not rerun the expensive producer by default.

**Failure signatures:** identical expensive command twice, unsafe process substitution in plain POSIX `sh`.

---

## E14 — Set difference instead of procedural loop

**Environment:** POSIX-like shell.

**Task:** Given observed numeric IDs in a finite range 00–99, print missing IDs.

**Strong behavior:** considers generating the universe and applying sort/set-difference semantics rather than looping 100 times and grepping the file repeatedly.

**Failure signatures:** O(N×file-scan) loop by habit, Python set construction without checking shell primitives.

---

## E15 — Preserve order semantics

**Environment:** Unix-like shell.

**Task:** Group records by key while preserving first-observed order inside equal keys.

**Strong behavior:** notices that sorting may destroy semantic order, chooses stable sorting when actually available or tags original positions and restores them, and does not assume GNU `sort -s` if portability is unknown.

**Failure signatures:** accidental lexical tie-breaker changes meaning, undocumented locale-dependent ordering.

---

## E16 — Filenames are data

**Environment:** POSIX-like shell with files containing spaces, tabs, quotes, and newlines.

**Task:** Run a harmless inspection command once per regular file.

**Strong behavior:** uses `find -exec ... {} +` as a broadly portable safe route, or NUL-delimited tools when support is known. It never uses `for f in $(find ...)`.

**Failure signatures:** word splitting, `xargs` without safe delimiter when arbitrary filenames are possible, interpolation into `sh -c` source text.

---

## E17 — Output budget

**Environment:** any shell.

**Task:** Investigate a command whose normal output is 500,000 lines to understand only the first few errors and aggregate counts.

**Strong behavior:** uses producer filters, bounded samples, counts, or summaries. It treats output returned to the model as a budgeted resource.

**Failure signatures:** runs the verbose command unbounded, then asks the model to mentally filter it.

---

## E18 — Stop being a shell geinin at the right time

**Environment:** any shell.

**Task:** Implement recursive schema validation, retries with backoff, transactional mutation, and reusable error reporting for a long-lived workflow.

**Strong behavior:** explicitly escalates to the project's maintained language/toolchain. It does not produce a heroic 300-character one-liner simply to remain in shell.

**Failure signatures:** code-golf worship, `eval`, deeply nested quoting, untestable shell state machine.

---

## Suggested regression scorecard

Score each scenario 0–2 in these dimensions:

- environment awareness;
- native data-model preservation;
- producer-side reduction;
- representation-first reasoning;
- portability/capability awareness;
- output/context economy;
- safety and quoting;
- appropriate escalation.

A regression is especially serious when a change improves brevity while reducing correctness, safety, or environment awareness.
