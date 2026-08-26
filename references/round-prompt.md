# Per-Round Review Instruction Template (execute every line each round)

## 0. State confirmation (in the conversation, no file reads)

- **Restate the previous rounds** (skip in round 1): list the previous rounds' findings (ID, level, status) and fixes (file + location) in your own words to confirm the conversation memory is complete; if the restatement is incomplete, complete it first — never skip.
- **External verification**: after restating, re-read the files touched by the fix list and check the actual working tree against the restatement — conversation memory can drift; the actual code wins; correct the list on mismatch.
- Read `references/rubric.md` to calibrate the severity scale and the category list.

## 1. Define the round scope (re-read the latest code)

- Scope = the user-confirmed scope from initialization (see SKILL.md → Workflow → Initialize); restate the chosen option in one line. If the scope turns out empty, report it and re-confirm with the user.
- **Re-run every round**: fetch the latest diff and re-read the relevant file contents; reconstructing the code state from memory is forbidden (including code you changed in the previous round yourself).
- List the files to review this round.
- **Attention allocation priority** (the key of the multi-round design; follow strictly):
  1. Deltas introduced by the previous round's fixes: use the fix list (file + location) from the conversation record and re-read those files plus their neighboring code — fixes are uncommitted, so don't rely on git log to locate them;
  2. Categories and files **not covered** in previous rounds (complementary coverage, see step 2);
  3. Finally, spot-check callers of the fixed code in already-covered categories.

## 2. Coverage log (recorded in the conversation)

- Record which categories (the categories in the rubric) and which files were checked this round.
- Compare with the previous round and explicitly mark: newly covered [X] / re-checked [Y].
- If this round's findings fall into categories already checked in previous rounds, say so honestly (that is an attention problem, worth flagging); findings in previously unchecked categories are normal coverage progression.

## 3. Review and record

- Process each candidate issue as follows:
  1. **Dedupe**: compare against the restated history list — the same issue reworded is not a new finding;
  2. **Evidence**: each finding must cite evidence: the relevant requirement/acceptance criteria, a reproducible failure scenario, or a failing test; drop it if no evidence can be given. Test evidence counts only if the assertion can actually fail on the target behavior's regression — an assertion that cannot fail (e.g., a serialization/string-match assertion never checked against the real output format) is defective, not evidence;
  3. **Grade**: label P0-P3 per the rubric, with reasoning;
  4. **Record**: record in the conversation with an ID (e.g., R2-1), including level/category/location/description/evidence.
- The review must check callers (`git grep` usage sites), not just the change itself.

## 4. Report and authorization (must come before fixing)

- Report this round's findings to the user **in the user's conversation language** (detect it from the user's messages; a Chinese user gets a Chinese report): the issue list (level + location + one-line description) plus a fix plan per issue;
- Fix recommendation tiers:
  - P0/P1: recommended to fix (must-fix);
  - P2/P3: give a "fix / optional" recommendation with reasoning; the user decides;
- **Ask for authorization**: one authorization covers all fixes of the round; the user may name exceptions;
- Issues without authorization are not fixed and go into the "unfixed + reason" list.

## 5. Fix (execute after authorization)

- Modify working-tree files directly, **no commits**; record each fix in the conversation: file + location + change summary + linked finding ID;
- Avoid touching unrelated code along the way.

## 6. Regression check (verify this round's fixes)

- Detect the project's test/lint commands (package.json scripts, Makefile, pyproject/pytest, Cargo, etc.); run them if available and record an output summary;
- **When the project has no test command, honestly write "tests unavailable, not verified" — never write "no regression"**;
- Diff trace: check the change surface of this round's fixes one by one; verify callers and neighboring logic are unaffected. **The fix's new code itself is a review target**: new state, resources, and caches must be bounded and consistent with existing conventions; issues found in the fix are recorded and graded this round — do not wait for the next round;
- Regressions found are recorded as new findings and graded the same way; P0/P1 regressions: ask for authorization again on the spot, then fix; P2/P3 regressions: add to the unresolved list (fold into the next round if there is one).

## 7. Record the round's conclusion (in the conversation)

- Findings this round: N (P0 × a / P1 × b / P2 × c / P3 × d);
- Fixes this round: N (file + location + linked ID), all uncommitted;
- Unresolved items and reasons;
- Scope and baseline this round (one line, e.g., merge-base → working tree / uncommitted only / commit range);
- Coverage list + search space: what was NOT checked or run this round (E2E not run, cross-environment/cross-implementation not verified, categories not covered). **A zero-finding round must declare its search space** — "no findings" only means "nothing found in the searched space";
- Regression check conclusion and evidence.
