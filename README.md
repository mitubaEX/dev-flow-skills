# dev-flow-skills

Personal [Claude Code](https://claude.ai/code) skills that codify the **software-development methodology and team-flow practices** I rely on day to day — code review triage, refactor hunting, release hygiene, post-mortem prep, and so on.

Each skill lives in its own directory under `skills/` and is shaped as a standard Claude Code skill: a single `SKILL.md` with YAML frontmatter (`name`, `description`).

## Skills

| skill | purpose | invocation |
|---|---|---|
| `smelly-prs` | Rank recently merged PRs of a GitHub repo by code-smell / process-smell heuristics. Read-only; never comments. | `/smelly-prs <owner/repo> [--limit N] [--since YYYY-MM-DD] [--base BRANCH]` |
| `layered-review` | Review a single PR or local diff with a disciplined three-layer top-down pass: production impact → chunk-level meaning (SOLID/DRY/YAGNI) → surface fit (shallow methods, foreign patterns). Read-only. | `/layered-review <PR-url\|PR-number\|--diff>` |

(More skills will land here as the workflow grows — e.g. release-readiness audits, refactor-candidate finders, on-call playbook helpers.)

## Install

Two options. Pick one.

### A. Plugin install (recommended once published)

```sh
# inside Claude Code
/plugin add github:mitubaEX/dev-flow-skills
```

### B. Symlink (local, no plugin marketplace)

```sh
git clone https://github.com/mitubaEX/dev-flow-skills.git ~/src/dev-flow-skills
mkdir -p ~/.claude/skills
ln -s ~/src/dev-flow-skills/skills/smelly-prs ~/.claude/skills/smelly-prs
```

Restart Claude Code (or start a new session) and the skill becomes available.

## Requirements

- [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth status` must show a logged-in account).

## License

MIT.
