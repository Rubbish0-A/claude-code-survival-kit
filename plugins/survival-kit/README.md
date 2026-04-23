# survival-kit

Three auto-triggering skills for Claude Code to prevent cost blowups and planning drift.

## Skills

- **[subagent-matrix](./skills/subagent-matrix/SKILL.md)** — Model × effort selection with tripwire escalation. Stops the default "everything is opus + max" pattern.
- **[plan-mode-review](./skills/plan-mode-review/SKILL.md)** — Three-layer input review before Plan Mode enters. Catches misdiagnosed problems early.
- **[context-cost-guard](./skills/context-cost-guard/SKILL.md)** — 4-source accumulation model, cache TTL mechanics, compact thresholds.

## Design principle

Each skill's frontmatter `description` is the auto-trigger contract. The assistant reads the description — not the body — to decide whether to activate. The bodies are kept under 120 lines each to minimize resident token cost once activated.

Details and evidence in the top-level [README](../../README.md).
