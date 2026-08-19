<div align="center">

# shell-geinin

Cross-platform shell reasoning for Claude Code and Codex CLI.

[![License: MIT](https://img.shields.io/github/license/redpeacock78/shell-geinin?style=flat-square)](LICENSE)
[![CI](https://github.com/redpeacock78/shell-geinin/actions/workflows/ci.yml/badge.svg)](https://github.com/redpeacock78/shell-geinin/actions/workflows/ci.yml)
[![Install with Skills CLI](https://img.shields.io/badge/install-npx%20skills-111827?style=flat-square)](https://github.com/vercel-labs/skills)

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

### Codex CLI: warm-session measurement

The measurements above are a shell and context proxy. Codex can expose a closer per-turn measurement through `codex exec --json`: the CLI emits newline-delimited JSON events, and `codex exec resume` continues the same session. See the [official Codex CLI command reference](https://learn.chatgpt.com/docs/developer-commands?surface=cli).

We used one initial turn to warm each condition, then compared the next two turns in the same session. The result below is an exploratory snapshot, not a claim that every task or model will save tokens.

- Environment: macOS/zsh, Codex CLI `0.148.0`, read-only sandbox, reasoning effort `low`, and the same locally configured model for both arms.
- Fixture: a deterministic 120,000-line, 11.4 MB log and a 50,000-record, 5.5 MB NDJSON file.
- Baseline: `shell-geinin` was explicitly disabled; inline Python was allowed.
- Treatment: `shell-geinin` was explicitly used and retained between turns.
- `input`, `cached input`, `output`, and `reasoning` come from each `turn.completed.usage` event.
- `command chars` and `stdout bytes` are local diagnostics from completed command events, not billing metrics.
- Both arms returned the same verified counts and selected records for the two warm tasks.

Warm turns only (`N=1` session per arm):

| Task | Arm | Input | Cached input | Output | Reasoning | Tool calls | Command chars | Stdout bytes |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| NDJSON filter + projection | baseline | 62,061 | 40,448 | 368 | 108 | 1 | 419 | 237 |
| NDJSON filter + projection | shell-geinin | 65,913 | 41,472 | 787 | 527 | 1 | 225 | 256 |
| Full-log grouped count | baseline | 76,316 | 47,616 | 373 | 161 | 1 | 410 | 63 |
| Full-log grouped count | shell-geinin | 105,934 | 101,632 | 1,042 | 538 | 2 | 859 | 82 |
| Warm aggregate | baseline | 138,377 | 88,064 | 741 | 269 | 2 | 829 | 300 |
| Warm aggregate | shell-geinin | 171,847 | 143,104 | 1,829 | 1,065 | 3 | 1,084 | 338 |

For this snapshot, the raw sum `input + output + reasoning` increased from 139,387 to 174,741 (+25.4%) with `shell-geinin`. The simple uncached-side sum `(input - cached input) + output + reasoning` decreased from 51,323 to 31,637 (-38.4%), but that is not a billing total. Cached-input and reasoning accounting depend on the Codex/model configuration, so the two figures must not be collapsed into one “savings” number.

The narrower conclusion is more useful: the Skill steered the warm tasks toward native `jq`/`awk` processing and bounded data output, but agent-generated output and reasoning still dominated this run. At least five sessions per arm, with cold and warm turns reported separately, are needed before making a general efficiency claim. Live Codex measurements belong in a manual or scheduled benchmark rather than ordinary pull-request CI because model behavior, authentication, and usage limits are external variables.

The raw usage can be collected with the CLI and reduced with `jq`:

```sh
codex exec --json --sandbox read-only 'warm-up task' > cold.jsonl
codex exec resume --json "$THREAD_ID" 'measurement task' > warm.jsonl
jq -c 'select(.type == "turn.completed") | .usage' warm.jsonl
```

This is the Codex-native equivalent of the article's local usage dashboard: JSONL is the measurement source, while SQLite/FastAPI/OTLP would be optional visualization layers.

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
