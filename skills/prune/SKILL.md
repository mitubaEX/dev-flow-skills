---
name: prune
description: Strip a feature, spec, plan, UI, or code change down to what the request and its real user actually need. Looks through four lenses — product scope (features, flows, states, settings), user experience (screens, controls, dialogs, copy, polish), code design (abstractions, options, defensive code, YAGNI), and delivery (docs, tests, flags, rollout) — and labels every detail required, load-bearing, speculative, or ornamental, then emits a cut list. Also works as a silent stance while planning or building so the result stays minimal. Triggers on "keep it simple", "minimal", "MVP", "too much", "don't over-engineer", "over-build", "gold plating", "YAGNI", "作りこみすぎ", "過剰", "シンプルに", "最小限で", "削ぎ落として", "盛りすぎ", or when a spec / diff is visibly growing past its request. Read-only in audit mode; applies cuts only when the user names them. Invoke as `/prune` (audit the current plan or conversation), `/prune --diff`, `/prune <PR-url|PR-number>`, or `/prune <spec / feature list / mockup description / plan text>`.
---

# prune

Cut a thing down to **what the request and its real user need now**. Not what would be nice, not what a future user might want, not what would make it look finished.

The thing can be a feature idea, a spec or PRD, a screen or flow, a settings page, a plan, a diff, or a PR. The method is the same at every level; only the smells differ.

Two moments:

1. **Stance** — rules applied silently while planning, designing, or building.
2. **Audit** — an explicit pass over a target that produces a cut list.

Use the stance always. Run the audit when invoked, or when you notice the target has grown past its request.

This skill assumes the work itself is worth doing. If that is still open, run `/worth-it` first; `prune` trims a change, it does not question it.

---

## The one test

For every feature, screen, state, control, option, message, layer, parameter, branch, test, and doc section, ask:

> **Does the person this is for need it, now, on the path they actually take — or is it required to make that path work?**

If neither, it does not go in. "It might be useful later", "some users might want it", "it is more complete this way", and "it was cheap to add" are not answers to that question.

Two people matter: the **requester** (who asked) and the **end user** (who will use it). Something can be unrequested yet load-bearing for the end user (an error message when saving fails). Something can be requested by nobody and useful to nobody (a theme picker on an internal admin tool). Judge against both.

---

## Stance — rules while planning or building

1. **Write the ask and the main path in one sentence each.** *Ask:* what was requested. *Main path:* the one thing the end user comes here to do. Everything you produce must trace back to one of the two.
2. **One way to do each thing.** No second entry point, shortcut, alternate layout, or "advanced" mode until the first way is in use and someone has hit its limit.
3. **Defaults instead of settings.** Every setting is a decision pushed onto the user. Choose the value; add a control only when two real users demonstrably need different values.
4. **The main path gets polish; side paths get correctness.** Empty states, edge-case screens, rare errors, and admin views need to work, not to be beautiful. Ship them plain.
5. **Match the existing altitude.** Do not introduce a layer, pattern, component, or visual style the product or codebase does not already use. Fitting in beats "doing it properly".
6. **Rule of three for abstraction and for generalisation.** Two similar blocks stay as two blocks; two similar screens stay as two screens. Generalise at the third instance, and only if the three are truly the same thing.
7. **Handle what happens here, not what cannot.** Validate at trust boundaries; message the failures a user can actually hit. No guards against your own code, no copy for states that cannot occur.
8. **No performance, scale, or flexibility work without a number.** No caching, pagination, batching, i18n scaffolding, multi-tenant hooks, or plugin points unless the request is about that or you have a measurement.
9. **No "while I'm here".** No renames, reformatting, redesigns, or copy edits outside what the request touches. Note them instead.
10. **Undecided means out, not both.** If you cannot tell whether something is needed, ask one question or leave it out and say so. Never build both branches to be safe.

Keep a short **Deferred** list as you go: what you noticed and deliberately left out. Report it at the end. That list is where the urge to over-build goes, instead of into the product.

---

## Smells — by lens

Over-building rarely looks wrong up close. Look for these shapes.

### Product scope

| smell | tell |
|---|---|
| Feature for a persona nobody named | "admins might want", "power users could", "for enterprise later" |
| Second way to do the same thing | keyboard shortcut + button + menu item + gesture, all in v1 |
| Settings instead of a decision | toggle, theme, layout, density, or notification preference with no one asking for the other value |
| Whole-lifecycle build | create is asked for; edit, duplicate, archive, export, and bulk-actions arrive with it |
| Platform sprawl | mobile / offline / i18n / dark mode / accessibility-beyond-baseline added to a request that named none |
| Speculative integration | webhooks, API, CSV export, Slack notifications "so it can connect to things" |
| Analytics for questions nobody asked | events, dashboards, funnels tracked "in case" |

### User experience

