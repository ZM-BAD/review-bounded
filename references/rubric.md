# Severity Rubric

Every finding in every round must be graded on this scale. **The scale must not drift between rounds**: when grading in later rounds, calibrate against the labels of similar issues from earlier rounds in the record so that rounds stay comparable.

## Severity definitions

| Level | Definition | Examples |
|---|---|---|
| P0 | Directly causes crashes, deadlocks, data loss/corruption, security vulnerabilities, or makes core functionality unavailable | Null pointer/segfault, concurrency deadlock, ever-growing memory leak, injection-type security vulnerability, uncaught exception that kills the process |
| P1 | A definite bug with a workaround or limited blast radius; or a clear violation of acceptance criteria | Wrong result on boundary input, missing error handling but main flow works, behavior diverging from requirements |
| P2 | Correct, but with a real robustness/performance/consistency/maintainability risk | Notable performance regression, hard-coded configuration, duplicated implementation, undiagnosable error messages |
| P3 | Non-functional suggestions: style, naming, comments, structural tweaks | Poor naming, dead code, formatting issues, readability suggestions |

## Grading rules

- Grade by "**what happens if this ships**", not by "how annoying it is to fix".
- Each issue is graded once; re-reporting across rounds (including reworded) counts as a duplicate — dedupe, don't escalate.
- An observation with no concrete failure scenario is not P0/P1; record it only if it is at least P2; pure personal preference is not recorded.

## Review categories (for the coverage log and complementary coverage)

| Category | Description |
|---|---|
| Correctness | Logic errors, wrong branches, boundary inputs |
| Concurrency / resources | Deadlocks, races, memory/connection/file-handle leaks, missing timeouts |
| Error handling | Exception catching, error propagation, failure paths, retry strategy |
| Security | Injection, auth, sensitive info leakage, input validation |
| Performance | Complexity regressions, N+1, needless recomputation |
| Consistency | Consistency with other modules / existing conventions, duplicated implementations |
| Maintainability | Naming, abstraction, dead code, testability, hard-coding |
| Compatibility | Interface change blast radius, data formats, config compatibility |
| Tests | Whether tests cover key paths, or only the happy path; whether assertions can actually fail on the target behavior's regression (an assertion that cannot fail is defective, not evidence) |
