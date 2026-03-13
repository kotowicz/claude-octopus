---
name: blind-debate
user-invocable: true
aliases:
  - blind-debate
  - blind
  - peer-review
  - independent-debate
description: "Academic blind peer review across Claude and Codex, with optional Gemini. All participants receive the identical blind prompt, write independent papers in separate sessions, review every other participant, revise using both critiques received and reviews authored, and iterate until convergence into one merged answer."
context: fork
metadata:
  author: Octo
  version: 2.3.0
  category: workflow-automation
  tags: [debate, peer-review, multi-model, convergence, synthesis]
---

# Blind Debate

## Design Principle

This skill models academic peer review, not reveal-and-react debate.

Two learning channels are mandatory in every round:
1. Each participant receives critiques of their own paper.
2. Each participant changes their own thinking by reviewing other papers.

The revision step must use both channels. That reflexive mechanism is the core of the workflow.

## Non-Negotiable Rules

- Claude and Codex are required participants.
- Gemini is optional and may join only when available, compatible with the required headless flags, and not excluded.
- Every participant must receive the exact same blind prompt.
- Claude must produce blind, review, and revision artifacts through separate `claude --print` invocations rather than the orchestrator's in-thread reasoning.
- No participant may see another participant's work during the blind phase.
- In every round, each participant must write a separate structured review of every other participant's latest paper.
- Every peer review must include a `SELF-REFLECTION` section capturing what the reviewer learned from doing the review.
- All review files for a round must be written before any revision file for that round.
- Every revision prompt must include both the reviews received and the reviews authored by that participant in the same round.
- `--rounds` is a maximum, not a fixed script. Stop early only when the convergence gate passes.
- `merged-answer.md` is the primary artifact and must be one merged answer, not a comparison table.
- Preserve genuine trade-offs in `synthesis.md` instead of flattening them away.

## Use This Skill When

Use this skill when the user explicitly asks for blind debate, independent ideation before comparison, pairwise review across models, or `/octo:blind-debate`.

Use standard `/octo:debate` instead when the user wants immediate interaction rather than blind independence plus peer review.

## Instructions

### Step 1: Parse inputs and verify providers

Support these flags:
- `--rounds` / `-r` with default `3`
- `--max-words` / `-w` with default `400`
- `--topic` / `-t`
- `--out-dir` / `-o`
- `--synthesize` / `-s`
- `--convergence-threshold` with default `0.75`
- `--no-convergence-check`
- `--no-gemini`

Run the provider and compatibility checks from `references/workflow.md` before any execution work.

`claude --print` and `codex exec --full-auto` are mandatory. If either CLI is unavailable or lacks the required non-interactive flag, stop and suggest `/octo:setup` instead of degrading the workflow.

Gemini is optional. Include it only when the CLI exists, supports `-p "" -o text --approval-mode yolo`, and the user did not pass `--no-gemini`.

Display the status banner from `references/workflow.md` before creating files.

### Step 2: Create the debate directory and save state

Use the base directory from `references/workflow.md` unless `--out-dir` overrides it.

Create:
- `context.md`
- `state.json`
- `rounds/`

Record the topic, UTC timestamp, active participants, dropped participants, max rounds, max words, convergence threshold, current round, and status.

### Step 3: Run the blind phase

Build one blind prompt and send it unchanged to every active participant.

Launch Codex and optional Gemini in parallel. Start Claude's separate blind session with the same prompt before reading any external blind file. Save each blind paper separately.

Do not read any blind output until every required blind file exists, is non-empty, and passes the artifact-validation rules in `references/workflow.md`.

If Gemini fails or does not support the required headless flags, remove it from the active participant set, record that in `state.json`, and continue with Claude plus Codex.

If Claude or Codex fails, rerun once with the same prompt and stop if the retry still fails.

### Step 4: Run peer-review and revision rounds

For each round from `1` to `MAX_ROUNDS`:

#### 4.1 Review stage

Every participant writes a structured review of every other participant's latest paper.

With Claude, Codex, and Gemini active, this produces six ordered reviews per round. With only Claude and Codex active, this produces two ordered reviews per round.

Each review must:
- preserve the strongest ideas worth keeping
- identify major concerns, blind spots, or unsupported assumptions
- identify missing nuance, edge cases, or implementation detail
- recommend concrete revisions for the author
- state what writing the review changed in the reviewer's own thinking

Use the round-1 review prompt for the first round and the later-round review prompt for rounds `2+`.

Write all review files before any revision file.

#### 4.2 Revision stage

After all reviews exist, each participant revises their own latest paper.

Each revision prompt must include:
- the participant's own latest paper
- all reviews received on that paper in the current round
- all reviews that participant authored in the current round

This is mandatory. The revision must reflect both criticism received and insights gained from reviewing others.

Revision output must follow the structured format from `references/workflow.md`.

#### 4.3 Convergence gate

After each revision stage, compare the latest revised papers.

Compute:
- the primary recommendation of each paper
- the largest cluster of substantially aligned recommendations
- `convergence_score = cluster_size / participant_count`

Save the decision to `rounds/rNNN_convergence.md` and append the same decision summary to `state.json`.

Stop early only when:
- `convergence_score >= --convergence-threshold` (default `0.75`)
- no new blocking contradiction appears in the latest review files
- remaining differences are caveats, framing, or implementation detail rather than competing core recommendations
- the latest revisions are refining rather than introducing new core positions

If `--no-convergence-check` is set, still compute and persist the convergence assessment, but continue until the round cap unless a hard failure occurs.

### Step 5: Write the merged answer and synthesis

Write `merged-answer.md` as one integrated answer that combines the strongest validated ideas from the final revised papers into a single coherent recommendation.

Write `synthesis.md` as the process trace: initial diversity, strongest critiques, reflexive changes, convergence by round, unresolved trade-offs, decision criteria, and dissenting views worth preserving.

Reference `merged-answer.md` instead of duplicating it.

### Step 6: Optional follow-on artifact

If `--synthesize` was requested, derive `deliverable.md` from `merged-answer.md` and `synthesis.md`.

## Common Issues

- If Gemini is unavailable or incompatible, continue with Claude and Codex and record the drop in `state.json`.
- If Claude or Codex CLI is unavailable, stop and suggest `/octo:setup`.
- If a provider returns an empty blind, review, or revision file, rerun that specific step once.
- If a file contains runner wrapper text or is missing the required section headings, rerun once with stricter prompt hygiene. If rerun still emits deterministic wrapper lines, trim only the obvious non-content wrapper before saving the final artifact.
- If convergence is not reached by the round cap, still write `merged-answer.md` and clearly mark unresolved tensions in `synthesis.md`.
- If the topic is ambiguous, ask one clarifying question before creating the debate directory.

## Integration

- `/octo:debate` for standard immediate back-and-forth
- `/octo:docs` for document export after synthesis
- `/octo:embrace` for front-loading a blind debate into a broader workflow
