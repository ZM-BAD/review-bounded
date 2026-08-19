# review-bounded

**English** | [中文版](./README.zh-CN.md)

A bounded code review skill: **three rounds** of AI code review with a fixed severity rubric, per-round fix authorization, and a convergence verdict — so "should I review again?" becomes a data-driven decision instead of an endless loop.

## Problem

```
"review the changes" → issues 123 → fix → "review again" → issues 456 → fix → "review again" → issues 789 → ...
```

AI code review never reaches "zero issues" — but it does converge: issue severity decreases round over round. This skill runs exactly **three** review-and-fix rounds on your current branch, then reports:

1. What was found and what was fixed in each round, every issue graded P0–P3 on the same rubric;
2. Whether issues are getting weaker round over round (severity trend);
3. Whether the fixes introduced regressions (with evidence).

It ends with a **CONVERGED / NOT CONVERGED** verdict, a "prosecutor's brief" (the strongest reasons not to merge), and a list of unresolved items — so you know when to stop.

## How it works

1. **Round 1**: review the diff from the base branch (main/master) to the current working tree, report findings (P0–P3), ask for authorization, apply fixes to the working tree (no commits yet), run regression checks (tests/lint when available).
2. **Round 2**: re-read the latest code (never from memory), prioritize fix deltas and previously-uncovered categories, dedupe against round 1, then the same report → authorize → fix → verify cycle.
3. **Round 3**: same cycle, then the final verdict. All fixes are still uncommitted — the skill then asks whether you want to commit them and how (one commit or grouped by finding).

Every round reviews the **actual latest code** via `git diff` and fresh file reads — never a reconstruction from memory. Fixes are applied to the working tree only and **never committed during the process**; after the final verdict, the skill presents the full fix list and you decide whether to commit and how. Findings are recorded in the conversation; no extra files are produced (on request, the agent writes a summary to `REVIEW.md`).

## Directory layout

```
SKILL.md                  # orchestration: rounds, recording, hard rules
references/
  rubric.md               # P0–P3 severity scale + review categories (loaded before round 1)
  round-prompt.md         # per-round instructions: fresh code, dedupe, evidence, authorization, regression check
  decision-rules.md       # final verdict rules + prosecutor's brief + output format
```

## Design principles

- **Orchestration layer, not execution**: the agent's own tools (git, file reads, bash) do the work; other review skills (code-review, pr-review-toolkit) can be delegated to if installed; built-in instructions are the fallback.
- **Zero runtime**: pure Markdown, no scripts, no dependencies, no APIs — portable to any agent that supports the SKILL.md standard (`npx skills add` compatible).
- **Structural constraints are hard-coded**: fixed rubric before round 1, per-round authorization, "uncertain → NOT CONVERGED", prosecutor's brief before the verdict.
- **Fresh code every round**: memory is never a substitute for reading the actual repository state.

## Usage

- Install per the agent skills standard (e.g. `npx skills add <owner>/review-bounded` once published), or drop into your agent's skills directory.
- Trigger: "run three rounds of CR" / "bounded review on this branch".
- Scope: current branch vs base branch (main/master); on the base branch, the uncommitted working-tree changes.

## Test matrix (before publishing)

| Agent | 3 rounds complete | record format consistent | verdict correct |
|---|---|---|---|
| Claude Code | ☐ | ☐ | ☐ |
| Codex | ☐ | ☐ | ☐ |
| ZCode | ☐ | ☐ | ☐ |

How to verify: a mock branch with intentionally planted P0/P1/P2 issues; run the full flow and check the round records and the final verdict against manual review.
