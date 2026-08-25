# Convergence Verdict After Three Rounds

Execute after the three-round loop finishes. **Fully restate the three rounds' findings and fix lists before the verdict** (from the conversation record) — never answer from memory or impression.

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

## Step 3: Prosecutor's brief (must be output before the verdict)

List the 3 strongest reasons against "safe to merge now", each with:
- The reason itself;
- An evidence strength label: `code evidence` (directly visible in the diff) / `test evidence` (has test output) / `speculation` (no direct evidence).

If the prosecutor's brief contains a P0/P1 issue with `code evidence`, the verdict MUST be NOT CONVERGED.

## Step 4: Verdict

Judge **CONVERGED** only when ALL hold:

- The severity trend converges: each round's top severity is not higher than the previous round's, and the P0/P1 count does not rise;
- Round 3 has no P0/P1 (only P2/P3 remain);
- No fix-introduced regression detected (with test evidence, or explicitly stating that tests are unavailable but the diff trace was completed);
- No irrefutable blocker in the prosecutor's brief.

If any condition fails, or **if anything is uncertain**, always judge **NOT CONVERGED**.

## Output format

```
Verdict: CONVERGED / NOT CONVERGED

Meta-question 1 (severity trend): top levels P1→P2→P3 across rounds, P0+P1 counts 2→1→0
Meta-question 2 (fix regressions): none detected; basis: all three rounds' tests passed + diff trace
Meta-question 3 (last-round issues): R3-1 (P2, boundary) ...

Prosecutor's brief:
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

## Step 5: Commit decision (the user decides)

Show the user the full fix list of all three rounds (file + location + linked finding ID) and ask:

1. Commit these fixes? (all / some / none for now)
2. If committing: one commit, or multiple commits grouped by finding?

Execute after the user decides; uncommitted changes stay in the working tree. **Never commit on your own.**

## Delivery notes

- Keep the spoken conclusion to the user short: verdict + one-line reason + unresolved list + "whether to keep fixing is your call" (if NOT CONVERGED, point out the parts that most need human eyes);
- After the verdict, go to the commit decision (step 5); whether and how to commit is the user's decision;
- When the user asks to archive, write the summary above to a file (suggested: `REVIEW.md`, repo root).
