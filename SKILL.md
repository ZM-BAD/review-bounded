---
name: review-bounded
description: 'Run exactly three rounds of bounded code review on the current branch. Each round re-reads the latest code, grades findings on a fixed P0-P3 severity rubric, reports them, asks for user authorization before fixing, and leaves fixes uncommitted in the working tree. After round three, output a per-round findings-and-fixes summary, answer the three convergence meta-questions, and give a CONVERGED / NOT CONVERGED verdict plus a list of unresolved items. Use when feature development is done and ready to merge, or when the user asks for a three-round / bounded / convergence code review (also in Chinese: "跑三轮 CR", "有界审查", "收敛性审查").'
---

# Review Bounded — Three-Round Bounded Code Review

## Positioning

This skill is the **orchestration layer** of code review; it does not re-implement the execution details:

- Execution uses the agent's own tools (git, file reads, Bash) and the project's existing tests / lint / static checks;
- If another review skill is installed in the environment (e.g., code-review, pr-review-toolkit), individual rounds may be delegated to it; this skill owns round orchestration, recording, and the convergence verdict;
- With no review skill installed, execute the built-in instructions in `references/round-prompt.md` — no external services required.

## When to use / when not to use

- **Use**: feature branch development is complete and ready to merge; the user asks for "three rounds of CR" / "bounded review" / "check whether it converges" / "don't review forever".
- **Don't use**: single-line or pure-docs changes (one round is enough); the user explicitly asks for a quick single round; the user asks to review someone else's PR (this skill only reviews the current workspace/branch).

## Workflow

1. **Initialize**
   - Read `references/rubric.md` (the severity scale; must be loaded before round 1 and governs all later rounds);
   - Determine the review scope: the full diff from the base branch (main, or master if absent) to the current working tree (`git diff <base>`, including staged and unstaged changes); when working directly on the base branch, only the uncommitted changes; if there are uncommitted changes, ask whether to include them (default: include).

2. **Three-round loop** (each round executes the full instructions in `references/round-prompt.md`)
   Each round = restate the previous rounds' finding lists (confirm memory, calibrate dedupe) → define the round scope and **re-read the project's latest code** → review against the rubric → report findings and **ask for authorization** → fix after authorization (changes stay in the working tree, **no commits**) → regression check (run tests/lint and record evidence) → record the round's conclusion (in the conversation).

3. **Final output**
   Per `references/decision-rules.md`: summarize the three rounds' findings and fixes, answer the three meta-questions, output the prosecutor's brief first, then give the CONVERGED / NOT CONVERGED verdict, the production-impact grading of the last round's issues, and the list of unresolved items; then show the three rounds' fix lists and **let the user decide whether and how to commit** (one commit or grouped by finding).

## Recording

- All round records live **in the conversation**; no extra files are produced;
- For archiving, create a summary document only when the user says "write the conclusions to a file" (suggested: `REVIEW.md` in the repo root).

## Hard rules

- Each round must **re-read the project's latest code** (git diff + file reads); reconstructing the code state from memory is forbidden.
- Every finding is graded on the rubric; the scale must not drift between rounds.
- Cross-round dedupe: re-reporting a previous round's issue in different wording is not a new finding.
- Findings without evidence are not reported; "no regression" without evidence is not written.
- Authorization must be obtained before fixing.
- No commits during the three rounds; afterward the user decides whether and how to commit.
- When unsure whether it converged, always judge NOT CONVERGED.
- The final verdict must be preceded by the prosecutor's brief (the strongest reasons against merging).
- No agent-private features (hooks, slash-command syntax) — keeps the skill portable across agents.

## Related files

- `references/rubric.md` — severity scale and review categories (read before round 1)
- `references/round-prompt.md` — per-round review instruction template (read each round)
- `references/decision-rules.md` — convergence rules after round three (read in the final phase)
