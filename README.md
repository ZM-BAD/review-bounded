# review-bounded — Three-Round Bounded Code Review Skill

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

**Contents**

- [What it is](#what-it-is)
- [Why AI review never ends](#why-ai-review-never-ends)
- [Why "bounded"?](#why-bounded)
- [The point: lighter issues, not zero issues](#the-point-lighter-issues-not-zero-issues)
- [Quick start](#quick-start)
- [What the verdict looks like](#what-the-verdict-looks-like)
- [How it works](#how-it-works)
- [FAQ](#faq)
- [Directory layout](#directory-layout)
- [Design principles](#design-principles)
- [License](#license)

## What it is

review-bounded is an [Agent Skills](https://agentskills.io/specification) (SKILL.md) package. It orchestrates **exactly three** review-and-fix rounds on your current branch: every issue graded on one fixed P0–P3 scale, every fix requiring your authorization, and a convergence verdict at the end.

Other bounded-review tools are platform plugins, sub-agent primitives, or CLI tools. review-bounded, by contrast, is a **Skill**: pure SKILL.md, zero runtime. Any agent that can read a SKILL.md can run it.

## Why AI review never ends

```
"review the changes" → finds 123 issues → fix
"review again"       → finds 456 issues → fix
"review again"       → finds 789 issues → fix
"review again"       → ...
```

The loop starts with **engineering discipline**. After the AI finds issues and fixes them, any careful engineer wants to verify the fixes — you never ship a fix you haven't checked. Code review is an important way to verify a fix. So the next step is natural: review the fixes again — and find new issues.

The next fix calls for the same verification, and so on.

AI code review **has basically no "zero issues" endpoint** — and that is not a flaw of any particular model, it is structural:

- **The task has no termination condition.** "Review this code" means "find problems", not "confirm it is good enough". A human reviewer stops out of fatigue, time pressure, or trust; an LLM has none of these — every call is an independent sample with the same open-ended goal.
- **The training distribution rewards finding problems.** Code snippets labeled as buggy carry the highest information weight, so the model statistically learns that "finding issues" is high-value output, while "this code is fine" is low-value noise. LLMs have learned what human reviewers learned long ago: the one who only ever says LGTM (Looks Good To Me) soon stops getting invited to reviews.
- **Every round re-reads the full code, not just the fix.** The input of each review is the whole codebase after the fixes — never "verify only the last round's issues" — so a new pass can always surface something the previous one missed.

So "review again" will always find more issues. The real question is not *whether* the review finds issues — it is *how severe* the remaining issues are, and whether they are light enough to accept.

## Why "bounded"?

Because the most direct and reliable way to stop an infinite loop is to commit to a number in advance: three rounds, no more, no less.

## The point: lighter issues, not zero issues

AI review never reaches zero — but it **converges**: severity drops round over round.

```
Round 1:  deadlocks, data loss, security holes      (P0/P1)
Round 2:  performance, robustness, error handling   (P2)
Round 3:  naming, dead code, style                  (P3)
```

This is not luck: findings are drawn from a **Zipf-like distribution** — the most salient issues (deadlocks, data loss, injection) come first, and later rounds are left sampling the long tail (naming, dead code, style). Severity drops because the same distribution is sampled in sequence — not because the model gets smarter or more tired. Note: plateauing also counts as converging — the verdict only requires no rise; "drops round over round" is the typical trajectory, not a hard requirement.

The purpose of code review is **not** to confirm "there are absolutely no problems" (that is unlikely anyway). It is to confirm that the issues found are **light enough to be acceptable** — backed by data, not by feeling.

This skill makes convergence measurable and "stop" a decision you can stand behind. These four things are exactly what other code review plugins/extensions rarely give you:

1. **Exactly three rounds, no more, no less** — a fixed count, so "review again" can never become an infinite loop. Reviewing once would leave you uneasy — wouldn't you at least run a regression check? But review forever, and where does it end?
2. **One fixed scale (P0–P3) across all three rounds** — "are the issues getting lighter?" becomes numbers, not feelings, with the P1→P2→P3 trend visible at a glance;
3. **Authorization before every round's fixes** — the AI proposes, you decide; nothing changes in your code without your say-so, and nothing is auto-fixed, ever;
4. **A CONVERGED / NOT CONVERGED verdict** — the AI is about to say "looks fine, ready to merge", but the rules make it argue against itself first: list the reasons not to merge. Solid, well-founded reasons mean there are still problems; only trivial nitpicks not worth mentioning mean it really is mergeable. If no solid counter-case can be raised, that is when the code is truly spotless.

## Quick start

Requirements: any coding agent that supports the Agent Skills standard (Claude Code, Codex, ZCode, and others).

**Install** — one command:

```sh
npx skills add ZM-BAD/review-bounded
```

**Trigger** — in your agent session:

- ZCode / Claude Code: `/review-bounded`, or just say "run three rounds of CR" / "do a bounded review on this branch";
- Codex: natural language — "run three rounds of bounded code review on this branch" (no slash command).

At the start, you pick the review scope — current code vs the base branch, uncommitted changes only, the branch's recent commits, or a scope you name yourself. Then the skill runs exactly three rounds and returns a verdict.

> Interactive use only: per-round authorization and the commit decision require a human in the loop; non-interactive modes (e.g. one-shot `-p` mode, CI) are not supported.

## What the verdict looks like

```text
Verdict: CONVERGED

Meta-question 1 (severity trend): top levels P1→P2→P3 across rounds, P0+P1 counts 2→1→0
Meta-question 2 (fix regressions): none detected — all three rounds' tests passed + diff trace
Meta-question 3 (last-round issues): R3-1 (P2, boundary): breaks only on malformed input

The case against merging:
1. New cache module has no eviction path (code evidence)
2. E2E not run (test evidence)
3. Two helper names drift from convention (speculation)

Production impact of last-round issues:
- Would definitely break: none
- Might break: malformed input (R3-1 scenario)
- Would not break: normal input path

Unresolved items (unauthorized / unfixed P2/P3):
- [ ] R2-2: retry logic without backoff (suggested handling)
```

**CONVERGED** means the last round's issues were light, no fix-introduced regression was found, and everything still open is explicitly listed. **NOT CONVERGED** means the last review still found real problems that shouldn't be ignored — continuing review-and-fix is the right call.

## How it works

1. **Round 1**: review the user-confirmed scope (e.g., the branch's own changes vs the base branch), report findings (P0–P3), ask for authorization, apply fixes to the working tree (no commits yet), run regression checks (tests/lint when available).
2. **Round 2**: re-read the latest code (never from memory), prioritize the previous fixes' deltas and previously-uncovered categories, dedupe against round 1, then the same report → authorize → fix → verify cycle.
3. **Round 3**: same cycle, then the final verdict. All fixes are still uncommitted — the skill then asks whether you want to commit them and how (one commit or grouped by finding).

Every round reviews the **actual latest code** via `git diff` and fresh file reads — never a reconstruction from memory. Fixes are applied to the working tree only and **never committed during the process**; after the final verdict, the skill presents the full fix list, and you decide whether to commit and how. Findings are recorded in the conversation; no extra files are produced (on request, the agent writes a summary to `REVIEW.md`).

## FAQ

- **Is three rounds enough?** The verdict answers this: CONVERGED means the remaining issues are light and nothing is hidden; NOT CONVERGED spells out exactly which problems remain.
- **Does it support my language/framework?** It's a language/framework-agnostic skill, not a linter: whatever the agent can read — code, tests, docs — it can review. No runtime, no dependencies.
- **Are there not already bounded-review tools?** Yes — Convergo, adversarial-review, and others. The existence of these tools validates the direction. The difference: those are platform-bound (plugins, sub-agent primitives, two-CLI setups) and mostly auto-fix without per-round authorization; this one is a **pure Skill** with zero platform lock-in, installable with one command.
- **Do tiny changes need all three rounds?** No — a single-line or pure-docs change is one round.

## Directory layout

```
SKILL.md                  # orchestration: rounds, recording, hard rules
references/
  rubric.md               # P0–P3 severity scale + review categories (loaded before round 1)
  round-prompt.md         # per-round instructions: fresh code, dedupe, evidence, authorization, regression check
  decision-rules.md       # final verdict rules + the case against merging + output format (English + Chinese templates)
```

## Design principles

- **Orchestration layer, not execution**: the agent's own tools (git, file reads, bash) do the work, executing the built-in instructions (`references/round-prompt.md`) directly — no delegation to other review skills.
- **Zero runtime**: pure Markdown, no scripts, no dependencies, no APIs — portable to any agent that supports the SKILL.md standard.
- **Structural constraints are hard-coded**: fixed rubric before round 1, per-round authorization, "uncertain → NOT CONVERGED", argue against the merge before any verdict.
- **Fresh code every round**: memory is never a substitute for reading the actual repository state.

⭐ Found this useful? Star the repo and share it with a team stuck in the review loop.

## License

[MIT](LICENSE)
