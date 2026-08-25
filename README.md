# review-bounded

<p align="center">
  <a href="README.md">English</a> | <a href="README.zh-CN.md">简体中文</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License"/>
  <img src="https://img.shields.io/badge/AGENTS%20Skills%20Standard-compatible-purple?style=flat-square" alt="AGENTS Skills Standard"/>
</p>

> **Three rounds of AI code review with a fixed severity scale — so "should I review again?" becomes a data-driven decision instead of an endless loop.**
>
> **review → fix → verdict.** One skill, no build step, zero dependencies.

## What it is

review-bounded is an [Agent Skills](https://agentskills.io/specification) (SKILL.md) package. It orchestrates **exactly three** review-and-fix rounds on your current branch: every issue graded on one fixed P0–P3 scale, every fix requiring your authorization, and a convergence verdict at the end.

**Why "bounded"?** Because the only reliable way to stop an infinite loop is a budget you commit to in advance — three rounds, no more, no less.

## Why AI review never ends

```
"review the changes" → finds 123 issues → fix
"review again"       → finds 456 issues → fix
"review again"       → finds 789 issues → fix
"review again"       → ...
```

AI code review has **no "zero issues" endpoint**. That is not a flaw of any particular model — it is structural:

- **A reviewer that finds nothing has failed its job.** LLMs are trained to be helpful and thorough; when asked to review, producing findings *is* the rewarded behavior. Silence earns nothing.
- **Critique costs the model nothing.** A false positive doesn't hurt the model — it costs *you* a round-trip. A missed real bug is invisible to it. Findings are always cheap to emit.
- **Every pass is a fresh sample.** Each new review re-reads the code from a new angle; a new pass can always surface something the last one missed.
- **There is no ground truth to stop against.** Nothing in the process says "done". The loop ends only when a human decides to end it.

So "review again" will always find more issues. The real question is not *whether* the review finds issues — it is *how heavy* the remaining issues are.

## The point: lighter issues, not zero issues

AI review never reaches zero — but it **converges**: severity drops round over round.

```
Round 1:  deadlocks, data loss, security holes      (P0/P1)
Round 2:  performance, robustness, error handling   (P2)
Round 3:  naming, dead code, style                  (P3)
```

The purpose of code review is **not** to reach "no findings". It is to get the findings **light enough to be acceptable** — and to know, with evidence, when that point has been reached.

This skill makes convergence measurable and stopping a decision you can defend:

1. **Exactly three rounds** — a fixed budget, so "review again" cannot become an infinite loop;
2. **One severity scale (P0–P3) across all rounds** — "are the issues getting lighter?" becomes a number, not a feeling;
3. **Authorization before every fix** — the AI proposes, you decide; nothing changes in your code without your say-so;
4. **A CONVERGED / NOT CONVERGED verdict** — delivered with the "prosecutor's brief" (the strongest reasons not to merge), the production impact of the last round's issues, and the unresolved list.

## Quick start

Requirements: any coding agent that supports the Agent Skills standard (Claude Code, Codex, ZCode, and others).

**Install** — one command:

```sh
npx skills add ZM-BAD/review-bounded
```

**Trigger** — in your agent session:

- ZCode / Claude Code: `/review-bounded`, or just say "跑三轮 CR" / "run three rounds of bounded review on this branch";
- Codex: natural language — "run three rounds of bounded code review on this branch" (no slash command).

At start you pick the review scope — current code vs the base branch, uncommitted changes only, the branch's recent commits, or a scope you name yourself. Then the skill runs exactly three rounds and returns a verdict.

> Interactive use only: per-round authorization and the commit decision require a human in the loop; non-interactive modes (e.g. `-p` / CI) are not supported.

## What the verdict looks like

```text
Verdict: CONVERGED

Meta-question 1 (severity trend): top levels P1→P2→P3, P0+P1 counts 2→1→0
Meta-question 2 (fix regressions): none detected — tests passed + diff trace
Meta-question 3 (last-round issues): R3-1 (P2, boundary): breaks only on malformed input

Prosecutor's brief:
1. New cache module has no eviction path (code evidence)
2. E2E not run (test evidence)
3. Two helper names drift from convention (speculation)

Unresolved items:
- [ ] R2-2: retry logic without backoff (suggested handling)
```

**CONVERGED** means the last round's issues were light, no fix-introduced regression was found, and everything still open is explicitly listed. **NOT CONVERGED** means you know exactly what still worries the reviewer — and that continuing to review is the right call.

## How it works

1. **Round 1**: review the user-confirmed scope (e.g., the branch's own changes vs the base branch), report findings (P0–P3), ask for authorization, apply fixes to the working tree (no commits yet), run regression checks (tests/lint when available).
2. **Round 2**: re-read the latest code (never from memory), prioritize the previous fixes' deltas and previously-uncovered categories, dedupe against round 1, then the same report → authorize → fix → verify cycle.
3. **Round 3**: same cycle, then the final verdict. All fixes are still uncommitted — the skill then asks whether you want to commit them and how (one commit or grouped by finding).

Every round reviews the **actual latest code** via `git diff` and fresh file reads — never a reconstruction from memory. Fixes are applied to the working tree only and **never committed during the process**; after the final verdict, the skill presents the full fix list and you decide whether to commit and how. Findings are recorded in the conversation; no extra files are produced (on request, the agent writes a summary to `REVIEW.md`).

## FAQ

- **Do I still need a human reviewer?** Yes — that is the point. The AI reports, you authorize every fix, and the final commit decision is yours. The skill is interactive-only by design.
- **Will the AI just fix everything itself?** No. Nothing changes in your code without your authorization, and no commits happen during the three rounds.
- **Is three rounds enough?** The verdict answers this: CONVERGED means the remaining issues are light and nothing is hidden; NOT CONVERGED names exactly what still worries the reviewer — keep going if you agree.
- **Does it work with my language/framework?** It is pure instructions, not a linter: whatever the agent can read — code, tests, docs — it can review. No runtime, no dependencies.

## Directory layout

```
SKILL.md                  # orchestration: rounds, recording, hard rules
references/
  rubric.md               # P0–P3 severity scale + review categories (loaded before round 1)
  round-prompt.md         # per-round instructions: fresh code, dedupe, evidence, authorization, regression check
  decision-rules.md       # final verdict rules + prosecutor's brief + output format
```

## Design principles

- **Orchestration layer, not execution**: the agent's own tools (git, file reads, bash) do the work, executing the built-in instructions (`references/round-prompt.md`) directly — no delegation to other review skills.
- **Zero runtime**: pure Markdown, no scripts, no dependencies, no APIs — portable to any agent that supports the SKILL.md standard.
- **Structural constraints are hard-coded**: fixed rubric before round 1, per-round authorization, "uncertain → NOT CONVERGED", prosecutor's brief before the verdict.
- **Fresh code every round**: memory is never a substitute for reading the actual repository state.

⭐ Found this useful? Star the repo and share it with a team stuck in the review loop.

## License

[MIT](LICENSE)
