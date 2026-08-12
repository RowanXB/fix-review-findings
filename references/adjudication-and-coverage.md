# Adjudication and Coverage Reference

Read this file before classifying findings or declaring repair completion.

## Verdict requirements

| Verdict | Required evidence | Effect on completion |
|---|---|---|
| `confirmed` | Reproduced behavior, authoritative contradiction, or concrete reachable failure | Must be repaired or explicitly accepted by the user |
| `duplicate` | Same violated invariant and same repair as another root-cause group | Retain all source IDs; do not count twice |
| `stale-or-factually-wrong` | Claim targets an old revision, unreachable path, incorrect platform fact, or behavior disproved by current evidence | No repair; record counter-evidence |
| `accepted-risk` | User explicitly accepts the described consequence for this scope | Non-blocking only within that authority |
| `overdesign` | Suggested change costs materially more than its evidenced benefit or serves no real caller/requirement | No repair; record cost-benefit reasoning |
| `needs-user-decision` | Correct action depends on a new product, architecture, compatibility, or risk choice | Blocks completion |
| `unverified` | Required code, environment, reproduction, or authority cannot be inspected | Blocks completion |

Do not use `accepted-risk` as a synonym for “the agent chose not to fix it.” Do not use `overdesign` merely because a valid fix is inconvenient.

## Target-specific stage gates

### Specification

Keep findings in the spec when they affect durable intent:

- domain contract and invariants;
- state model and transition semantics;
- identity, provenance, ownership, and authorization;
- failure, uncertainty, retry, and idempotency semantics;
- data model or architecture choices needed to establish feasibility;
- compatibility and externally visible behavior.

Defer implementation mechanics to an implementation plan unless they are necessary to make the contract unambiguous or feasible:

- concrete file names and helper signatures;
- command sequences;
- test file layout and fixture mechanics;
- step-by-step patch ordering;
- incidental framework details.

### Implementation plan

Classify a defect as a planning blocker when deferring it would likely cause expensive rework, unsafe execution, or an irreversible wrong turn. Typical blockers include schema and migration direction, lock ordering, transaction boundaries, authorization, state identity, compatibility boundaries, and cross-package API shape.

“Implementation tests will naturally find this” is a valid reason not to block only when all are true:

1. the plan explicitly requires the relevant test;
2. the test runs before irreversible or broad downstream work;
3. failure localizes the issue clearly;
4. the likely repair will not change the spec, schema, authorization model, shared protocol, or major task decomposition.

Otherwise settle the issue in the plan.

### Code

Verify actual reachability and impact across correctness, security, authorization, data integrity, concurrency, failure behavior, compatibility, observability, tests, and documented requirements. A theoretical issue with no real caller may be `overdesign`; a rare but catastrophic reachable issue may still be `confirmed`.

## Cost-benefit record

For every root cause, record:

- `repair cost`: low, medium, or high, with the main affected surface;
- `regression risk`: low, medium, or high, with the behavior at risk;
- `benefit`: prevented failure and its likelihood or severity;
- `action`: fix now, defer by explicit decision, reject, or ask user.

Prefer the smallest change whose benefit clearly exceeds its expected implementation and regression cost. Evaluate expected value, not rarity alone.

## Shared-helper and isolation matrix

Create this matrix only when the repair changes a shared helper, stateful boundary, cache, lock, transaction, authorization rule, retry, timeout, or terminal transition. Derive rows from actual callers rather than inventing hypothetical ones.

Suggested axes:

| Axis | Positive case | Isolation or negative case |
|---|---|---|
| User | authorized owner succeeds | another user cannot observe or mutate |
| Goal or parent | correct parent is affected | sibling parent remains unchanged |
| Run or attempt | intended run changes | another run remains isolated |
| Retry | safe retry preserves invariant | duplicate or non-idempotent work is prevented |
| Timeout or unknown | confirmed outcome is handled | uncertain outcome is not relabeled or replayed |
| Terminal state | allowed transition succeeds | terminal or forbidden transition is rejected |

Mark each relevant cell as covered by code inspection, automated test, runtime test, or unreviewed. Never claim matrix coverage for a cell that was inferred but not checked.

## Required decision table

Use one row per root-cause group:

| Root cause | Sources | Severity | Verdict | Stage blocker | Evidence | Repair cost | Regression risk | Benefit | Action |
|---|---|---|---|---|---|---|---|---|---|

After the table, list any source comment that was split across groups so provenance remains complete.
