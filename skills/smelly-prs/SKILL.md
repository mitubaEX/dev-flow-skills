---
name: smelly-prs
description: Triage recently merged PRs of a GitHub repository and rank the ones that "smell" — God PRs, mixed concerns, missing tests, no reviewer, SOLID/DRY/KISS/YAGNI violations, GoF design-pattern misuse (Singleton abuse, missing Strategy/State, anemic domain, etc.), risky migrations, shotgun-surgery hotspots, and more. Use when the user wants to audit merge hygiene of a repo, find refactor candidates, hunt for risky merges to post-mortem, or pick suspicious PRs from a recent window. Requires the `gh` CLI to be authenticated against the target repo. Invoke as `/smelly-prs <owner/repo> [--limit N] [--since YYYY-MM-DD] [--base BRANCH]`.
---

# smelly-prs

Pick "smelly" PRs out of a repository's recently merged set, using classic software-engineering heuristics (Fowler's code smells, SOLID, DRY/KISS/YAGNI, Law of Demeter, plus team-flow process smells).

This skill is **read-only**: it never pushes, comments, or modifies any repo. It only fetches PR metadata and diffs via the `gh` CLI and produces a ranked report.

---

## Inputs

Parse from the user's invocation:

| arg | meaning | default |
|---|---|---|
| `<owner/repo>` | target GitHub repo (required) | — |
| `--limit N` | how many recent merged PRs to scan | `20` |
| `--since YYYY-MM-DD` | only consider PRs merged on/after | 14 days ago |
| `--base BRANCH` | restrict to this base branch | omit (any base) |

If `<owner/repo>` is missing, ask the user once. Do not guess.

---

## Procedure

### 1. Preflight

```bash
gh auth status
```

If not authenticated, stop and tell the user to run `gh auth login` (do not attempt it yourself).

### 2. List candidates

```bash
gh pr list --repo <owner/repo> \
  --state merged \
  --search "merged:>=<since>" \
  --limit <N> \
  --json number,title,url,mergedAt,mergeCommit,author,additions,deletions,changedFiles,baseRefName,labels,reviewDecision,body
```

If `--base` was supplied, append `--base <branch>` instead of filtering after the fact.

### 3. Per-PR analysis (parallel)

For each PR, spawn an **Explore** subagent in parallel (cap concurrency at ~8; if there are more PRs, batch). Each agent receives the PR's JSON metadata and runs:

```bash
gh pr view <n>   --repo <owner/repo> --json reviews,comments,files,commits,closingIssuesReferences
gh pr diff <n>   --repo <owner/repo>
```

If the diff exceeds ~3000 lines, ask the agent to read only the first 1500 lines and a sampled tail (~500 lines) and explicitly note in its findings that the diff was truncated.

Each agent returns a structured finding per detected smell:

```
{ pr: <n>, category: <string>, smell: <string>, severity: 1|2|3, evidence: <short snippet or file:line>, principle: <SOLID|DRY|KISS|YAGNI|LoD|GoF|process|risk|history> }
```

### 4. Detection rubric

Score each PR by summing severities across the categories below. **Score conservatively — when in doubt, don't fire.**

#### A. Scale & scope smells

- **God PR** (severity 3): > 800 added lines OR > 30 changed files in a single PR.
- **Mixed concerns** (2): feature + refactor + formatting + dependency bumps in one PR (heuristic: ≥3 of: prod-code add, test-only diff, lockfile bump, formatting-only diff, rename-only diff).
- **Silent dependency bump** (1): lockfile changes without PR body mentioning *why*.

#### B. Design / OO smells (Fowler)

- **Long Method** (1): new function > ~60 lines.
- **Long Parameter List** (1): a new/changed function with ≥5 params.
- **God Class** (2): a class file grows past ~400 lines or gains > 5 new public methods in one PR.
- **Feature Envy / LoD violation** (1): chains like `a.b().c().d()` introduced.
- **Primitive Obsession** (1): repeated raw strings/ints where a small type would clearly help (e.g., status codes as bare strings everywhere).
- **Duplicated Code (DRY)** (2): near-identical blocks (≥6 lines) added in two places in the same PR.
- **Shotgun Surgery** (2): one logical change requires edits across ≥6 unrelated-looking files.
- **Divergent Change** (1): a single file is touched for two clearly distinct reasons.

#### C. Principle violations

- **SRP violation** (2): a class/module gains a second responsibility (e.g., parsing + IO; auth + logging).
- **OCP violation** (1): a `switch`/`if-else` chain on a type tag is extended again rather than via polymorphism.
- **KISS violation** (1): clever one-liner / nested ternaries / deep generics where simpler code would do.
- **YAGNI violation** (1): new abstraction with a single caller, or config flag with no consumer, or "future use" parameter.

#### D. Design-pattern smells (GoF & enterprise patterns)

Look for both *missing* patterns (a hand-rolled mess where a known pattern would fit) and *misapplied* patterns (a pattern bolted on where simpler code would do). Cite the pattern name explicitly in `evidence` so reviewers can verify.

