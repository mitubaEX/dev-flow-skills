---
name: worth-it
description: Decide whether the work about to be implemented is necessary at all, before any code is written. Separates the underlying problem from the requested solution, checks whether the problem is observed or imagined, asks what happens if nothing is done, whether existing means already cover it, and what the smallest move would be, then returns one verdict — Don't / Already covered / Smaller / Later / Build — with the evidence and what would change it. Use at the start of any ticket, feature request, refactor, "tech-debt" item, dependency upgrade, or fix, or when the user asks "do we really need this?", "is this worth doing?", "should we build this?", "本当に必要?", "やる価値ある?", "そもそも要る?", "やらなくてもいい?". Read-only; never implements. Upstream of `prune` (which trims a change already agreed on) and of `grill-me` (which sharpens how to do it). Invoke as `/worth-it <task / ticket / request / plan>` or `/worth-it` for the task currently under discussion.
---

# worth-it

Answer one question before anything is built: **is this work necessary?**

Most requests arrive as solutions ("add a retry button", "upgrade to v5", "extract a service"). The solution is rarely the problem. This skill finds the problem, checks whether it is real, asks what doing nothing costs, and only then decides whether the requested work, a smaller move, or nothing is the right answer.

This skill is **read-only** and ends with a verdict and a recommendation. It never starts implementing. The decision belongs to the user; after the verdict, wait.

Relationship to the neighbouring skills:

- `worth-it` — *whether* to do it.
- `grill-me` / `grilling` — *how* to do it, once it is worth doing.
- `prune` — *how little* of it to do, once the shape is agreed.

---

## Procedure

### 0. Get the target

| invocation | target |
|---|---|
| `/worth-it` | the task, ticket, or change currently under discussion in this conversation |
| `/worth-it <text>` | a ticket, feature request, plan, PR description, or one-line task pasted by the user |
| `/worth-it <issue-url>` / `<PR-url>` | a GitHub issue or PR (`gh issue view` / `gh pr view`; requires `gh auth status` to pass) |

If nothing is parseable, ask once: *"What are we deciding on — the current task, or something you'll paste?"* Do not guess.

### 1. Separate the problem from the solution

Write two lines:

- **Requested solution:** what was literally asked for, quoted where it exists.
- **Underlying problem:** who is hurt, how, and how often. If the request does not say, infer the most plausible problem and *label it as inferred*.

If no underlying problem can be stated — only "it would be nice", "best practice", "other tools have it", "it's been on the list" — say so. That is already most of the verdict.

### 2. Check the evidence

Is the problem **observed** or **imagined**?

Observed looks like: a bug report, a support ticket, a metric, a failing test, an incident, a user quote, a commit that worked around it. Imagined looks like: "might", "could", "someday", "in case", "users probably".

If a fact can be found in the environment, **look it up rather than asking**: `git log -S`, `git blame`, issue history, logs, existing tests, usage counts, the code path in question. Report what you found and where. Only the *decisions* go back to the user.

### 3. The do-nothing test

State concretely what happens if this is not done for the next three months. One sentence. Either a specific consequence ("the nightly job keeps failing twice a week and someone reruns it by hand") or, honestly, "nothing observable".

### 4. The existing-means test

Can the underlying problem be handled with what already exists — an existing feature or flag, a config change, a doc or a one-line answer to the person asking, a process change, a one-off script, a different tool already in the stack? Name it. This is where a large share of requests end.

### 5. The smallest move

If something must be done, what is the **smallest action that removes the pain** — not the requested solution, the pain? Often a one-line fix, a default change, a deleted branch, a comment, or a manual step documented once. Compare it to the requested solution on: lines touched, new surface area to maintain, and how reversible it is (two-way door vs one-way door).

### 6. Verdict

Exactly one:

| verdict | meaning | what follows |
|---|---|---|
| **Don't** | no real problem, or cost of inaction is ~zero | close it; say so plainly |
| **Already covered** | the problem is real and existing means handle it | point at the means; no code |
| **Smaller** | the problem is real; the requested solution is oversized | name the smaller move; then `/prune` it |
| **Later** | the problem is real but not yet; a trigger will tell | write the trigger ("when a second customer asks", "when the job fails 3× in a week") |
| **Build** | the problem is real, present, and the requested solution is the smallest move | proceed; carry the problem statement into the work as the acceptance test |

Add **confidence** (high / medium / low) and **what would change the verdict** — the single fact that, if true, flips it.

---

## Output

Keep it short. A reader should get the answer in the first line.

```
Verdict: <Don't | Already covered | Smaller | Later | Build>  (confidence: <high|medium|low>)

Requested:   <quoted or paraphrased solution>
Problem:     <who / how / how often>   [observed | inferred]
Evidence:    <what you found, where — or "none found">
Do nothing:  <concrete consequence, or "nothing observable">
Existing:    <means that already covers it, or "none">
Smallest:    <the smallest move, vs the request>

Recommendation: <one or two sentences>
Would flip it: <the one fact>
```

Then stop and wait. Do not implement, scaffold, or "just start on the easy part".

---

## Smells of unnecessary work

Any of these should make the do-nothing test bite harder:

| smell | tell |
|---|---|
| Solution-shaped request | names a mechanism, never a sufferer |
| No sufferer | "would be nice", "best practice", "the linter says", "other tools have it" |
| Single anecdote | one person, one time, no repeat |
| "While we're at it" | attached to unrelated work to avoid its own justification |
| Refactor with no upcoming change | code that nobody will touch next quarter gets restructured now |
| Tech debt without pain | called debt, but no incident, slowdown, or blocked change is named |
| Rewrite because unfamiliar | the code is disliked, not broken |
| Upgrade without a need | new major version, no needed feature, no security advisory |
| Generalising for a second consumer | the second consumer does not exist yet |
| Hardening without a path | defending against a threat with no route to the system |
| Tests for code about to go | coverage for something scheduled for deletion |
| Resume-driven | the technology is the point, the problem is the excuse |

---

## Calibration — worth doing even without a loud problem

The gate must not turn into a reason to never act. These pass even when the evidence is thin:

- A security issue with a real, described path to exploitation.
- A data-loss or corruption risk, even if not yet triggered.
- A legal, compliance, or contractual requirement.
- A blocking dependency for work the team has already committed to.
- Something the user has explicitly decided after hearing the verdict once. Do not re-litigate; move to `/prune`.

When two readings of the request both seem plausible, evaluate the *problem* under the more modest reading and say which one you took.
