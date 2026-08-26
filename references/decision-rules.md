# Convergence Verdict After Three Rounds

Execute after the three-round loop finishes. **Fully restate the three rounds' findings and fix lists before the verdict** (from the conversation record) — never answer from memory or impression.

## Output language

Output the whole report in the **user's conversation language**, detected from the user's messages — never default to English. A user writing in Simplified Chinese gets the Chinese template below (判定：收敛 / 判定：未收敛, 关键问题 1/2/3, 反对合并的理由, 代码证据 / 测试证据 / 推测, 未修复项); a user writing in English gets the English template. When the user mixes languages, follow their most recent messages. Use the fixed strings in the templates verbatim — do not paraphrase or re-translate them. Language-neutral parts (finding IDs like R2-1, severity levels P0-P3, file:line references) stay unchanged in both languages. Step 5's commit-decision questions and the delivery notes follow the same rule: a Chinese user is asked in Chinese — 这些修复要提交吗？（全部 / 部分 / 暂不）；如果要提交：一个 commit，还是按 finding 分组多个 commit？Step 1's summary labels (Scope/baseline, Findings, Fixes) localize the same way — Chinese: 范围/基准、发现、修复.

## Step 1: Output the three-round summary

Output in the following structure (this is the core deliverable to the user):

```
Round 1:
- Scope/baseline: <one line, e.g., merge-base main → working tree>
- Findings: R1-1 (P0, concurrency, x.go:42, deadlock) ...
- Fixes: x.go:42 (R1-1), y.go:18 (R1-2) ... (uncommitted)

Round 2:
- Scope/baseline: <one line>
- Findings: R2-1 (P1, error handling, y.go:18, ...) ...
- Fixes: ...

Round 3:
- Scope/baseline: <one line>
- Findings: R3-1 (P2, boundary, z.go:5, ...) ...
- Fixes: ...

All findings above are graded on the same scale (rubric P0-P3).
```

## Step 2: Answer the three meta-questions

1. **Severity trend**: does the top severity and the P0/P1 count drop round over round? Show the three-round comparison (top level / P0+P1 count / total findings).
2. **Fix regressions**: did the three rounds detect any "new bug introduced by a fix"? Cite evidence (test results, diff trace); when tests are unavailable and no trace was done, honestly write "not verified" — never "no regression".
3. **Last-round issues**: what are round three's issues? For each, what would break if it shipped today (give a concrete scenario)?

## Step 3: The case against merging (must be stated before the verdict)

List the 3 strongest reasons against "safe to merge now", each with:
- The reason itself;
- An evidence strength label: `code evidence` (directly visible in the diff) / `test evidence` (has test output) / `speculation` (no direct evidence).

If the case against merging contains a P0/P1 issue with `code evidence`, the verdict MUST be NOT CONVERGED.

## Step 4: Verdict

Judge **CONVERGED** only when ALL hold:

- The severity trend converges: each round's top severity is not higher than the previous round's, and the P0/P1 count does not rise;
- Round 3 has no P0/P1 (only P2/P3 remain);
- No fix-introduced regression detected (with test evidence, or explicitly stating that tests are unavailable but the diff trace was completed);
- No unanswerable objection in the case against merging.

If any condition fails, or **if anything is uncertain**, always judge **NOT CONVERGED**.

## Output format

Use the template matching the user's conversation language (see "Output language" above).

**English template** (user writes in English):

```
Verdict: CONVERGED / NOT CONVERGED

Meta-question 1 (severity trend): top levels P1→P2→P3 across rounds, P0+P1 counts 2→1→0
Meta-question 2 (fix regressions): none detected; basis: all three rounds' tests passed + diff trace
Meta-question 3 (last-round issues): R3-1 (P2, boundary) ...

The case against merging:
1. ... (evidence strength: code evidence)
2. ... (evidence strength: test evidence)
3. ... (evidence strength: speculation)

Production impact of last-round issues:
- Would definitely break: ... (scenario)
- Might break: ... (scenario)
- Would not break: ...

Unresolved items (unauthorized / unfixed P2/P3):
- [ ] R2-2: ... (suggested handling)
```

**Chinese template** (user writes in Simplified Chinese):

```
判定：收敛 / 判定：未收敛

关键问题 1（严重度趋势）：三轮最高级别 P1→P2→P3，P0+P1 数量 2→1→0
关键问题 2（fix 回归）：未检测到——三轮测试通过 + diff 追溯
关键问题 3（最后一轮问题）：R3-1（P2，边界）……

反对合并的理由：
1. ……（代码证据）
2. ……（测试证据）
3. ……（推测）

最后一轮问题的上线影响：
- 必然出问题：……（场景）
- 可能出问题：……（场景）
- 不会出问题：……

未修复项（未授权 / 未修复的 P2/P3）：
- [ ] R2-2：……（建议处理方式）
```

## Step 5: Commit decision (the user decides)

Show the user the full fix list of all three rounds (file + location + linked finding ID) and ask:

1. Commit these fixes? (all / some / none for now)
2. If committing: one commit, or multiple commits grouped by finding?

Execute after the user decides; uncommitted changes stay in the working tree. **Never commit on your own.**

## Delivery notes

- Keep the spoken conclusion to the user short: verdict + one-line reason + unresolved list + "whether to keep fixing is your call" (if NOT CONVERGED, point out the parts that most need human eyes);
- After the verdict, go to the commit decision (step 5); whether and how to commit is the user's decision;
- When the user asks to archive, write the summary above to a file (suggested: `REVIEW.md`, repo root).
