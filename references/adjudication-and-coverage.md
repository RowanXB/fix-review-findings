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

## Repair boundary record

For each adopted root cause, fill these slots before editing and update them with observed results. Keep the record proportional to the actual repair; it bounds the work, not an invitation to audit unrelated code.

| Required slot | Content |
|---|---|
| Invariant | The complete observable contract, including behavior the repair must preserve; distinguish the reported symptom from that contract. |
| Origin | Existing omission, introduced by an earlier repair, changed requirement, or unknown. Cite the relevant revision or decision; an increasing finding count alone proves none of these. |
| Affected surfaces | Actual producers, callers, consumers and derived artifacts. Record each as change, checked unchanged, outside scope, or unverified, with evidence and reason. |
| New responsibilities | State, effects, dependencies or resource costs added or moved by the proposed fix; pair each with a legitimate counterexample and its expected behavior. Use “none” only with a reason. |
| Validation | Original failure, preserved behavior and counterexample checks; name what each check proves, what it substitutes, and what remains untested. |
| Closure evidence | Current disposition, evidence references and remaining gaps. If an earlier receipt overclaimed closure, identify the exact claim being withdrawn or narrowed and the replacement evidence required. Retain still-valid evidence and the original receipt as history. |

For code repairs, follow the changed contract through the real operation to its next observable use. Inspect actual callers and owners rather than only the named function. Include data writers/readers, enclosing lifecycle, fixtures, and contract artifacts when that path reaches them. Cross-check schema, migrations, generated metadata, workflow contracts and documentation only when the repair changes the contract they describe. Do not infer that every listed category needs editing.

Select counterexamples from supported behavior: a legal state the patch might now reject, an unaffected consumer, an alternate order of operations, or a change in input/data volume. Check whether a local fix moves work into a broader execution scope or changes failure semantics. Derive quantitative limits from the approved contract and actual representation; introducing a new product limit requires a decision.

Test the boundary supporting the claim. A substituted dependency cannot prove that dependency's behavior or cost. Use the real boundary, inspect its implementation with explicitly narrower evidence, or report the gap. Exercise complete operation sequences, including the next operation, rather than only a helper's return value. A test that merely restates the chosen patch is not independent evidence of the invariant.

When findings recur, use the origin and new-responsibility slots to revise the repair model. More tests or more reviewers do not replace a missing boundary. Close the authorized repair when its adopted defects and directly affected contracts are verified; keep optional maintenance, new decisions, full-branch review, CI and activation gates separately identified. Neither repeated advice nor a severity label creates a new gate.

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
