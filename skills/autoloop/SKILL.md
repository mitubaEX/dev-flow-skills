---
name: autoloop
description: Work through a backlog file with no human in the loop — each iteration picks the next task from `.autoloop/backlog.md`, gates it with `worth-it`, settles the design by self-grilling, trims it with `prune`, implements it red → green on an `autoloop/<slug>` branch, self-reviews with `layered-review`, and opens a PR. Never merges. Tasks it should not or cannot do are marked skipped or blocked with the reason, so the human's only jobs are adding tasks, answering blocked questions, and merging PRs. Designed to be driven by `/loop /autoloop` (self-paced; schedules its own next wakeup and stops when the backlog is drained or a guardrail trips). Use when the user wants an autonomous / unattended / hands-off dev loop, "放置で回す", "自律ループ", "バックログを消化して", or invokes `/autoloop`. Requires `git`, a pushable remote, and an authenticated `gh` CLI.
---

# autoloop

Run the dev flow of this repository — **worth-it → grilling → prune → implement → layered-review → PR** — one backlog task per iteration, with nobody answering questions.

A single `/autoloop` does **one iteration**. `/loop /autoloop` repeats it until the backlog is empty or a guardrail stops it.

The human does three things, all asynchronously:

1. add `- [ ]` lines to the backlog,
2. answer `[!]` blocked questions (edit the note, flip it back to `[ ]`),
3. review and merge the PRs.

Everything else is this skill's job. Where a neighbouring skill says "ask the user and wait", this skill **decides with the recommended answer** or **blocks the task** — it never waits for input.

---

## The backlog

Default path: `.autoloop/backlog.md` at the repository root. `/autoloop <path>` overrides it. The `.autoloop/` directory must be **ignored by git** (add `.autoloop/` to `.gitignore` or `.git/info/exclude`) so that its state survives branch switches and never lands in a commit. If it is tracked, stop and say so.

Format — one task per top-level checkbox line; indented lines under it are notes for that task:

```markdown
- [ ] Add a header row to the CSV export
  - exporter lives in src/export.ts
- [~] Improve the login error message      (autoloop/login-error-message, 2026-09-30)
- [x] Fix typo in README                   → https://github.com/o/r/pull/12
- [-] Theme switcher in settings           worth-it: Later — one request, no repeat
- [!] Raise the plan limit                 needs decision: 100 or 500?
```

| mark | meaning | set by |
|---|---|---|
| `[ ]` | todo | human |
| `[~]` | in progress (branch, date) | autoloop, at pick-up |
| `[x]` | PR opened (URL) | autoloop |
| `[-]` | skipped by `worth-it` (verdict + one-line reason) | autoloop |
| `[!]` | blocked — needs a human (the exact question or failure) | autoloop |

Autoloop edits **only the mark and the trailing annotation** of the task line it is working on. It never rewrites, reorders, or deletes task text or notes, and never adds tasks.

Iteration outcomes are also appended to `.autoloop/log.md`:

```markdown
## 2026-09-30 14:02 — Add a header row to the CSV export
- worth-it: Build (high) — two support tickets quote the missing header
- decisions: header names = column keys (matches the JSON export)
- tests: red (1 failing) → green (all 14 pass)
- review: 1 Layer-2 finding fixed; no Layer-1 findings
- result: [x] https://github.com/o/r/pull/31
```

---

## One iteration

### 0. Preflight — stop rather than guess

Check all of these. On any failure: do not pick a task, append the reason to the log, and **stop the loop** (see step 9).

- the working tree is clean (`git status --porcelain` ignoring `.autoloop/`) and on the default branch (`gh repo view --json defaultBranchRef` or `git symbolic-ref refs/remotes/origin/HEAD`); `git pull --ff-only` succeeds
- the backlog exists and `.autoloop/` is git-ignored
- no line is `[~]` — a leftover `[~]` means a previous iteration died mid-task; a human must look at it (mention the branch)
- `gh auth status` passes
- the guardrail counters (below) are not exhausted

### 1. Pick

Take the **first** `[ ]` line. If there is none, the backlog is drained: log it and stop the loop.

Choose a branch name `autoloop/<slug>` (kebab-case, ≤ 40 chars, from the task text; add `-2` etc. if it exists locally or on the remote). Mark the line `[~]` with the branch and today's date **before** doing anything else — this is the lock.

### 2. Gate — `worth-it`, unattended

Run the `worth-it` procedure on the task text and its notes, researching facts yourself (code, git log, tests) exactly as that skill says. Then act on the verdict instead of waiting:

| verdict | action |
|---|---|
| **Don't** / **Already covered** / **Later** | mark `[-]` with `worth-it: <verdict> — <one line>`; go to step 8 |
| **Smaller** | continue with the smaller move as the task (record it in the log) |
| **Build** | continue |

If the verdict hinges on a fact only a human knows (e.g. "is anyone actually hurt by this?" with no evidence either way in the repo), mark `[!]` with that question instead of guessing.

