# Next-Generation AI Operating System Prompt

This repository contains a full reverse-engineering of a production Claude Fable 5
(claude.ai consumer) system prompt and the design of its successor: a layered,
model-agnostic "AI Operating System" prompt optimized for long-horizon agentic work,
large-codebase software engineering, and research — while remaining tuned for
Claude Fable 5.

## Contents

| File | Deliverable |
|------|-------------|
| `01-architecture-analysis.md` | Part 1 — Architectural analysis of the source prompt; Part 2 — Weakness report; Part 3 — Improvement strategy |
| `02-universal-master-os-prompt.md` | Part 4 — Universal Master AI Operating System Prompt (model-agnostic, deployable on Claude, GPT-5, Codex, Gemini CLI, Antigravity, RooCode, OpenHands, Aider, Cline) |
| `03-claude-fable-5-os-prompt.md` | Part 5 — Claude Fable 5 Optimized AI Operating System Prompt |

## Design summary

The source prompt is a ~42k-token monolith that succeeds through five techniques:
decision procedures instead of abstract values, deliberate repetition of hard limits,
progressive disclosure via skills, paired positive/negative examples, and
meta-cognitive tripwires. It fails through uncontrolled duplication (~15–20% token
overhead), rule conflicts produced by copy drift, hardcoded volatile facts, missing
agentic subsystems (no verify/recover loop, no long-horizon state, no engineering
standards), and a structure that cannot be reused outside one product surface.

The successor keeps the five techniques, deletes the failure modes, and adds the
missing subsystems as composable modules on a small kernel with an explicit
precedence order:

```
L0  Kernel        identity, precedence, safety invariants        (always loaded)
L1  Reasoning     decomposition, planning, verification, recovery (always loaded)
L2  Modules       swe / research / files / connectors / memory   (per deployment)
L3  Tool layer    routing rules; schemas live in the harness      (per deployment)
L4  Session       date, user context, environment facts           (per session)
TAIL Recap        hard-limit anchors in the recency window         (always loaded)
```

Safety content is preserved at full strength — deduplicated, not weakened.
