---
name: context-cost-guard
description: >-
  Apply when context is growing fast, session cost looks high, or the user asks
  why a conversation is expensive. Explains the 4 accumulation sources (system
  prompt / conversation / tool results / skill triggers), the cost formula with
  cache_creation vs cache_read pricing, the 5-minute cache TTL, the 200K/300K/
  400K compact thresholds, sub-agent cost isolation, and 4 avoidance techniques.
  Triggers on "token cost", "session expensive", "long conversation", "why so
  costly", "should I compact", "context too large", "how much am I spending",
  or observable rapid context growth during execution.
version: 1.0.0
---

# Context Cost Guard

Every Claude API call sends the **entire context** (stateless nature of the LLM). Longer context means every turn is more expensive — and content, once in, **does NOT leave unless compacted**.

## Cost Formula

```
per-turn cost = cache_read × existing context tokens      ← paid every turn
              + cache_creation × newly added this turn    ← ~10× cache_read price
              + input × pure new input not cached         ← usually small
              + output × reply tokens                     ← not cached, full price
```

Opus 4.7 pricing (estimates):

- `output`: $75/M
- `cache_creation`: $18.75/M (25% more than input)
- `input` (uncached): $15/M
- `cache_read`: $1.50/M (10× cheaper, but not free)

## Four Accumulation Sources (all grow over time)

1. **System prompt** (~11K, fixed on first turn): rules + memory index + skill descriptions + tool schemas
2. **Conversation history** (linear growth): each user/assistant message 500-3000 tokens, does not auto-evict
3. **Tool results** (block growth): a `Read` of a 700-line file ~8K, `Grep` 50 lines ~2K, `Bash` 30 lines ~1K — **all stays in context**
4. **Skill triggers** (one-shot big chunk): calling the Skill tool reads the full SKILL.md into context (e.g. ~4K per skill)

## Cache 5-Minute TTL

- Active conversation within 5 minutes → prior content served via `cache_read` (10× cheaper).
- 5+ minutes idle → cache expires, next turn rebuilds at `cache_creation` (10× more expensive).
- This is why "I stepped away for 30 min, now it costs more" — the cache decayed.

## Sub-Agent Cost Isolation

- Sub-agents run in **isolated contexts** — the tokens they consume do NOT inflate the main session.
- The main session only receives the sub-agent's **concise report** (usually <3K).
- So dispatching sub-agents for independent research / exploration is an effective way to save main-session context.

## Compact Thresholds

- **200K tokens**: internal watch — start considering compact
- **300K tokens** (warn): actively suggest "compact or start a new session"
- **400K tokens** (fail): hard warning — quality degradation becomes visible

## Four Avoidance Techniques

1. **Outsource research/exploration to sub-agents** — main session only gets the report.
2. **Read large files with `Grep` + `offset/limit`** — avoid full-file `Read` when a slice will do.
3. **Plan Mode auto-clears context on accept** (`showClearContextOnPlanAccept=true`).
4. **Segment tasks** — compact or start a new session at natural breaks.

## When to Proactively Alert the User

- Execution phase, context ballooning quickly (>10× growth in minutes) → proactively flag.
- Peak approaching the warn threshold → proactively suggest cleanup techniques.
- **Do NOT** preventively suggest when there are no symptoms (avoid noise).

## "Build Day" vs "Routine Day"

- **Build day**: making structural changes (rules migration, skill reorganization, big plan) → $150+ daily is reasonable.
- **Routine day**: specific work on a clean baseline → $10-30 per session is reasonable.
- Exceeding routine baseline by 2-3× → audit that session's tool usage.

---

## Notes from the field

This model crystallized after one 2h session burned $115+ across 96 `opus` calls, peaking at 365K context. The surprise was that **`cache_read` is not free** — at $1.50/M against a 300K context, every turn was still paying ~$0.45 just to re-read the same cache. The four accumulation sources explain why long conversations compound: system prompt + history + tool results + skill triggers all add, and none subtract until compact. Cache TTL is the other silent killer — stepping away for lunch can 10× the next turn's cost.
