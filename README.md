<div align="center">

# shell-geinin

Cross-platform shell reasoning for Claude Code and Codex CLI.

[![License: MIT](https://img.shields.io/github/license/redpeacock78/shell-geinin?style=flat-square)](LICENSE)
[![CI](https://github.com/redpeacock78/shell-geinin/actions/workflows/ci.yml/badge.svg)](https://github.com/redpeacock78/shell-geinin/actions/workflows/ci.yml)
[![Install with Skills CLI](https://img.shields.io/badge/install-npx%20skills-111827?style=flat-square)](https://github.com/vercel-labs/skills)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/redpeacock78/shell-geinin)

</div>

[🍔 English](README.md) | [🍡 日本語](README.ja.md)

> A reusable Agent Skill for choosing clear, safe shell commands that fit the active shell and its native data model.

## 🚀 Quick start

```sh
npx skills add redpeacock78/shell-geinin
```

Then let the host agent use `shell-geinin` for repository investigation,
file search, Git inspection, structured data reduction, and command design.

Invoke it from the host agent with:

- Claude Code: `/shell-geinin`
- Codex CLI: `$shell-geinin`

## ⬇️ Install

Install the `shell-geinin` skill for both Claude Code and Codex CLI:

```sh
npx skills add redpeacock78/shell-geinin \
  --skill shell-geinin \
  --agent claude-code codex \
  --global \
  --yes
```

Omit `--global` for a project-scoped install. Use `--copy` when a standalone
copy is preferable to the CLI's default symlink.

> [!TIP]
> Use `npx skills add redpeacock78/shell-geinin --list` to inspect the skills
> exposed by this repository before installing them.

## 🧠 What it teaches

- detect the active shell instead of guessing from the operating system
- preserve text streams in POSIX shells and objects in PowerShell
- filter, project, group, and bound data at the producer
- treat filenames, quoting, ordering, whitespace, and untrusted input as correctness concerns
- move to a maintained tool such as `awk`, `sed`, `jq`, `git`, or `gh` when a shell pipeline is the wrong abstraction

## 🌐 Supported environments

| Environment | Guidance |
| --- | --- |
| POSIX shells, macOS/BSD, Linux, and BusyBox | Portable text-stream pipelines |
| PowerShell | Object-native pipelines and property selection |
| Windows `cmd.exe` | Simple text pipelines with explicit quoting |
| WSL, MSYS2, Cygwin, and Git Bash | Unix-style pipelines with path-boundary care |

## 📊 Measured impact

This benchmark measures the amount of command text and standard output that an agent needs to carry into context.

It does not measure hidden model generation, system prompts, tool-wrapper tokens, stderr, or billing tokens.

The intent is to separate savings from shorter syntax from savings caused by producer-side bounds and field projection.

### Method

- `o200k_base` is the primary tokenizer, with `cl100k_base` used as a cross-check.
- Total tokens are command-string tokens plus stdout tokens.
- Inputs include this repository's file inventory, a 5,000-line log, a 120-commit fixture repository, and 1,000 JSON records.
- The extended run uses controlled fixtures: a 200,000-line log, 100,000 NDJSON records, and a 1,000-commit history.
- The baseline collects broad output, while the shell-geinin form limits at the producer or projects only the required fields.
- The inventory and bounded-search rows returned byte-for-byte identical output.

Representative comparisons:

```sh
grep -n 'shell' events.log
rg -n -m 25 'shell' events.log

git log --all --stat --oneline
git log -20 --oneline --no-decorate

jq -c '.[]' items.json
jq -c '.[] | {id,status}' items.json
```

| Scenario | Command tokens | Output tokens | Total tokens | Reduction |
| --- | ---: | ---: | ---: | ---: |
| File inventory, same output | 26 → 14 | 87 → 87 | 113 → 101 | 10.6% |
| Search, first 25 results, same output | 12 → 12 | 517 → 517 | 529 → 529 | 0.0% |
| Search, all matches → first 25 | 8 → 12 | 107,335 → 517 | 107,343 → 529 | 99.5% |
| Git history, full stats → latest 20 summaries | 9 → 12 | 1,122 → 184 | 1,131 → 196 | 82.7% |
| JSON records, all fields → `id,status` | 8 → 12 | 40,923 → 9,000 | 40,931 → 9,012 | 78.0% |

The largest savings come from bounding output and projecting fields, not from shortening command syntax alone.

The same-output search row shows 0.0% token reduction because both commands return the same 25 lines.
The producer-bounded form can still reduce processing work, but runtime and subprocess savings are outside this table.

### Larger and more complex cases

These cases deliberately compare a broad default request with a task-shaped request.
They are not byte-for-byte equivalent outputs; the same-output comparison above is the appropriate reference when output equivalence is required.

The fixtures were 27.1 MB of 200,000 log lines, 16.9 MB of 100,000 NDJSON records, and a 1,000-commit Git history.
The primary `o200k_base` results were:

```sh
rg -n 'status=(ERROR|WARN) service=(api|worker)' events-200k.log
rg -n -m 50 'status=(ERROR|WARN) service=(api|worker)' events-200k.log

jq -c '.' records-100k.ndjson
jq -c -n 'limit(50; inputs | select(.status == "error" and .service == "api") | {id,status,service})' records-100k.ndjson

git -C history-repo log --format=fuller --stat
git -C history-repo log -50 --format='%h %s' --no-decorate
```

| Scenario | Command tokens | Output tokens | Total tokens | Reduction |
| --- | ---: | ---: | ---: | ---: |
| 200k-line log, all matching rows → first 50 | 21 → 25 | 1,846,421 → 2,257 | 1,846,442 → 2,282 | 99.9% |
| 100k NDJSON, all records → matching `error/api` projections, first 50 | 10 → 40 | 4,599,001 → 689 | 4,599,011 → 729 | 99.98% |
| 1,000 commits, full detail → latest 50 one-line summaries | 14 → 21 | 93,562 → 667 | 93,576 → 688 | 99.3% |

The `cl100k_base` cross-check rounded to one decimal place as 99.9%, 100.0%, and 99.3%.
The result is therefore primarily driven by output cardinality and field width, not by a tokenizer-specific quirk.
These percentages are measurements on controlled fixtures, not a promise of the same saving for every repository or query.

### Inline Python versus shell-geinin

Large and structured inputs often tempt an agent to write a disposable `python3 -c` program.
This comparison keeps the task and stdout identical, then measures only the command text and that shared output.
All three pairs produced byte-for-byte identical stdout.

The `o200k_base` results are shown as `command tokens + output tokens = total tokens`:

| Scenario | Inline Python | shell-geinin form | Total reduction |
| --- | ---: | ---: | ---: |
| 200k-line log, bounded multi-condition search | 73 + 2,257 = 2,330 | 25 + 2,257 = 2,282 | 2.1% |
| 100k NDJSON, filter + projection + first 50 | 89 + 689 = 778 | 40 + 689 = 729 | 6.3% |
| 200k-line log, grouped counts | 89 + 10 = 99 | 62 + 10 = 72 | 27.3% |

Representative command pairs:

```sh
python3 -c 'import re,itertools; p=re.compile(r"status=(ERROR|WARN) service=(api|worker)"); [print(f"{n}:{line}",end="") for n,line in itertools.islice(((n,line) for n,line in enumerate(open("events-200k.log"),1) if p.search(line)),50)]'
rg -n -m 50 'status=(ERROR|WARN) service=(api|worker)' events-200k.log

python3 -c 'import json,itertools; records=(json.loads(line) for line in open("records-100k.ndjson")); selected=(r for r in records if r["status"]=="error" and r["service"]=="api"); [print(json.dumps({key:r[key] for key in ("id","status","service")},separators=(",",":"))) for r in itertools.islice(selected,50)]'
jq -c -n 'limit(50; inputs | select(.status == "error" and .service == "api") | {id,status,service})' records-100k.ndjson

python3 -c 'from collections import Counter; counts=Counter(line.split()[2].split("=")[1] for line in open("events-200k.log") if line.split()[1].split("=")[1] in ("ERROR","WARN") and line.split()[2].split("=")[1] in ("api","worker")); print("\n".join(f"{key} {counts[key]}" for key in sorted(counts)))'
awk '$2 ~ /^status=(ERROR|WARN)$/ && $3 ~ /^service=(api|worker)$/ { sub(/^service=/, "", $3); count[$3]++ } END { for (service in count) print service, count[service] }' events-200k.log | sort
```

The comparison does not show that Python is always the wrong choice.
When both forms return thousands of rows, the shared output dominates and syntax savings are small.
When the result is a compact aggregate, command text accounts for a larger share and the shell form saves more.
If the shell composition becomes harder to read or maintain, a tested script is the documented escalation path.

### Codex CLI: controlled A/B measurement

The shell measurements above are context proxies. Codex provides a closer per-turn source through `codex exec --json`: it emits newline-delimited JSON events, and `codex exec resume` continues the same session. See the [official Codex CLI command reference](https://learn.chatgpt.com/docs/developer-commands?surface=cli).

The earlier one-session snapshot is superseded here. Its baseline was not given a per-turn control instruction, so Codex could load the installed Skill by itself. The corrected run below verifies that baseline command events contain no `SKILL.md` reference.

#### Design

- `M=5` independent sessions per arm and `N=10` identical tasks per session.
- Baseline starts with the first real task and has no warm-up.
- Treatment has one Skill-only warm-up, followed by the same ten tasks.
- Both arms use macOS/zsh, Codex CLI `0.148.0`, read-only sandbox, reasoning effort `low`, and the same locally configured model.
- The fixture is deterministic: a 120,000-line, 11.4 MB log and a 50,000-record, 5.5 MB NDJSON file.
- Each task is read-only, uses the same task sequence, and reports compact counts or selected records.
- `M` measures variation between independent sessions. `N` controls how the treatment warm-up is amortized.

The baseline control instruction forbids reading or invoking any Agent Skill and allows inline Python. The treatment warm-up explicitly reads the complete `shell-geinin` Skill and retains it in the resumed session. This instruction is part of the harness, not a claim about how a normal user should phrase a task.

The primary steady-state comparison excludes the treatment warm-up. The amortized comparison divides `(treatment warm-up + N treatment tasks)` by `N`; baseline has no artificial warm-up added.

- `input` and `cached input` are the local `turn.completed.usage` fields.
- `uncached input` is `input - cached input`.
- `command chars` and `stdout bytes` come from completed command events.
- `shell I/O proxy` is `command chars + stdout bytes`; it is not a tokenizer result or a billing metric.
- `output` and `reasoning` are reported as secondary diagnostics and are excluded from the primary proxy comparison.

#### Results

Steady state, ten measured tasks per session, averaged over 50 turns. Parentheses contain the median of the five session means.

| Metric per task | Baseline | shell-geinin | Mean delta |
| --- | ---: | ---: | ---: |
| Input tokens | 121,986 (119,854) | 129,494 (125,492) | +6.2% |
| Uncached input tokens | 10,493 (10,695) | 6,977 (5,121) | -33.5% |
| Command chars | 549 (544) | 517 (476) | -5.9% |
| Stdout bytes | 21,300 (343) | 261 (273) | -98.8% |
| Shell I/O proxy | 21,849 (930) | 778 (733) | -96.4% |
| Output tokens, secondary | 535 (541) | 615 (544) | +15.0% |
| Reasoning tokens, secondary | 97 (95) | 213 (169) | +121.0% |

One baseline task emitted 1,049,466 stdout bytes from an unbounded intermediate command while still returning the expected final aggregate. The means include this event; the session medians show the typical session more clearly.

A response-quality check found 49/50 successful baseline task responses and 49/50 successful treatment task responses.
The two failures were baseline session 3 task 1, which returned four zero counts, and treatment session 4 task 1, which returned no counts after incorrectly claiming a nonstandard fixture.
Those turns remain in the token aggregates, so the tables measure attempted work and are not quality-adjusted savings.

The warm-up itself averaged 119,673 input tokens, 22,137 uncached input tokens, and 20,073 shell I/O bytes. Including that warm-up and dividing by `N=10` gives the following session-level average:

| Metric per task after session amortization | Baseline `N/N` | shell-geinin `(warm-up + N)/N` | Mean delta |
| --- | ---: | ---: | ---: |
| Input tokens | 121,986 | 141,461 | +16.0% |
| Uncached input tokens | 10,493 | 9,191 | -12.4% |
| Shell I/O proxy | 21,849 | 2,785 | -87.3% |

The amortized shell I/O mean is dominated by the baseline outlier. The median amortized shell I/O was 930 for baseline and 2,741 for treatment, so the warm-up did not pay back in every typical ten-task session.

The N curve shows why both estimands are needed:

| N | Steady uncached input, baseline | Steady uncached input, shell-geinin | Amortized shell I/O, baseline | Amortized shell I/O, shell-geinin |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 23,497 | 8,914 | 2,114 | 21,856 |
| 2 | 16,406 | 6,543 | 1,375 | 11,076 |
| 5 | 11,966 | 8,017 | 43,397 | 5,247 |
| 10 | 10,493 | 6,977 | 21,849 | 2,785 |

This is a controlled A/B result for this fixture, CLI version, model configuration, and task sequence. It supports a narrower claim: shell-geinin reduced uncached input and bounded tool output in the measured steady state. It does not support a universal savings percentage, because total input, model output, reasoning, and warm-up payback depend on the workload.

The same measurement can be reproduced with one session per arm:

```sh
# Baseline: first turn is already a real task; no warm-up.
codex exec --json --sandbox read-only \
  'CONTROL ARM. Do not read or invoke any Agent Skill or SKILL.md. Run the measurement task.' \
  </dev/null \
  > baseline-task01.jsonl

# Treatment: warm up only the Skill, then resume with the same task.
codex exec --json --sandbox read-only \
  'Read the complete shell-geinin SKILL.md only as a warm-up. Do not inspect fixture data.' \
  </dev/null \
  > treatment-warmup.jsonl
THREAD_ID=$(jq -r 'select(.type == "thread.started") | .thread_id' treatment-warmup.jsonl | head -1)
codex exec resume --json "$THREAD_ID" \
  'Run the same measurement task.' </dev/null > treatment-task01.jsonl

jq -c 'select(.type == "turn.completed") | .usage' treatment-task01.jsonl
jq -c 'select(.type == "item.completed" and .item.type == "command_execution") | .item' treatment-task01.jsonl
```

This is the Codex-native equivalent of the article's local usage dashboard. JSONL is the measurement source; SQLite, FastAPI, and OTLP remain optional visualization layers.

## 📁 Repository layout

```text
skills/shell-geinin/SKILL.md       # Agent Skills entrypoint
skills/shell-geinin/references/    # Detailed, on-demand guidance
.claude-plugin/plugin.json         # Claude Code plugin metadata
.codex-plugin/plugin.json          # Codex plugin metadata
README.ja.md                       # Japanese documentation
```

The standard `skills/<name>/SKILL.md` layout makes the repository discoverable
by the Skills CLI while the plugin manifests keep native Claude Code and Codex
CLI workflows available.

## 🛠️ Development

```sh
git clone https://github.com/redpeacock78/shell-geinin.git
cd shell-geinin
npx skills add . --list
git diff --check
```

Every push and pull request runs the lightweight package validation in
`.github/workflows/ci.yml`.
It checks the JSON manifests, Skill layout and frontmatter, documented reference files, and whitespace errors.

Validate plugin metadata when changing the manifests:

```sh
claude plugin validate . --strict
jq empty .claude-plugin/plugin.json .codex-plugin/plugin.json
```

## 📚 References

- [Vercel Labs Skills CLI](https://github.com/vercel-labs/skills)
- [OpenAI Codex CLI developer commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
- [Comamoca's repository workflow](https://scrapbox.io/comamoca/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E4%BD%9C%E6%88%90%E3%83%BB%E5%85%AC%E9%96%8B%E6%96%B9%E6%B3%95)
- [Comamoca/baserepo](https://github.com/Comamoca/baserepo)

## 📜 License

[MIT](LICENSE)
