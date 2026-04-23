---
name: plan-mode-review
description: >-
  Apply before EnterPlanMode for non-trivial implementation tasks. Enforces a
  three-layer input review (problem validity / information sufficiency with
  P0/P1/P2 tagging / hidden issues and root cause) and confidence tags
  (Confident/Likely/Unclear) to prevent planning drift. Caps revisions at 3
  rounds. Triggers on "architect", "design", "refactor", "migrate", "new
  feature", "how should I approach this", "which approach is best", or multi-
  file / multi-module change requests. Skips for single-file tweaks, bug
  reproduction, config changes, and trivial fixes.
version: 1.0.0
---

# Plan Mode Review

Produce "proposal, issue list, architecture doc, multi-dimensional decision table" class documents only after entering Plan Mode and getting user approval.

## Triggers (any match)

- **Semantic**: architecture / design / approach / system / integration / refactor / new feature / "how should I" / "which way is best"
- **Scale**: crosses 3+ files / multiple modules / new dependency / changes data structure or API contract

## Skips

- Single-file change, bug reproduction, log/format fix, config change, chat, consult, factual Q&A
- User explicitly says "just change it" / "small edit" / "skip planning" → exit immediately

## Input Review (before planning)

Before entering planning, audit three layers (skippable only if the user explicitly says "just change it" / "small edit"):

1. **Is the problem real?** — Is the user's attribution reliable? Are the premises verified? Could the observed symptom actually be normal behavior?
2. **Is information sufficient?** — Tag missing info as `P0 must-have` / `P1 nice-to-have` / `P2 bonus`. Do NOT proceed past a `P0` gap with assumptions.
3. **Hidden issues?** — Given the symptom, is there a related issue the user missed, or a deeper root cause beneath the surface?

If any layer surfaces "a real issue the user hasn't seen", report back first and do NOT enter `EnterPlanMode`.

## Planning Flow

1. `EnterPlanMode` → sketch dimensional skeleton → user approves → `ExitPlanMode` → write formal doc.
2. **Architecture triggers**: first Grep/Read 5-15 related files, present a hypothesis list with `file:line` evidence for user confirmation (no interview-style Q&A).
3. **Tag every hypothesis/decision** as `Confident` / `Likely` / `Unclear`. `Unclear` items must be resolved before being written into the plan.
4. **After writing, back-check**: Are all key requirements tracked as tasks? Is the dependency graph free of breaks and cycles? Is wiring complete? Is the context budget sufficient? Does any of this conflict with prior decisions?
5. **Revision cap**: if the plan has been revised >3 rounds without convergence, commit the current version and move remaining refinements to a backlog. Don't loop forever.
6. **When iterating**, explicitly state what has been *removed / downgraded / deferred*, not just what has been added.
7. **Before writing any decision into the plan, self-check**: why this? What's the trade-off? Who's on the hook for this call?

---

## Notes from the field

The three-layer review was added after too many planning sessions where the "problem" turned out to be a misdiagnosed symptom. Layer 1 (is the problem real?) catches roughly 30% of requests before they turn into full plans. Layer 3 (hidden issues) is where the real value is — the user's attribution was right, but there's a deeper root cause they didn't see. `P0 / P1 / P2` tagging on missing info stops the assistant from marching ahead with assumptions. The 3-round revision cap exists because plans that don't converge in 3 rounds usually won't converge at all — commit the least-bad version and iterate with code.
