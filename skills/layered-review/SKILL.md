---
name: layered-review
description: Review a single PR (or a local diff / branch) using a top-down three-layer method — (1) production-impact check (feature-flag gating, dead references), (2) chunk-level comprehension check (SOLID / DRY / YAGNI at the "does this shape make sense" altitude, not line-by-line), (3) surface check (shallow methods, patterns foreign to this repository). Use when the user wants to review a specific PR carefully, self-review their own working-tree change before pushing, or apply a consistent review discipline. Complements `smelly-prs` — that skill triages many merged PRs; this skill goes deep on one change. Read-only by default; never pushes, comments, or edits. Requires `gh` CLI when reviewing a remote PR. Invoke as `/layered-review <PR-url|PR-number|--diff>`.
---

# layered-review

Review one code change — a specific PR, or the local working-tree diff — with a disciplined **three-layer top-down pass**. Each layer has a different altitude and a different question. Do not skip layers, do not merge them: the point of the discipline is that a shallow layer's answer changes whether you spend time on the deeper ones.

The three layers, in order:

1. **Production impact** — could this change *hurt* if it lands? (flag gating, dead references)
2. **Meaning at the chunk level** — does the change *make sense as a shape*? (SOLID / DRY / YAGNI, viewed as blocks, not lines)
3. **Surface** — does the change *fit this codebase*? (shallow methods, out-of-place patterns)

This skill is **read-only**. It fetches a PR / diff and produces a written review. It never comments on GitHub, edits files, or pushes.

---

## Inputs

Parse from the user's invocation:

