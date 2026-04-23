---
name: subagent-matrix
description: >-
  Use when dispatching sub-agents via the Task/Agent tool to choose model and
  effort correctly and avoid cost blowups. Provides the model x effort selection
  matrix, the tripwire principle (start xhigh, escalate only on observed
  failure), never-pair combinations, and cost guardrails. Triggers when the
  assistant is about to call Task/Agent, plans parallel sub-agents, or the user
  asks "which model for X", "should I use opus/sonnet/haiku", "is this xhigh or
  max", "can I downgrade", or discusses sub-agent costs.
version: 1.0.0
---

# Sub-Agent Dispatch Matrix

When dispatching sub-agents via the Task/Agent tool, pick **both** model and effort based on task nature. Stop dispatching every sub-agent as `opus + max` "to be safe" — it is the single biggest preventable Claude Code cost leak.

## Parallel Task Execution

ALWAYS use parallel Task execution for independent operations:

- GOOD: Launch 3 agents in parallel — one for security audit, one for performance, one for type checking
- BAD: Sequential when the work is genuinely independent

Send one message with multiple `tool_use` blocks — not one message per agent.

## Multi-Perspective Analysis

For complex problems, consider split-role sub-agents when the user explicitly requests deep analysis:

- Factual reviewer
- Senior engineer
- Security expert
- Consistency reviewer
- Redundancy checker

**Cost cap**: parallel roles use `sonnet + xhigh`, NOT `opus + max`. The synthesis step in the main session keeps `opus + max` if needed. Never spin up 5 `opus + max` agents concurrently — it floods context and cost.

Most tasks do not need 5 parallel agents.

## Model x Effort Selection

| Task Type | model | effort | Typical scenarios |
|---|---|---|---|
| Pure retrieval | haiku | low | Search files, list directories, git log, count/stat |
| Simple edit | haiku/sonnet | medium | Single-file tweaks, format, rename, docs |
| Routine coding | sonnet | high | CRUD, config changes, tests, simple refactoring |
| Complex coding | opus (inherit) | xhigh | Multi-file changes, complex algorithms, agent logic |
| Initial review | sonnet | high | Code review for small diffs, low-risk changes |
| Deep review | opus (inherit) | max ⚠ | Auth, payment, data migration, concurrency, security |
| Deep reasoning | opus (inherit) | max ⚠ | Architecture design, complex analysis, Plan agent |
| Uncertain | opus (inherit) | xhigh | Default — escalate on observed failure, not prediction |

`⚠` = requires explicit justification in the sub-agent prompt (why xhigh is insufficient).

## Escalation Conditions

**Escalate model to opus** when:

- Changes span 3+ files with interdependencies
- Logic involves authentication, payment, data migration, or security
- Sub-agent results appear contradictory or unreliable
- Task requires understanding cross-module call chains or framework conventions

**Escalate effort to max** (on top of opus) when:

- An xhigh run produced shallow analysis or missed an obvious edge case
- Multiple sub-agents returned contradictory conclusions
- Task explicitly involves formal verification, cryptography, or regulatory compliance
- **Do NOT** pre-emptively escalate to max "to be safe" — use tripwire, not prediction

## Sub-Agent Output Discipline

Instruct sub-agents to return concise results:

- File paths, key findings, and a confidence level
- Do NOT paste large code blocks or full logs back into the main conversation
- This reduces context bloat in the parent session

## Effort Level Guide

On Opus 4.7+, effort is **orthogonal to model** — tune them as a pair.

| Effort | When to use | Typical model pairing |
|---|---|---|
| low | Pure retrieval, file listing, git log | haiku |
| medium | Simple CRUD, config edits, docs | sonnet |
| high | Routine coding, small-diff review | sonnet / opus |
| xhigh | **Default.** Coding + agentic work, tool-call debugging | opus |
| max | Architecture, security, payment, migration, deep debugging | opus |

## Cost Guardrails

**Savings come from model, not effort.** Opus 4.7 low-effort already matches Opus 4.6 medium-effort — aggressive effort-downgrade on opus has diminishing returns. **Downgrade the model first, effort second.**

1. **Model downgrade ≈ 3-5× the cost savings of effort downgrade.**
   Considering `opus + xhigh → opus + high` to save cost? Try `sonnet + xhigh` instead.

2. **Tripwire, not prediction.**
   Start at xhigh. Escalate to max only after observing concrete failure (shallow analysis, missed edge case, contradictory output). No pre-emptive max.

3. **`[1m]` context is a silent cost multiplier.**
   Every thinking token gets re-processed across the full context.
   `[1m] + max` on a 500K session ≈ 5× the cost of `[1m] + xhigh`.
   Open `[1m]` only when the task genuinely spans >200K tokens.

4. **Parallel agent budget.**
   Max 3 concurrent opus agents, or 5 concurrent sonnet agents.

## Never Pair

- `haiku × {xhigh, max}` — waste, small models can't exploit deep thinking
- `opus × low` — inverted, use `sonnet + medium` instead
- `[1m] × max` on routine tasks — open `[1m]` only when task spans >200K tokens

---

## Notes from the field

This matrix is the single biggest cost leak I fixed. Before: every sub-agent was dispatched as `opus + xhigh` "by default". After: the mix is `haiku` for retrieval, `sonnet` for routine coding, and `opus + max` reserved for genuine architecture work. One 2h session that burned $115+ across 96 `opus` calls looks very different once the matrix is honored — most of those calls should have been `sonnet + high` or `haiku + low`.
