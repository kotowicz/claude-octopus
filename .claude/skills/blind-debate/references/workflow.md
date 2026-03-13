# Blind Debate Workflow Reference

Use this file for exact CLI patterns, compatibility checks, prompt templates, output layout, state tracking, artifact validation, convergence scoring, and the quality checklist.

## Output Layout

Store each run under:

```text
${HOME}/.claude-octopus/blind-debates/${CLAUDE_CODE_SESSION:-local}/${DEBATE_ID}/
```

Suggested `DEBATE_ID` format:

```text
blind-debate-YYYYMMDDTHHMMSSZ
```

Required artifacts:

- `context.md`
- `state.json`
- `rounds/r000_blind_claude.md`
- `rounds/r000_blind_codex.md`
- `rounds/r000_blind_gemini.md` when Gemini is active
- `rounds/r001_review_claude_of_codex.md`
- `rounds/r001_review_codex_of_claude.md`
- `rounds/r001_review_claude_of_gemini.md` when Gemini is active
- `rounds/r001_review_gemini_of_claude.md` when Gemini is active
- `rounds/r001_review_codex_of_gemini.md` when Gemini is active
- `rounds/r001_review_gemini_of_codex.md` when Gemini is active
- `rounds/r001_revised_claude.md`
- `rounds/r001_revised_codex.md`
- `rounds/r001_revised_gemini.md` when Gemini is active
- `rounds/r001_convergence.md`
- later-round review, revision, and convergence files following the same naming pattern
- `merged-answer.md`
- `synthesis.md`
- `deliverable.md` when `--synthesize` is requested

## Status Banner

Display this before any setup or execution work:

```text
CLAUDE OCTOPUS ACTIVATED - Blind Debate
Topic: [question]
Mode: Blind papers -> Explicit peer review -> Reflexive revision -> Convergence gate -> Merged answer
Participants: Claude, Codex, [Gemini]
Max rounds: [n]
Convergence threshold: [threshold]
```

## Compatibility Checks

Run these probes before the blind phase:

```bash
claude_available=$(command -v claude >/dev/null 2>&1 && echo "yes" || echo "no")
codex_available=$(command -v codex >/dev/null 2>&1 && echo "yes" || echo "no")
gemini_available=$(command -v gemini >/dev/null 2>&1 && echo "yes" || echo "no")

claude_print=$(
  if [[ "$claude_available" == "yes" ]] && claude --help 2>&1 | grep -F -- '--print' >/dev/null 2>&1; then
    echo "yes"
  else
    echo "no"
  fi
)

codex_full_auto=$(
  if [[ "$codex_available" == "yes" ]] && codex exec --help 2>&1 | grep -F -- '--full-auto' >/dev/null 2>&1; then
    echo "yes"
  else
    echo "no"
  fi
)

gemini_headless=$(
  if [[ "$gemini_available" == "yes" ]] && gemini --help 2>&1 | grep -F -- '--approval-mode' >/dev/null 2>&1; then
    echo "yes"
  else
    echo "no"
  fi
)
```

Stop if `claude_available != yes`, `claude_print != yes`, `codex_available != yes`, or `codex_full_auto != yes`.

Treat Gemini as optional. If `gemini_available != yes` or `gemini_headless != yes`, exclude Gemini from the run and record the reason in `state.json`.

## state.json Schema

Use `state.json` to track progress:

```json
{
  "debate_id": "blind-debate-20260313T100000Z",
  "question": "string",
  "created_at_utc": "2026-03-13T10:00:00Z",
  "participants": ["claude", "codex", "gemini"],
  "active_participants": ["claude", "codex"],
  "max_rounds": 3,
  "max_words": 400,
  "convergence_threshold": 0.75,
  "current_round": 0,
  "status": "blind",
  "convergence_history": [],
  "dropped": [
    {"participant": "gemini", "reason": "missing --approval-mode support", "round": 0}
  ]
}
```

Append one object to `convergence_history` after each round:

```json
{
  "round": 1,
  "primary_recommendations": {
    "claude": "string",
    "codex": "string",
    "gemini": "string"
  },
  "cluster_size": 2,
  "participant_count": 3,
  "convergence_score": 0.67,
  "blocking_objections": ["string"],
  "decision": "continue",
  "rationale": "string"
}
```

## CLI Patterns

Claude blind, review, or revision execution:

```bash
claude --print "$PROMPT"
```

Codex blind, review, or revision execution:

```bash
codex exec --full-auto "$PROMPT"
```

Gemini blind, review, or revision execution:

```bash
printf '%s' "$PROMPT" | gemini -p "" -o text --approval-mode yolo
```

## Prompt Hygiene Footer

Append this footer to every blind, review, and revision prompt:

```text
Return ONLY the requested Markdown body. Do not include preambles, code fences, tool logs, or commentary outside the document.
```

## Blind Prompt

Send this identical prompt to every active participant without per-agent edits:

```text
You are participating in a blind academic peer-review study.

Multiple teams are answering the same question independently and will not see each other's papers until the blind collection phase is complete.

Write an independent paper answering the question below. Do not assume what the other participants will say. Do not pre-compromise. Commit to your best current answer with clear reasoning.

Address the high-priority concerns thoroughly. Also include the lower-priority nuances you believe still matter.

Word limit: [MAX_WORDS] words.

QUESTION:
[QUESTION]

Return ONLY the requested Markdown body. Do not include preambles, code fences, tool logs, or commentary outside the document.
```