### 3. Design — self-grilling

Walk the decision tree the way `grilling` does, but answer each question yourself:

- **facts** → look them up in the repository and its history;
- **decisions with a clear recommended answer** (follows an existing pattern, is reversible, is what the task text implies) → take the recommendation and log it;
- **decisions only a human can make** — product or business choices, user-visible wording or values the task leaves open, anything irreversible (data migrations, public API removal, deleting user data), anything touching auth / billing / secrets → mark `[!]` with the question and your recommended answer, go to step 8.

Keep the result as a short plan: files to touch, behaviour to change, tests to add.

### 4. Trim — `prune`

Apply `prune` to the plan. Cut everything labelled speculative or ornamental; keep required and load-bearing. Do not reintroduce cuts later.

### 5. Implement — red → green

On a fresh branch from the up-to-date default branch (`git switch -c autoloop/<slug>`):

1. Find how tests run (CLAUDE.md, README, package scripts, Makefile, CI config). If the repository has no way to run tests for the changed code, mark `[!]` "no runnable tests for <area>" and go to step 8.
2. **Red:** write the test(s) for the new behaviour first; run them and confirm they fail *for the expected reason*.
3. **Green:** implement the smallest change that passes; run the **whole** suite and linters/type-checkers the repo already uses.
4. Never delete, skip, or loosen an existing test to get green. If green is not reached after three focused attempts, mark `[!]` with the failing output summary, go to step 8 (the branch stays local and unpushed for a human to inspect).

### 6. Self-review — `layered-review --diff`

Run `layered-review` against the branch's diff to the default branch. Fix findings and re-run the tests; at most **two** review → fix rounds.

- If a **Layer 1** (production impact) finding remains after two rounds, open the PR as **draft** in step 7, and mark `[!]` instead of `[x]`, quoting the finding.
- Remaining Layer 2 / 3 findings go into the PR body under "Known review notes".

### 7. Ship — commit, push, PR

- Commit with a message describing the change (follow the repo's commit style; include any attribution lines the session requires).
- `git push -u origin autoloop/<slug>`
- `gh pr create --base <default> --head autoloop/<slug>` with a body containing: the task line, the worth-it verdict, decisions taken in step 3, what was cut in step 4, test evidence (red → green), and review notes. Add the label `autoloop` if the repository has it.
- Mark the task `[x] → <PR URL>`.
- If push or PR creation fails, mark `[!]` with the error (the branch name is already on the line); do not retry in a loop.

### 8. Close the iteration

- `git switch <default>` — always return to the default branch, whatever happened.
- Append the iteration to `.autoloop/log.md`.
- Update counters: processed +1; consecutive failures +1 if the outcome was `[!]` for a failure (tests, push, PR, review), else reset to 0. A `[!]` for a human decision is not a failure.

### 9. Continue or stop

Stop when any holds: backlog has no `[ ]`; **5** tasks processed in this `/loop` run; **2** consecutive failures; preflight failed.

- **Under `/loop` (dynamic mode):** to continue, call `ScheduleWakeup` with `delaySeconds: 60`, the `/loop` input passed back verbatim as `prompt`, and a reason naming the next task; to stop, call `ScheduleWakeup` with `stop: true`.
- **Invoked directly:** do not schedule anything; just report.

Counters live in the log: count this run's iterations from the `## ` entries since the last `# run <timestamp>` header, which you write on the first iteration of a `/loop` run (i.e. when the loop was just started, not woken up).

End every iteration with a three-line report: task → outcome (mark + URL/reason), what is next, whether the loop continues.

---

## Guardrails

- **Never** merge, approve, force-push, push to the default branch, rewrite published history, or close PRs/issues.
- **Never** touch tasks other than the one picked, and never edit task text.
- **Never** change CI config, secrets, credentials, or permission settings as part of a task — block instead.
- **Never** delete, skip, or weaken existing tests.
- Stay inside the task: no drive-by refactors (that is what `prune` is for). Unrelated problems you notice go into the log as a suggestion, not into the backlog.
- One task at a time, in order. No parallel branches.

---

## Setting it up (for the human)

1. `echo .autoloop/ >> .git/info/exclude` (or add it to `.gitignore`) and write `.autoloop/backlog.md`.
2. Allow the loop to run without prompts for the commands it needs — `git`, `gh pr create`, the test runner — via auto mode or a `permissions.allow` list in `.claude/settings.json`. Anything still prompting will stall the loop at that step.
3. Start it: `/loop /autoloop`. Watch `.autoloop/log.md`; answer `[!]` lines; merge PRs. Start `/loop /autoloop` again whenever you've added tasks or cleared a stop.

## Relation to other skills

`autoloop` does not duplicate the neighbours; it chains them and replaces every "ask the user" with *decide by recommendation* or *block with the question*:

- `worth-it` — whether (step 2)
- `grilling` — how (step 3)
- `prune` — how little (step 4)
- `layered-review` — is it right (step 6)
