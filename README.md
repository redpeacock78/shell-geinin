# shell-geinin

Cross-platform shell-gei as an Agent Skill for Claude Code and Codex CLI.

It teaches an agent to inspect the active shell, preserve its native pipeline data model, reduce data at the producer, and choose the shortest clear and safe command composition.

## Contents

- `skills/shell-geinin/SKILL.md` — the skill entrypoint
- `skills/shell-geinin/references/` — platform notes, technique reference, research notes, and maintainer evaluations
- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `.codex-plugin/plugin.json` — Codex plugin manifest

The main `SKILL.md` stays under 500 lines. Detailed material is loaded from `references/` only when needed.

## Use with Claude Code

Test the plugin directly from this checkout:

```sh
claude --plugin-dir .
```

Invoke it as `/shell-geinin:shell-geinin`.

For a project-scoped standalone skill, copy `skills/shell-geinin/` to `.claude/skills/shell-geinin/` in that project.

## Use with Codex CLI

For a repository-scoped standalone skill, copy `skills/shell-geinin/` to `.agents/skills/shell-geinin/` in the target repository. For a reusable installation, use Codex's `$skill-installer` with this skill directory after the repository is published.

The `.codex-plugin/` manifest makes this checkout a skill-only Codex plugin for use through a Codex plugin marketplace.

## Repository workflow

This checkout follows the practical flow described in [Comamoca's repository creation guide](https://scrapbox.io/comamoca/%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E3%81%AE%E4%BD%9C%E6%88%90%E3%83%BB%E5%85%AC%E9%96%8B%E6%96%B9%E6%B3%95): create the local repository, write and verify the contents, then explicitly create and push the remote repository.

For this already-existing directory, the equivalent publication command is:

```sh
gh repo create shell-geinin --public --source=. --remote=origin --push
```

Review the target owner, visibility, and pending commits before running it. GitHub topics are remote metadata and must be added after the repository exists.

## License

MIT. See [LICENSE](LICENSE).