- **Strategy / State opportunity** (2): a `switch` / `if-else if` chain that dispatches on a type tag / enum / string is *extended* again in this PR. Cite the existing branches plus the new one. (Closely related to the OCP violation in section C — score once, prefer this label when a clear pattern fix exists.)
- **Singleton abuse** (2): a new global / static instance with mutable state, or a class with a private constructor + `getInstance()` that owns IO, config, or cache. Especially smelly if it's instantiated inside business logic (hidden dependency, kills testability).
- **Service Locator** (1): code calls a global registry (`Container.get(...)`, `ServiceLocator.resolve(...)`) inside a function body to fetch its own collaborators, instead of receiving them as parameters / constructor args.
- **Anemic Domain Model** (1): a new data-only class (just fields + getters/setters) whose behavior lives entirely in a sibling `*Service` / `*Helper` / `*Util` class. Note: in DTO / wire-format layers this is fine — only flag inside the domain layer.
- **Factory missing** (1): a constructor takes a discriminator argument (`type: "A" | "B"`) and branches internally to build different shapes. A factory / abstract factory would localize the choice.
- **Template Method opportunity** (1): two or more new (or expanded) methods share an identical skeleton and differ only in 1–2 inner steps. Cite the duplicated outer flow.
- **Decorator over inheritance** (1): a new subclass adds a cross-cutting concern (logging, caching, retry, auth) that varies orthogonally to the base type — composition (Decorator) would scale better than the inheritance tree being grown.
- **Observer / event-bus by hand** (1): a new code path manually maintains a list of listeners + notify loop, when the project already has an event bus / signal abstraction, or vice versa (drops the existing bus to call collaborators directly, breaking decoupling).
- **Facade / Mediator missing** (1): a caller wires up ≥4 collaborators directly. Hiding that orchestration behind a Facade / Mediator would isolate the churn.
- **Visitor / Builder overkill** (1): a Visitor introduced over a hierarchy with ≤2 stable concrete types, or a Builder with ≤3 fields and no real construction complexity. Pattern tax > benefit.
- **Premature abstraction interface** (1): a new `interface` / abstract base with exactly one implementation and no foreseeable second. (YAGNI cousin — score once.)
- **Magic constant / Replace Magic Number** (1): unnamed numeric or string literals introduced where a named constant or small enum/value-object would document intent. Skip for obviously self-evident values (`0`, `1`, `-1`, HTTP `200`/`404`, etc.).

When you flag a pattern smell, in `evidence` write the pattern name and the alternative, e.g. `Strategy: dispatch on "kind" extended for the 4th time in src/payment/router.ts:88` or `Singleton abuse: new Cache.getInstance() in src/api/handler.ts:14 — pass as constructor arg`.

#### E. Process / team-flow smells

- **Zero reviewers** (3): `reviewDecision` is null or no review comments, yet merged.
- **No tests added** (2): production code changed, no test file in the diff. Skip if PR is labeled `docs`, `chore`, `infra` etc.
- **Empty PR body** (1): description is empty or just a template skeleton.
- **Merged-then-reverted** (3): a later PR in the window reverts this one. Detect with: `gh pr list --search "in:title revert #<n>"` or by scanning later PR titles for `Revert "<this title>"`.
- **TODO/FIXME added** (1): diff introduces new `TODO` / `FIXME` / `XXX` markers.
- **Disabled tests** (3): diff contains new `.skip`, `xit(`, `xdescribe(`, `@Ignore`, `t.Skip(`, `#[ignore]`, etc.

#### F. Risk smells

- **Migration + app code mixed** (3): SQL/migration file changed alongside application logic (hard to roll back independently).
- **Swallowed exception** (2): new `catch` block with empty body or only a log.
- **Possible secret** (3): regex hit for AWS keys, private-key headers, long base64-looking tokens. Report as suspicion, not certainty.
- **Vulnerable dep** (2): only flag if the agent recognises a well-known vulnerable version pin (do not speculate).

#### G. History smells

- **Hotspot file** (1 per file, cap 3): the same file appears in ≥3 of the recent merged PRs in the window. Compute once across the candidate set, not per-PR.

### 5. Rank & label

Sum severities into a total score per PR. Bucket:

- **🚨 High** — score ≥ 7, OR any single severity-3 smell
- **⚠️ Medium** — score 3–6
- **💡 Low** — score 1–2
- **(skip)** — score 0

### 6. Report

Output a single markdown message. Format precisely:

```
# Smelly PRs in <owner/repo>
window: merged since <since> · scanned <N> PRs · flagged <K>

## 🚨 High

### #<num> <title> — score <total>
<url> · <author> · merged <mergedAt> · +<add>/-<del> across <files> files
Top smell: **<one-line summary of the strongest smell>**

Smells:
- **<category>** *(<principle>, sev <n>)* — <smell name>: <evidence (file:line if possible)>
- ...

---

## ⚠️ Medium

(same shape)

---

## 💡 Low

(same shape, may be omitted if the user asked for high-signal only)
```

End the report with a one-line aggregate, e.g.:

```
> Hotspot files this window: src/payment/processor.ts (5), src/api/router.ts (3)
```

If `K == 0`, say so plainly: "No smelly PRs in the last <since> for <owner/repo>. Merge hygiene looks clean."

---

## Guardrails — what NOT to do

- **Do not** clone the repo or run its tests.
- **Do not** post comments back to GitHub. Never call `gh pr comment`, `gh pr review`, `gh issue comment`.
- **Do not** flag style nits a linter would catch (formatting, import order, naming) — those are noise here.
- **Do not** flag pre-existing issues that just happen to appear in the diff context.
- **Do not** fabricate evidence. If you cannot point at a file:line or snippet, drop the finding.
- **Do not** moralise. Output is for triage, not blame. Stick to mechanics.

---

## Examples

User: `/smelly-prs anthropics/claude-code`
→ Default window (14 days), 20 PRs, any base branch.

User: `/smelly-prs mitubaEX/nvim_lua_config --limit 50 --since 2026-01-01`
→ Scan the last 50 merges since Jan 1, 2026.

User: `/smelly-prs foo/bar --base main --limit 10`
→ Only merges into `main`, last 10.
