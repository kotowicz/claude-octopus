---
command: blind-debate
description: "Blind multi-model academic peer review with identical prompts, reflexive revision, convergence gating, and a merged final answer"
skill: blind-debate
aliases:
  - blind
  - peer-review-debate
---

# Blind Debate

Use the `blind-debate` skill for this command.

When the user invokes `/octo:blind-debate` or `/blind-debate`, execute the blind peer-review workflow instead of answering directly or falling back to `/octo:debate`.

Execution contract:
1. Parse the question and supported flags.
2. Create the run directory under `~/.claude-octopus/blind-debates/`.
3. Start separate blind sessions for Claude and Codex with the identical prompt. Add Gemini only when available and not disabled.
4. Write all blind papers before revealing any blind output.
5. For each round, write all ordered peer reviews, then all revisions, then the convergence assessment.
6. Stop only when the convergence gate passes or the max round cap is reached.
7. Write `merged-answer.md` as the primary deliverable and `synthesis.md` as the process trace.
8. Report the output directory to the user.

Ask one clarifying question only when the topic or constraints are genuinely ambiguous.
