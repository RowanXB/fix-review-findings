# Fix Review Findings

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Turn a finite set of review comments into verified, proportionate repairs—with an audit trail and without blindly implementing every suggestion.

> 中文简介：这个 Skill 用来处理已经收到的代码、实施计划或规格审查意见。它先核实事实、按根因去重并评估收益与风险，再完成最小但完整的修复；它不会把审查意见直接当成命令，也不会把一次局部复审包装成“整个分支已经没有问题”。

## Why this exists

Review feedback is useful, but it is not automatically correct, current, independent, or worth implementing. Several reviewers may describe the same root cause, a comment may refer to an older revision, and a technically possible edge case may require a disproportionately risky redesign.

Fix Review Findings makes those decisions explicit. It is designed to answer four questions:

1. Is the finding true on the exact target being repaired?
2. Is it distinct, or another symptom of an existing root cause?
3. Is repairing it now worth the cost and regression risk?
4. What is the smallest complete repair, and what evidence supports closure?

## When to use it

Use this Skill when you already have a finite collection of findings, such as:

- pull-request comments from one or more reviewers;
- findings from security, architecture, or test reviews;
- feedback on an implementation plan;
- feedback on a specification or design document.

Use [Converge Branch Review](https://github.com/RowanXB/converge-branch-review) instead when the main goal is to discover unknown defects across an entire branch and repeatedly review the full base-to-target diff.

## How it works

1. **Pin the repair target.** Record the relevant base, head, dirty state, review sources, authority, and write scope.
2. **Normalize the feedback.** Preserve every source identifier, split compound comments into independently testable claims, and group duplicates by root cause.
3. **Adjudicate each claim.** Inspect the actual code and authoritative documents before deciding whether the comment should be adopted.
4. **Apply the right abstraction boundary.** A specification, implementation plan, and code change require different levels of detail.
5. **Make a minimal complete repair.** Use the [repair boundary record](references/adjudication-and-coverage.md#repair-boundary-record) to trace the full invariant, actual consumers, new responsibilities and counterexamples. When a prior repair was incomplete, supersede its unsupported closure claim without discarding still-valid evidence.
6. **Run focused repair review.** A fresh read-only reviewer checks the new repair diff. Material fixes create a new snapshot and another round.
7. **Report bounded evidence.** Local tests, exact-head CI, real PostgreSQL, browser checks, and production-artifact checks are reported separately.

The root agent remains the only writer. Review agents are read-only and do not inherit the repair history they are meant to challenge.

## Finding decisions

Every normalized claim receives one of these outcomes:

| Verdict | Meaning |
| --- | --- |
| `confirmed` | The issue exists and should be repaired. |
| `duplicate` | The evidence is real but belongs to another root cause already being handled. |
| `stale-or-factually-wrong` | The claim does not match the pinned target or authoritative facts. |
| `accepted-risk` | The issue is real, but the user or governing decision explicitly accepts it. |
| `overdesign` | The proposed change costs or risks more than the demonstrated benefit justifies. |
| `needs-user-decision` | Repair requires a new product, architecture, or risk decision. |
| `unverified` | Available evidence is insufficient to decide safely. |

The report records the evidence, repair cost, regression risk, expected benefit, and action for each claim. This makes “do not fix” a reasoned decision rather than a silent omission.

## The implementation-plan gate

A confirmed issue should be settled in the plan when deferring it would make the eventual repair substantially more expensive, unsafe, or irreversible—for example, a wrong schema direction, lock order, transaction boundary, authorization model, state identity, compatibility contract, or cross-package API shape.

“Tests will catch it during implementation” is an acceptable reason to defer only when all of the following are true:

- the detecting test is explicitly included in the plan;
- it runs before irreversible work or broad downstream implementation;
- a failure will localize the problem clearly;
- repairing the failure will not force changes to the specification, schema, authorization model, shared protocol, or major task decomposition.

For specification reviews, the inverse boundary applies: keep durable product and architecture contracts in the spec, but leave filenames, helper choices, commands, and test mechanics to the implementation plan.

## Evidence and closure

Focused repair rounds write immutable evidence outside the target repository. A typical evidence directory contains:

```text
repair-start.json
adjudication.json
round-001/
  repair.diff
  snapshot.json
  reviewer.prompt.md
  reviewer.meta.json
  reviewer.output.md
```

The saved diff bytes are hashed with SHA-256. If the target changes, previous review authority is invalidated and the snapshot must be regenerated.

The strongest successful local claim is deliberately scoped:

> 0 adopted blockers within the declared repair coverage.

That means the known findings are closed and the repair is ready for full-diff convergence. It does **not** mean the whole branch is defect-free or production-ready.

## Requirements and fail-closed behavior

You need:

- an agent runtime that can load Agent Skills;
- a Git working tree or another target with stable snapshot identity;
- a writable evidence directory outside the target repository;
- fresh, read-only reviewer contexts for bounded review closure;
- explicit authority for any edits, commits, pushes, or external actions.

The Skill can still triage and repair when some validation infrastructure is unavailable, but it will not claim bounded review closure without the required fresh-reviewer and identity evidence. Missing CI, database, browser, or production checks remain visible gaps.

## Installation

Clone the repository into a directory your agent runtime scans for skills. A common workspace layout is `.agents/skills/`:

```bash
mkdir -p .agents/skills
git clone https://github.com/RowanXB/fix-review-findings.git \
  .agents/skills/fix-review-findings
```

User-level skill directories vary by runtime; adjust the destination path as needed. The executable instructions live in [`SKILL.md`](SKILL.md).

## Example request

```text
Use Fix Review Findings to verify these PR comments against the current branch,
deduplicate them by root cause, repair only justified findings, and report the
cost/risk decision and validation evidence for every item. Do not commit or push.
```

If your runtime supports explicit skill mentions, you can use its equivalent of `$fix-review-findings`.

## Repository layout

```text
SKILL.md                              Executable workflow instructions
agents/openai.yaml                    Skill metadata
references/adjudication-and-coverage.md
                                      Verdict and coverage guidance
evals/evals.json                      Evaluation scenarios
LICENSE                               MIT license
```

## Relationship to Converge Branch Review

These Skills are intentionally separate:

- **Fix Review Findings** consumes known findings and closes a bounded repair scope.
- **Converge Branch Review** discovers unknown findings across the full branch diff.
- Converge may call Fix for adopted findings, then start a new three-reviewer full-diff round on the repaired snapshot.
- Fix does not recursively start whole-branch convergence unless the user explicitly requests that broader scope.

## License

Released under the [MIT License](LICENSE).
