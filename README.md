<p align="right">
  <b>English</b> ·
  <a href="./README.zh-CN.md">中文</a>
</p>

<p align="center">
  <img src="./assets/cover.png" alt="claude-code-survival-kit — 3 auto-triggering Claude Code skills for cost and drift prevention" width="100%">
</p>

# claude-code-survival-kit

> **Don't go into production without these.**
>
> Three flagship skills extracted from 6 months of real Claude Code usage.
> Not coding principles. Not spec-driven workflow. Just the survival kit.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-orange.svg)](https://code.claude.com)
[![Status](https://img.shields.io/badge/status-v1.0%20MVP-blue.svg)](#roadmap)

---

## Why this exists

One 2-hour Claude Code session. 96 `opus` calls. Peak context: **365K tokens**. Bill: **$115+**.

That was one session. I ran many.

After 6 months of tracking where tokens actually go and where plans actually drift, I extracted three specific rules. On cache-hit math and sub-agent cost isolation, they **cut similar sessions by ~80%**.

This repo ships those three rules as auto-triggering Claude Code skills. Install takes 10 seconds. No config.

---

## What makes this different

This is not another "best practices" repo. The Claude Code ecosystem already has excellent ones. Pick your lane:

| Repo | What it teaches | What we DON'T try to be |
|---|---|---|
| [`karpathy-skills`](https://github.com/forrestchang/andrej-karpathy-skills) | Coding principles (Think Before Coding, Simplicity First, ...) | We don't teach *how to write code* |
| [`cc-sdd`](https://github.com/gotalab/cc-sdd) | Spec-driven development workflow (17 skills, Kiro-style) | We don't run your dev pipeline |
| [`best-practice`](https://github.com/shanraisshan/claude-code-best-practice) | Comprehensive tutorial site, videos, guides | We don't duplicate their breadth |
| **`claude-code-survival-kit`** | **Cost model. Drift prevention. Meta-methodology.** | |

We complement them. We do not compete with them.

**The niche**: everything that happens *around* your code — not in your code. How to pick sub-agent model/effort without burning $100 in an afternoon. How to not march into planning with a misdiagnosed problem. How to understand why your 3-hour session cost 10× what the last one did.

---

## Install

**Prerequisite**: Claude Code installed (any recent version with plugin support).

```bash
# Step 1: add this marketplace
/plugin marketplace add Rubbish0-A/claude-code-survival-kit

# Step 2: install the plugin
/plugin install survival-kit@claude-code-survival-kit
```

That's it. Three skills now auto-trigger when relevant. Nothing to configure.

To verify:
```bash
/plugin list
# Should show: survival-kit (enabled)
```

---

## The 3 Rules

Each skill is an auto-triggering Markdown file. You don't invoke them — Claude Code does, when the description matches your current task.

### 1. `subagent-matrix` — Stop burning `opus + max` by default

**Symptom you'll see**: every sub-agent spawned as `opus + xhigh`. Bill climbs fast. Most of those calls should have been `sonnet + high` or `haiku + low`.

**What it does**: provides a concrete model × effort selection matrix keyed to task type (pure retrieval → haiku+low, routine coding → sonnet+high, architecture/security → opus+max). Includes the **tripwire principle** (start at xhigh, escalate to max ONLY on observed failure, never preemptively), never-pair combinations (`haiku × max` is always waste), and the math that **downgrading the model saves 3-5× what downgrading the effort does**.

**When it triggers**: any time the assistant is about to dispatch sub-agents, or you ask "which model should I use for X".

[→ View SKILL.md](./plugins/survival-kit/skills/subagent-matrix/SKILL.md)

---

### 2. `plan-mode-review` — Stop planning misdiagnosed problems

**Symptom you'll see**: a 20-minute planning session that ends with "wait, actually the problem isn't X, it's Y". Lost time, lost cache, user frustration.

**What it does**: enforces a **three-layer input review** BEFORE entering Plan Mode:

1. **Is the problem real?** — Validate the user's attribution and premises. Normal behavior often gets reported as bugs.
2. **Is information sufficient?** — Tag gaps as `P0 must-have` / `P1 nice-to-have` / `P2 bonus`. Don't proceed past P0 gaps with assumptions.
3. **Hidden issues?** — Is there a deeper root cause the user missed?

Plus `Confident / Likely / Unclear` tags on every plan hypothesis, a hard cap of 3 revision rounds, and explicit *remove / downgrade / defer* callouts when iterating.

**When it triggers**: "architect", "design", "refactor", "migrate", "new feature", "how should I approach", or multi-file change requests. Skips trivial fixes.

[→ View SKILL.md](./plugins/survival-kit/skills/plan-mode-review/SKILL.md)

---

### 3. `context-cost-guard` — Understand why your session got expensive

**Symptom you'll see**: "I only talked for 3 hours, why is the bill $150?" Or: "I stepped away for 30 min, the next message cost 10×".

**What it does**: makes the 4-source token accumulation model explicit:

1. **System prompt** (~11K, fixed)
2. **Conversation history** (linear growth — never auto-evicts)
3. **Tool results** (block growth — `Read` 700 lines = ~8K, all stays)
4. **Skill triggers** (one-shot ~4K per skill)

Then the cost formula: `cache_read × existing_context + cache_creation × new_this_turn + input + output`. Why `cache_read` is NOT free (~$1.50/M on a 300K context still adds up every turn). Why the **5-minute cache TTL** explains the "stepped away and now it's expensive" problem. The 200K/300K/400K compact thresholds. Four avoidance techniques (sub-agent outsourcing, Grep+offset, Plan Mode auto-clear, task segmentation).

**When it triggers**: "why so expensive", "should I compact", "context too large", or observable rapid context growth.

[→ View SKILL.md](./plugins/survival-kit/skills/context-cost-guard/SKILL.md)

---

## Evidence

The anchor session: **one 2h session, 96 `opus` calls, peak 365K context, $115+ bill**. This is the session that originally forced the cost model in rule #3 into existence.

Savings from applying these rules (against the mechanism each rule targets):

| Lever | Mechanism | Savings |
|---|---|---|
| `subagent-matrix` downgrade paths | Model downgrade ≈ 3-5× cost cut per call; most sub-agents shouldn't be opus | 40-60% of sub-agent cost |
| `context-cost-guard` 4-source awareness | Sub-agent outsourcing + Grep+offset + segmented sessions | 20-30% of main-session context |
| `plan-mode-review` 3-layer review | Catches ~30% of misdiagnosed problems before planning starts | Eliminates re-work cycles |

The $115 anchor is a real, tracked session. The goal isn't to sell a number — it's to give you the mental model so you can run your own math.

> 📏 **On screenshots**: no before/after bill images by design. Single-session variance is too high for any such image to be fair; the math does the honest work instead.

---

## Roadmap

**v1.0 (this release)**: 3 flagship auto-triggering skills. Enough to prevent the top 3 cost/drift leaks.

**v1.1 (coming soon)**: 5 additional manual-copy rules and reference docs:

- `methodology-vs-tool` — when NOT to install a plugin; the three-path evaluation (install / skip / extract methodology)
- `tool-minimalism` — why "should I add X" deserves a rising bar over time, not a lower one
- `auto-vs-manual-triggering` — classifying which automation belongs in rules vs which should stay user-invoked
- `skill-as-loader-pattern` — when a SKILL.md should be a 3KB index instead of a 30KB body
- `verify-before-deny` — why rules lag behind client updates and how to keep the assistant honest

Watch this repo (★ + Watch → Releases only) to get pinged when v1.1 lands.

---

## FAQ

**Q: Why just 3 skills? Other repos have 20+.**

> Because 3 is the right number of auto-triggering skills. Each resident skill costs system-prompt tokens in every session. Piling skills into a repo to pad count is the exact behavior these rules exist to correct. `skill-as-loader-pattern` (in v1.1) goes deeper on this — the short version: your plugin's token cost is a feature you ship to users, not free.

**Q: Is this for solo devs or teams?**

> Solo devs and individual contributors. Everything here is about managing your own Claude Code session cost and quality. Team-level agent orchestration isn't the lane.

**Q: Will this conflict with `cc-sdd` / `karpathy-skills` / other installed skills?**

> No. The three skills here trigger on orthogonal keywords (sub-agent dispatch, plan mode, context cost). Run them alongside anything.

**Q: Does the `~80%` savings claim hold up?**

> The math is in the [Evidence](#evidence) section above. If your sub-agents are already mostly `sonnet+high`, your sessions stay under 200K, and your plans never misdiagnose, you're already past the typical leak — these rules just won't do much for you.

**Q: How do I contribute a rule?**

> Open an issue with: (1) the symptom, (2) the mechanism, (3) at least one real-world incident where the rule would have saved you time/money. Rules without incident data get closed — "would be nice" is how this kind of repo bloats into noise.

---

## Credits & Prior Art

This repo builds on — and explicitly does not overlap with:

- `karpathy-skills` for coding principles
- `cc-sdd` for spec-driven workflow
- `best-practice` for tutorial depth
- Claude Code's native `Plan Mode` and `Skill` systems (these rules sit on top of them, not around them)

Numbers and thresholds (200K/300K/400K) reference community work — notably Thariq's April 2026 analysis on context rot.

---

## License

MIT. See [LICENSE](./LICENSE).

Notes from the field are authored from practitioner experience and are not affiliated with Anthropic.
