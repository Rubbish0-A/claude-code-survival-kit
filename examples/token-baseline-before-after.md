# Token Baseline: Before / After

This document walks through the cost deltas the three `survival-kit` skills
deliver, anchored to the $115+ session referenced in the [README](../README.md).

---

## Anchor Session (real, tracked)

| Metric | Value |
|---|---|
| Duration | 2 hours |
| Opus calls | 96 |
| Peak context | 365K tokens |
| Bill | $115+ |
| Average per-call cost | ~$1.20 |

Observed symptoms:

- Most calls were `opus + xhigh` even for pure retrieval sub-tasks
- Main session context accumulated from full-file `Read` + tool-result explosion
- 30-minute idle gap triggered a `cache_creation` rebuild on resume

This is the session that made the cost model click. No project identifiers, no
names — just what one afternoon with Claude Code defaults looks like.

---

## Mechanism 1: Sub-Agent Downgrade (`subagent-matrix`)

| Call type | Before (default) | After (matrix) | Approx savings |
|---|---|---|---|
| "List files matching X" | `opus + xhigh` | `haiku + low` | ~10× |
| "Count matches, show paths" | `opus + xhigh` | `haiku + low` | ~10× |
| "Write tests for this module" | `opus + xhigh` | `sonnet + high` | ~3-5× |
| "Format and rename" | `opus + xhigh` | `sonnet + medium` | ~3-5× |
| "Refactor auth middleware" | `opus + xhigh` | `opus + xhigh` | same (correct) |
| "Design migration strategy" | `opus + xhigh` | `opus + max` | slightly higher (correct) |

**Key insight from the skill**: *model* downgrade delivers ~3-5× the cost cut of
*effort* downgrade. If the 96 opus calls had been split realistically — say,
30 haiku, 50 sonnet, 16 opus — the bill lands in the $30-40 range instead of
$115+.

---

## Mechanism 2: Context Accumulation (`context-cost-guard`)

Starting context: 11K (system prompt).

**Without** `context-cost-guard` discipline, a single long session accumulates roughly:

```
  11K   system prompt (fixed first turn)
+ 80K   conversation history      (20 turns × 4K avg)
+120K   tool results              (15 full-file Reads × 8K)
+ 50K   small tool calls          (25 Grep/Bash × 2K avg)
+ 20K   skill triggers            (5 skills × 4K)
─────
 281K   main session context, still growing
```

Every subsequent turn pays:

```
cache_read × 281K   +   cache_creation × (new tokens)   +   output × (reply)
```

At $1.50/M for `cache_read`, that's ~$0.42 per turn just to re-read the same
cached context — before adding any new work.

**With** `context-cost-guard` discipline applied (sub-agent outsourcing, `Grep`
+ `offset` for big files, Plan Mode auto-clear, task segmentation):

```
  11K   system prompt (fixed)
+ 40K   conversation history      (trimmed updates, shorter turns)
+ 15K   tool results              (Grep + offset on big files)
+ 10K   small tool calls          (minimized)
+ 12K   skill triggers            (3 skills max this session)
─────
  88K   main session context
```

Roughly a 3× reduction. Every turn now pays `cache_read × 88K` instead of 281K
— the same discipline compounds every turn after.

---

## Mechanism 3: Planning Drift Avoided (`plan-mode-review`)

This one doesn't reduce token cost directly — it avoids wasting 20-40 minutes
of planning context on the wrong problem.

Composite example (sanitized):

- **User reports**: "function returns wrong value in edge case X"
- **Without** `plan-mode-review`: assistant enters Plan Mode, designs a fix
  across 3 files (45 min, ~60K tokens), then user says: "oh wait, edge case X
  is intentional behavior — I misread the spec."
- **With** `plan-mode-review`: Layer 1 (is the problem real?) flags the premise
  for verification; a 2-minute clarification avoids the 45-minute waste.

In my own tracking, ~30% of planning sessions hit this class of issue. That's
where the drift multiplier in the math below comes from.

---

## The ~80% math

Multiplicative, worst-case:

```
  sub-agent downgrade     ≈ 0.3× per-call cost
× context discipline       ≈ 0.4× per-turn cost on long sessions
× avoided drift re-work    ≈ 0.8× (eliminate ~20% of wasted planning)
─────
= ~0.096×                  → an 80-90% cut on worst-case sessions
```

On a well-run session where you're already disciplined? Closer to 20-30%.
On a session that was already a mess (like the anchor)? 80%+.

**This is the math the rules encode.** Run your own session, diff your own bill.

---

## What this doesn't ship

- **A before/after bill screenshot.** Single-session variance is too high for any
  such image to be fair; the math does the honest work instead.
- **Other people's token logs.** Privacy.
- **A live dashboard.** If you want one, [`token-guard`](https://github.com/Rubbish0-A/token-guard)
  is a great companion — `survival-kit` is about *why* your bill looks the way
  it does; `token-guard` is about *how much* and *where*.