| smell | tell |
|---|---|
| Confirmation theatre | dialogs on actions that are cheap or reversible |
| Onboarding for a one-screen tool | tours, tooltips, coach marks, welcome modals |
| Empty-state art | illustrations and copywriting for a state users see once |
| Edge-case screens with main-path polish | error pages, 0-result pages, and admin views given animations and custom layouts |
| Micro-copy sprawl | helper text, placeholders, tooltips, and info icons all explaining the same field |
| Choice overload | filters, sorts, view modes, and column pickers on a list with a dozen rows |
| Decoration | animation, transitions, badges, avatars, icons added because the design "felt flat" |
| Personalisation nobody requested | rename, reorder, pin, favourite, colour-code |

### Code design

| smell | tell |
|---|---|
| Speculative generality | interface / base class / generic with exactly one implementation |
| Plugin-shaped code | registries, hooks, strategy maps, event buses with one registrant |
| Phantom configuration | option, flag, env var, or parameter whose only value is the default |
| Defensive theatre | null-checks and type-guards on values your own code just produced |
| Impossible error paths | `catch` / `else` branches that cannot be reached |
| Premature performance | cache, memo, pool, batch, worker, index with no benchmark |
| Imported ceremony | a layer or pattern that appears nowhere else in this repository |
| Compatibility for no one | shims, adapters, deprecation paths for code with no external callers |
| Helper with one caller | a `utils` function used once, or a wrapper that only forwards |
| Naming tells | `Manager`, `Handler`, `Base`, `Abstract`, `Generic`, `Common`, `Util`, `Factory` — a question each, not a verdict |

### Delivery

| smell | tell |
|---|---|
| Hypothetical tests | tests for inputs no caller can produce, mocks deeper than the change |
| Docs in excess of behaviour | README sections, ADRs, diagrams for a change a reader would understand from the diff |
| Feature flag for a flip | a flag, rollout plan, or migration path guarding a change that could just ship |
| Drive-by refactor | renames, moves, reformatting in files the request did not need |

---

## Audit — procedure

### 0. Get the target

| invocation | target |
|---|---|
| `/prune` | the plan, spec, or change discussed so far in this conversation |
| `/prune --diff [--base BRANCH]` | local working-tree diff vs the base branch |
| `/prune <PR-url>` / `/prune <PR-number> [--repo owner/repo]` | a GitHub PR (`gh pr view`, `gh pr diff`; requires `gh auth status` to pass) |
| `/prune <text>` | a spec, PRD, feature list, mockup description, ticket, or plan pasted by the user |

If none is parseable, ask once: *"What should I prune — the current plan, the local diff, a PR, or a spec you'll paste?"* Do not guess.

### 1. Write the ask and the main path

Two sentences. *Ask:* what was actually requested, quoted where it exists; for a PR, from title and body, not from what the diff does. *Main path:* the one thing the end user does with it. Everything below is judged against these two.

### 2. Classify every unit

Walk the target in units — a feature, a screen, a state, a control, a message, a file, a function, a parameter, an option, a branch, a test, a doc section — and give each exactly one label:

| label | meaning | verdict |
|---|---|---|
| **Required** | the ask, or the main path, directly | keep |
| **Load-bearing** | neither asked for nor the main path, but the main path breaks, confuses, or misfits the product / repo without it (a save-failed message, an existing convention, a trust-boundary check, the test for the changed behaviour) | keep, one-line justification |
| **Speculative** | serves a future, hypothetical, or unnamed person — another persona, another scale, another platform, another caller | cut |
| **Ornamental** | serves neither; it is there because it felt complete, thorough, or polished | cut |

Be concrete: name the unit. "The settings page is too big" is not a finding. "Settings › Notifications › digest frequency: nobody asked for digests; delete the control and send immediately" is. For code: `path:symbol`.

### 3. Produce the cut list

Output, in this order:

1. **Ask / main path** — the two sentences from step 1.
2. **Cut list** — every Speculative and Ornamental unit: name, lens (scope / UX / code / delivery), label, one sentence why, and the minimal replacement ("delete", "hardcode to X", "inline into Y", "plain text instead of a dialog"). Order by how much each removes.
3. **Kept with justification** — Load-bearing units only. Required units are the point and are not listed.
4. **Minimal shape** — three to six lines describing the target after the cuts: which screens / flows / files remain and what the user does. If a large fraction is cut, say so plainly with a rough count (features, screens, or lines).
5. **Deferred** — anything worth doing *later* that the cuts removed, so nothing is lost, only postponed. Each with the trigger that would justify it ("when a second team needs export", "once the list exceeds ~100 rows").

Do not apply the cuts. If the user says to, apply them as a separate step and only the ones they named.

---

## Calibration — what is *not* over-building

The brake must not turn into an unusable or unmaintainable result. Keep these even though nobody literally asked:

- An error message for a failure the user can actually hit, and a loading state where a wait is noticeable.
- Baseline accessibility on the main path: labels, focus order, contrast, keyboard operability.
- Following a pattern the product or repository already uses everywhere, even if heavier than you would choose.
- Input validation at a real trust boundary.
- A test for the behaviour that changed.
- A name, a label, or a small type that makes the thing clearer to the next person.
- The third extraction, when the same thing genuinely appears three times.

When two readings of the ask both seem plausible, the smaller one wins, and you say which reading you took.
