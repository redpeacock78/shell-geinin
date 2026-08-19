# shell-geinin

An agent skill for doing shell work that survives contact with real repositories.

`shell-geinin` helps Claude Code and Codex CLI choose commands that fit the
active shell, keep data in the shell's native pipeline model, reduce output at
the producer, and stop when a maintained tool is the better abstraction.

## Install

The quickest path is the Vercel Labs Skills CLI:

```sh
npx skills add redpeacock78/shell-geinin
```

For a reproducible, non-interactive install of the `shell-geinin` skill into
both supported agents, use:

```sh
npx skills add redpeacock78/shell-geinin \
  --skill shell-geinin \
  --agent claude-code codex \
  --global \
  --yes
```

Omit `--global` for a project-scoped install. Use `--copy` when symlinks are
not appropriate for your environment.

Inspect the repository before installing:

```sh
npx skills add redpeacock78/shell-geinin --list
```

The CLI discovers `skills/shell-geinin/SKILL.md` automatically. Its default
installation is symlink-based, so updates remain easy to pull; `--copy` makes
an independent copy instead.

## What it covers

- shell and platform detection across POSIX shells, macOS/BSD, BusyBox,
  PowerShell, Windows command prompt, WSL, MSYS2, and Cygwin
- producer-side filtering, projection, and bounded output
- safe handling of filenames, quoting, ordering, whitespace, and untrusted
  input
- Git and repository inspection with compact, reviewable commands
- a deliberate escape hatch to `awk`, `sed`, `jq`, `go`, or another maintained
  tool when a shell pipeline becomes the wrong abstraction

## Use it

After installation, invoke the skill through the host agent:

- Claude Code: `/shell-geinin`
- Codex CLI: `$shell-geinin`

The source entrypoint is [`skills/shell-geinin/SKILL.md`](skills/shell-geinin/SKILL.md).
The longer platform and technique notes live under
[`skills/shell-geinin/references/`](skills/shell-geinin/references/).

## Repository layout

```text
skills/shell-geinin/SKILL.md       # Agent Skills entrypoint
skills/shell-geinin/references/    # Detailed, on-demand guidance
.claude-plugin/plugin.json         # Claude Code plugin metadata
.codex-plugin/plugin.json          # Codex plugin metadata
```

Keeping the skill in the standard `skills/<name>/SKILL.md` layout lets the
same repository work with the Skills CLI while retaining native plugin
metadata for Claude Code and Codex CLI workflows.

## Develop and verify

```sh
git clone https://github.com/redpeacock78/shell-geinin.git
cd shell-geinin
npx skills add . --list
git diff --check
```

Validate plugin metadata when working on the manifests:

```sh
claude plugin validate . --strict
jq empty .claude-plugin/plugin.json .codex-plugin/plugin.json
```

## References

- [Vercel Labs Skills CLI](https://github.com/vercel-labs/skills)
- [Comamoca's repository workflow](https://scrapbox.io/comamoca/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E4%BD%9C%E6%88%90%E3%83%BB%E5%85%AC%E9%96%8B%E6%96%B9%E6%B3%95)
- [Comamoca/baserepo](https://github.com/Comamoca/baserepo)

## License

[MIT](LICENSE)