## Blind Phase Execution

Launch Codex and optional Gemini in parallel. Start Claude's blind paper before reading any external blind file:

```bash
codex exec --full-auto "$BLIND_PROMPT" > "${DEBATE_DIR}/rounds/r000_blind_codex.md" &
codex_pid=$!

if [[ "${USE_GEMINI}" == "yes" ]]; then
  printf '%s' "$BLIND_PROMPT" | gemini -p "" -o text --approval-mode yolo > "${DEBATE_DIR}/rounds/r000_blind_gemini.md" &
  gemini_pid=$!
fi

claude --print "$BLIND_PROMPT" > "${DEBATE_DIR}/rounds/r000_blind_claude.md"

wait "$codex_pid"

if [[ "${USE_GEMINI}" == "yes" ]]; then
  wait "$gemini_pid" || USE_GEMINI="no"
fi
```

Validation rules:
- do not read any blind file until every required blind file exists and is non-empty
- if Claude or Codex fails, rerun once with the same prompt, then stop if the retry also fails
- if Gemini fails, drop it from `active_participants`, append a `dropped` entry in `state.json`, and continue
- every blind file must pass the artifact-validation rules below before it is treated as a paper

## Peer Review Prompt (Round 1)

Each participant writes a review of each other participant's latest paper. The reviewer sees their own paper so they can compare approaches:

```text
You are conducting academic peer review. Below is your own paper and another participant's paper on the same question.

QUESTION:
[QUESTION]

YOUR CURRENT PAPER:
[REVIEWER_LATEST]

PAPER UNDER REVIEW (by [AUTHOR_NAME]):
[AUTHOR_LATEST]

Write a structured review with these sections:

1. STRENGTHS: What does this paper get right and what should survive into any final answer?
2. CONCERNS: What assumptions, errors, or risks need to be challenged?
3. MISSING NUANCE: What important edge cases, caveats, or implementation details are absent?
4. RECOMMENDED REVISIONS: What concrete changes should the author make?
5. SELF-REFLECTION: What did reviewing this paper change in how you think about your own paper?

Be rigorous but constructive. The goal is to improve the final answer, not to win.

Word limit: [MAX_WORDS] words.

Return ONLY the requested Markdown body. Do not include preambles, code fences, tool logs, or commentary outside the document.
```

## Peer Review Prompt (Rounds 2+)

For later rounds, shift attention toward remaining disagreements and merge opportunities:

```text
This is peer-review round [ROUND]. All participants have already reviewed and revised at least once.

QUESTION:
[QUESTION]

YOUR CURRENT PAPER:
[REVIEWER_LATEST]

PAPER UNDER REVIEW (by [AUTHOR_NAME], revision [ROUND-1]):
[AUTHOR_LATEST]

Write a structured review with these sections:

1. REMAINING GAPS: What important issues are still unresolved?
2. STRONGEST ELEMENT: What is the single best idea in this paper that must survive?
3. DISAGREEMENTS: Where do you still genuinely disagree, and what evidence would change your mind?
4. MERGE OPPORTUNITIES: What from this paper and yours could be combined into something better?
5. SELF-REFLECTION: What would you now change in your own paper after reviewing this revision?

Word limit: [MAX_WORDS] words.

Return ONLY the requested Markdown body. Do not include preambles, code fences, tool logs, or commentary outside the document.
```

## Revision Prompt

After all reviews are written, each participant receives:
- their own prior paper
- the reviews others wrote about that paper in the current round
- the reviews they wrote about other papers in the current round

```text
You submitted a paper and also served as peer reviewer for the other participants. Now revise your own paper.

QUESTION:
[QUESTION]

YOUR PRIOR PAPER:
[THIS_AGENT_PREVIOUS]

REVIEWS YOU RECEIVED THIS ROUND:
[REVIEWS_RECEIVED]

REVIEWS YOU AUTHORED OF OTHER PAPERS THIS ROUND:
[REVIEWS_AUTHORED]

Tasks:
1. Identify the critiques of your own paper that matter most.
2. Identify what you learned by reviewing the other papers.
3. Explain what changed in your thinking.
4. Write your revised paper.
5. Note any unresolved trade-offs that still remain.

Return these sections:
- Change Log
- What Peer Review Changed For Me
- Revised Paper
- Remaining Open Questions

Word limit: [MAX_WORDS] words.

Return ONLY the requested Markdown body. Do not include preambles, code fences, tool logs, or commentary outside the document.
```

## Reviewer Dispatch Pattern

Use the reviewer identity to choose the correct CLI:

```bash
case "$REVIEWER" in
  claude)
    claude --print "$REVIEW_PROMPT" > "$REVIEW_FILE"
    ;;
  codex)
    codex exec --full-auto "$REVIEW_PROMPT" > "$REVIEW_FILE"
    ;;
  gemini)
    printf '%s' "$REVIEW_PROMPT" | gemini -p "" -o text --approval-mode yolo > "$REVIEW_FILE"
    ;;