| arg | meaning |
|---|---|
| `<PR-url>` | e.g. `https://github.com/foo/bar/pull/123` |
| `<PR-number>` | e.g. `#123` — only valid when `cwd` is a checkout of the target repo |
| `--diff` | review the local working-tree diff (`git diff` vs the base branch) |
| `--base BRANCH` | with `--diff`, override the base branch (default: origin's default branch) |
| `--repo owner/repo` | with `<PR-number>` when the shell is not inside that repo |

If nothing parseable is supplied, ask the user once: *"What are we reviewing — a PR URL/number, or the local working-tree diff?"* Do not guess.

---

## Procedure

### 0. Preflight

- If the target is a GitHub PR: run `gh auth status`. If not authenticated, stop and ask the user to run `gh auth login` themselves. Do not attempt to log in.
- If the target is `--diff`: verify you are inside a git worktree and there is at least one changed file vs the base branch. If the working tree is clean, stop and say so.

### 1. First pass — read everything once

Before any layered analysis, **read the entire change end-to-end once**, quickly. Do not take notes on issues yet. The goal is a mental map of *what this PR is trying to do*.

For a PR:

```bash
gh pr view <n> --repo <owner/repo> --json title,body,author,baseRefName,additions,deletions,changedFiles,labels,files,commits,reviewDecision
gh pr diff <n> --repo <owner/repo>
```

For `--diff`:

```bash
git diff --stat <base>...HEAD
git diff <base>...HEAD
```

If the diff is very large (> ~3000 lines), read the PR body / commit messages carefully and sample the diff (all touched file headers + the first meaningful hunk of each file + any migration/config/flag files in full). Note in the final report that a full read was not possible.

At the end of this pass, write down (internally) one sentence: *"This change tries to __."* If that sentence does not form, stop and ask the user what the change is supposed to do — you cannot layer-review a change whose intent you cannot state.

---

### 2. Layer 1 — Production impact

Question: **Can this change hurt production the moment it lands?** Only two sub-checks.

#### 2.1 Flag / gating

For any new user-visible behavior, external side effect, schema change, or expensive path:

- Is it behind a feature flag / config toggle / role check / environment gate?
- Is the flag **off by default** in production configs?
- Is the flag actually *checked* on all entry paths — not just the primary one?
- Is there a documented rollback path (flip the flag, revert the migration, etc.)?

Flag a finding when: new behavior ships unconditionally, or the flag exists but one code path bypasses it, or the flag defaults to on.

Do **not** flag: pure refactors, internal-only helper additions, test-only changes, docs — even if they lack a flag.

#### 2.2 References / dead code

For anything the diff *removes*, *renames*, or *narrows in visibility*:

- Are there remaining references in the repository? (`rg`/`grep` the old symbol name — including strings, YAML/JSON configs, generated bindings, migrations, and templates)
- For renamed public APIs: is there a deprecation shim or a migration note?
- For newly added abstractions: is there at least one **real** caller in this same diff? (an unused new module, class, or interface is a Layer-1 concern because it will bit-rot silently)

Flag a finding when: a removed/renamed symbol still has live references, or a newly added public surface has zero real callers.

Do **not** flag: internal helpers that are obviously wired up further down the diff — read the whole PR before firing.

**Layer-1 exit rule:** if Layer 1 produces any severity-3 finding (see §5), *stop and report*. Do not descend into Layer 2 / 3 — the change is not review-ready. A note that says "Layer 2 and 3 skipped: this change is not safe to land as-is" is a valid, useful review.

---

### 3. Layer 2 — Meaning at the chunk level

Question: **Do the pieces of this change make sense as blocks, without reading the lines?**

You are looking at the diff as *shapes*: what modules were added, what methods were added to which class, what the seams are. Do **not** open every hunk line-by-line here. That is Layer 3's job.

Judge against these three principles — cite the principle explicitly when firing a finding:

#### 3.1 SOLID (mostly S and O)

- **SRP** — does each new / substantially-changed module have *one* reason to change? A module that grew a second responsibility in this PR (e.g. request parsing + IO + audit logging in one class) is a finding.
- **OCP** — was an `if/else` or `switch` chain on a type tag extended *again* in this PR? That's the tell that the extension point is wrong. (This overlaps with the "Strategy opportunity" pattern smell — pick one label, do not double-count.)
- **LSP / ISP / DIP** — only flag if a violation is glaring; do not fish. In practice L2 finds SRP and OCP hits far more often.

#### 3.2 DRY

- Is the *same logic* (not the same tokens — the same *decision*) repeated in two or more places introduced by this PR?
- Ignore surface-level duplication that's genuinely coincidental (two config keys that happen to share a name).
- A finding here has to point at both duplicates (`file:line` each).

#### 3.3 YAGNI

- Any new abstraction with **exactly one caller** and no near-term second caller in flight? (interface with one impl, factory that only builds one type, config knob with no consumer, "future use" parameter, generic type parameter that's always instantiated the same way.)
- Any dead branch (`if version >= 2` when version is always 2)?
- Any TODO/FIXME/XXX added that hides scope that should have been in this PR?

**Layer-2 exit rule:** Layer 2 findings are almost never blockers on their own, but they are the ones that *shape long-term maintainability*. Write them as **redesign notes**, not "please fix this line" comments.

---

### 4. Layer 3 — Surface / fit

Question: **Does this code look like this codebase?**

Now open the hunks. Two things to look for.

#### 4.1 Shallow methods

A "shallow" method is one where the **interface is nearly as complex as the body**: the signature has ~as many parameters and concepts as the body has statements. Classic examples:

- `fooManager.setFooEnabled(user, enabled)` whose body is `user.fooEnabled = enabled`.
- Wrapper methods that only re-order arguments or rename them.
- One-line delegations that add no invariant, no logging, no defaulting — just a rename.

Ousterhout's rule of thumb: **deep methods are good, shallow methods are noise**. A method earns its keep by hiding non-trivial work behind a simpler interface. Prefer inlining shallow methods, or bundling multiple shallow methods behind one deeper method that owns an invariant.

Do **not** flag: intentionally minimal public API methods that hide a *future* implementation (that's a Layer-2 YAGNI check, not this). Do **not** flag: test helpers — they are supposed to be shallow.

#### 4.2 Foreign patterns

Look for constructs that this repository does **not** otherwise use. Examples:

- A new dependency-injection framework used in one file when the rest of the codebase passes deps as constructor args.
- Async / concurrency primitives (goroutines, `Promise.all`, threading) introduced in a codebase that is otherwise synchronous.
- A new logging / error-handling / config mechanism when a house one already exists.
- A stylistic choice (functional pipeline chain, decorators, metaclasses, reflection) that no other file uses.
- Copy-pasted patterns from a library's README that ignore this codebase's own wrappers.

To check: `rg` for one distinctive keyword of the pattern across the repo (excluding the diff itself). If the count is near zero outside this PR, fire the finding.

Do **not** flag: patterns that are new *because they are being introduced by this PR intentionally*. That will be obvious from the PR body — read it. In that case Layer 3 is instead: *"Is the introduction rationale documented? Is there a follow-up plan to convert existing sites, or an ADR?"*

---

### 5. Severity & findings format

Each finding gets one severity:

- **sev 3 (blocker)** — Layer 1 flag/reference miss, or an obvious correctness / security regression noticed while reading. Change is not review-ready.
- **sev 2 (redesign)** — Layer 2 shape problem that will bite maintainability. Ask for a redesign or explicit ack.
- **sev 1 (polish)** — Layer 3 fit / shallow-method issue. A comment; not a blocker.

Structure every finding as:

```
- **[Layer <n> · <principle>]** sev <s> — <one-line claim>
  evidence: <file:line or file@hunk>
  suggestion: <one sentence — what to do about it>
```

Do not invent evidence. If you cannot cite a `file:line`, drop the finding.

---

### 6. Report

Output one markdown message with this shape. Keep prose short; be specific.

```
# Review — <PR title or "local diff on <branch>">
<url or "working-tree diff vs <base>"> · +<add>/-<del> across <files> files

**Intent (as I read it):** <one sentence>

## Layer 1 — Production impact

<summary line: "clean" / "one flag issue" / "removed symbol still has references">

<findings, if any, in the format above>

## Layer 2 — Meaning at the chunk level

<summary line>

<findings, if any>

## Layer 3 — Surface / fit

<summary line>

<findings, if any>

---

**Verdict:** ✅ ready to merge | 🟡 ready after Layer-<n> notes are addressed | 🚨 not review-ready (Layer 1 blocker)

**What I did not check:** <e.g. "did not run tests locally", "diff was truncated at 3000 lines", "did not audit the migration SQL semantics">
```

If Layer 1 tripped the exit rule (§2), the report contains Layer 1 findings, the 🚨 verdict, and a line saying `Layer 2 and 3 skipped — resolve Layer 1 first.`

---

## Guardrails — what NOT to do

- **Do not** post the review to GitHub. Never call `gh pr comment`, `gh pr review`, `gh issue comment`. The user decides what to send.
- **Do not** edit files, stage, commit, or push. This skill is read-only.
- **Do not** cross the layers. If you catch yourself deep in a hunk during Layer 1, stop and come back later. Layer discipline is the whole value.
- **Do not** flag lint-level nits (formatting, import order, trivial naming). A linter's job, not this skill's.
- **Do not** flag pre-existing issues that merely appear in the diff context. Only new / changed / deleted lines count.
- **Do not** invent `file:line` citations. Drop the finding if you cannot point at real evidence.
- **Do not** moralize. Output is for the reviewer to act on, not for scoring the author.
- **Do not** claim the review is complete when the diff was truncated. Say so in "What I did not check."

---

## Examples

User: `/layered-review https://github.com/mitubaEX/dev-flow-skills/pull/12`
→ Fetch PR #12, do the three-layer pass, output the report.

User: `/layered-review --diff`
→ Review the current worktree's diff vs `origin/main` (or the detected default branch).

User: `/layered-review --diff --base develop`
→ Review the worktree's diff vs `origin/develop`.

User: `/layered-review #42 --repo foo/bar`
→ Fetch PR #42 from `foo/bar` and review it, even though cwd is a different repo.

---

## Relation to other skills

- **`smelly-prs`** — bulk-triages *many* recently merged PRs and ranks the smelliest. Use it when you want to *find* a PR worth reviewing. Then use **`layered-review`** on that PR.
- **`code-flow:review-respond`** — actually replies to reviewer comments on your own PR. Use it after `layered-review` if the review target is your PR and you want to act on feedback you received (not on feedback *this* skill produced).
