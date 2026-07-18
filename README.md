# dev-flow-skills

Personal [Claude Code](https://claude.ai/code) skills that codify the **software-development methodology and team-flow practices** I rely on day to day — code review triage, refactor hunting, release hygiene, post-mortem prep, and so on.

Each skill lives in its own directory under `skills/` and is shaped as a standard Claude Code skill: a single `SKILL.md` with YAML frontmatter (`name`, `description`).

## Skills

| skill | purpose | invocation |
|---|---|---|
| `smelly-prs` | Rank recently merged PRs of a GitHub repo by code-smell / process-smell heuristics. Read-only; never comments. | `/smelly-prs <owner/repo> [--limit N] [--since YYYY-MM-DD] [--base BRANCH]` |
| `grill-me` | Relentless one-question-at-a-time interview to sharpen a plan or design before writing any code. Explicit-invocation wrapper around `grilling`. Vendored from [mattpocock/skills](https://github.com/mattpocock/skills). | `/grill-me <rough plan or idea>` |
| `grilling` | The interview engine behind `grill-me`; also auto-triggers on "grill" phrasing when you want your thinking stress-tested. | `/grilling` or automatic |

(More skills will land here as the workflow grows — e.g. release-readiness audits, refactor-candidate finders, on-call playbook helpers.)

## Install

Two options. Pick one.

### A. Plugin marketplace (recommended)

This repo is a Claude Code plugin marketplace (`.claude-plugin/marketplace.json`) exposing all skills as a single `dev-flow-skills` plugin.

```sh
# inside Claude Code
/plugin marketplace add mitubaEX/dev-flow-skills
/plugin install dev-flow-skills@dev-flow-skills
```

To pick up new skills later: `/plugin marketplace update dev-flow-skills`.

### B. Symlink (local, no plugin marketplace)

```sh
git clone https://github.com/mitubaEX/dev-flow-skills.git ~/src/dev-flow-skills
mkdir -p ~/.claude/skills
for s in ~/src/dev-flow-skills/skills/*/; do ln -s "${s%/}" ~/.claude/skills/; done
```

Restart Claude Code (or start a new session) and the skill becomes available.

## Requirements

- [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth status` must show a logged-in account).

## License

MIT.
