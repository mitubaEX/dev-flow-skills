# dev-flow-skills

Personal [Claude Code](https://claude.ai/code) skills that codify the **software-development methodology and team-flow practices** I rely on day to day — code review triage, refactor hunting, release hygiene, post-mortem prep, and so on.

Each skill lives in its own directory under `skills/` and is shaped as a standard Claude Code skill: a single `SKILL.md` with YAML frontmatter (`name`, `description`).

## Skills

| skill | purpose | invocation |
|---|---|---|
| `smelly-prs` | Rank recently merged PRs of a GitHub repo by code-smell / process-smell heuristics. Read-only; never comments. | `/smelly-prs <owner/repo> [--limit N] [--since YYYY-MM-DD] [--base BRANCH]` |
| `layered-review` | Review a single PR or local diff with a disciplined three-layer top-down pass: production impact → chunk-level meaning (SOLID/DRY/YAGNI) → surface fit (shallow methods, foreign patterns). Read-only. | `/layered-review <PR-url\|PR-number\|--diff>` |
| `grill-me` | Relentless one-question-at-a-time interview to sharpen a plan or design before writing any code. Explicit-invocation wrapper around `grilling`. Vendored from [mattpocock/skills](https://github.com/mattpocock/skills). | `/grill-me <rough plan or idea>` |
| `grilling` | The interview engine behind `grill-me`; also auto-triggers on "grill" phrasing when you want your thinking stress-tested. | `/grilling` or automatic |
| `prune` | Strip a feature, spec, UI, plan, diff, or PR down to what the request and its real user need. Four lenses — product scope, user experience, code design, delivery — each unit labelled required / load-bearing / speculative / ornamental, then a cut list and a deferred list. Also a silent stance while building. Read-only. | `/prune [--diff \| <PR-url\|PR-number> \| <spec / plan text>]` or automatic on "keep it simple" / "作りこみすぎ" / "削ぎ落として" phrasing |
| `worth-it` | Gate before implementing: separates the problem from the requested solution, checks whether it is observed or imagined, runs the do-nothing / existing-means / smallest-move tests, and returns one verdict — Don't / Already covered / Smaller / Later / Build — with what would flip it. Read-only; never implements. Upstream of `prune` and `grill-me`. | `/worth-it [<task \| ticket \| issue-url \| plan text>]` or automatic on "本当に必要?" / "is this worth doing?" phrasing |
| `autoloop` | Unattended dev loop over a backlog file (`.autoloop/backlog.md`): each iteration picks one task, gates it with `worth-it`, self-grills the design, trims with `prune`, implements red → green on an `autoloop/<slug>` branch, self-reviews with `layered-review`, and opens a PR. Never merges; tasks it should not or cannot do are marked skipped `[-]` or blocked `[!]` with the reason. Stops on an empty backlog, 5 tasks, or 2 consecutive failures. | `/loop /autoloop` (or `/autoloop [backlog-path]` for one iteration) |

(More skills will land here as the workflow grows — e.g. release-readiness audits, refactor-candidate finders, on-call playbook helpers.)

## Running the autonomous loop

`autoloop` chains the other skills into a loop that needs no human input between tasks.

```sh
echo .autoloop/ >> .git/info/exclude
mkdir -p .autoloop && cat > .autoloop/backlog.md <<'MD'
- [ ] Add a header row to the CSV export
- [ ] Show the retry count in the sync error toast
MD
```

Let `git`, `gh pr create`, and your test runner run without prompts (auto mode or `permissions.allow` in `.claude/settings.json`), then start it inside Claude Code with `/loop /autoloop`. Progress is logged to `.autoloop/log.md`. Your part: add `[ ]` tasks, answer `[!]` questions (then flip them back to `[ ]`), merge the PRs.

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
