<div align="center">

# shell-geinin

Cross-platform shell reasoning for Claude Code and Codex CLI.

[![License: MIT](https://img.shields.io/github/license/redpeacock78/shell-geinin?style=flat-square)](LICENSE)
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
jq -c 'limit(50; select(.status == "error" and .service == "api") | {id,status,service})' records-100k.ndjson

git -C history-repo log --format=fuller --stat
git -C history-repo log -50 --format='%h %s' --no-decorate
```

| Scenario | Command tokens | Output tokens | Total tokens | Reduction |
| --- | ---: | ---: | ---: | ---: |
| 200k-line log, all matching rows → first 50 | 21 → 25 | 1,846,421 → 2,257 | 1,846,442 → 2,282 | 99.9% |
| 100k NDJSON, all records → matching `error/api` projections, first 50 | 10 → 36 | 4,599,001 → 16,453 | 4,599,011 → 16,489 | 99.6% |
| 1,000 commits, full detail → latest 50 one-line summaries | 14 → 21 | 93,562 → 667 | 93,576 → 688 | 99.3% |

The `cl100k_base` cross-check rounded to the same reductions: 99.9%, 99.6%, and 99.3%.
The result is therefore primarily driven by output cardinality and field width, not by a tokenizer-specific quirk.
These percentages are measurements on controlled fixtures, not a promise of the same saving for every repository or query.

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

Validate plugin metadata when changing the manifests:

```sh
claude plugin validate . --strict
jq empty .claude-plugin/plugin.json .codex-plugin/plugin.json
```

## 📚 References

- [Vercel Labs Skills CLI](https://github.com/vercel-labs/skills)
- [Comamoca's repository workflow](https://scrapbox.io/comamoca/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E4%BD%9C%E6%88%90%E3%83%BB%E5%85%AC%E9%96%8B%E6%96%B9%E6%B3%95)
- [Comamoca/baserepo](https://github.com/Comamoca/baserepo)

## 📜 License

[MIT](LICENSE)
