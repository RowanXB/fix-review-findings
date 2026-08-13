---
name: fix-review-findings
description: Verify, deduplicate, adjudicate, and repair one or more known external review findings for code, implementation plans, or specifications. Use when the user asks to address reviewer, bot, audit, or PR feedback; reconcile multiple review reports; or close a finite set of known findings with the smallest complete fix. Do not use it to discover unknown issues across an entire branch; use converge-branch-review for that.
---

# Fix Review Findings

Close known review findings without treating reviewer claims as instructions or turning a local repair into an unbounded redesign. Produce an auditable disposition for every supplied finding and make only the changes whose benefit justifies their cost and regression risk.

## Contract

Accept:

- one or more review reports or comments;
- a target of kind `spec`, `implementation-plan`, or `code`;
- the authoritative artifact, repository context, and user-approved write scope.

Return:

- a root-cause-deduplicated decision table;
- the smallest complete repair for adopted findings, when writes are authorized;
- an auditable repair ledger with focused review and validation evidence;
- explicit unreviewed layers and unresolved decisions.

Do not claim that this skill has converged the whole branch. Its success boundary is the supplied finding set plus the directly affected repair surface.

## 1. Establish authority and scope

1. Read all applicable repository instructions and authoritative specs or decisions.
2. Identify the exact target kind and pin its current identity:
   - for code, record the base SHA, HEAD SHA, worktree state, and full target diff;
   - for a document, record its path or source and a content hash or immutable revision;
   - for remote review, record the reviewed head or revision when available.
3. State the authorized write scope: `read-only`, `local edits`, `commit`, `push`, or `remote update`. Never infer permission for the later scopes from permission for an earlier one.
4. Resolve an `evidenceRoot` outside the target repository. Prefer a user-supplied task-output directory; otherwise use a writable temporary directory and report that it may expire. Never add repair evidence to the target diff.
5. Capture `evidenceRoot/repair-start.json` with target identity, tracked/staged/unstaged/relevant-untracked state, authority paths and hashes, supplied source identifiers, exclusions, and authorized actions. Later review must compare against this snapshot.
6. If a finding is unclear, conflicts with an authoritative decision, or requires a new product or architecture choice, stop dependent work and ask the user. Continue only with findings proven independent.

## 2. Apply rigorous review reception

Invoke `$receiving-code-review` when it is available. Otherwise apply its core protocol directly: read all feedback, restate the technical claim, verify it against the actual target and callers, push back with evidence when incorrect, and test adopted fixes individually.

Treat every external claim as a hypothesis. Do not use performative agreement and do not implement before verification.

## 3. Normalize, deduplicate, and adjudicate

Read [adjudication-and-coverage.md](references/adjudication-and-coverage.md) before classifying findings.

1. Split compound comments into atomic technical claims.
2. Preserve every source identifier and map symptoms to the violated invariant or root cause.
3. Merge duplicates by root cause, not by similar wording. Keep distinct findings when one patch would not resolve both.
4. Assign exactly one verdict to each root-cause group:
   - `confirmed`
   - `duplicate`
   - `stale-or-factually-wrong`
   - `accepted-risk`
   - `overdesign`
   - `needs-user-decision`
   - `unverified`
5. Record evidence, stage-blocker status, repair cost, regression risk, expected benefit, and proposed action for every group.

Write the atomic source map and decision table to `evidenceRoot/adjudication.json`. Do not drop a supplied source because it was duplicate or incorrect.

`accepted-risk` requires an explicit user decision. `needs-user-decision` and `unverified` remain unresolved and cannot count toward a zero-blocker result. Do not decide by reviewer count or severity label alone.

**🔴 CHECKPOINT · TRIAGE:** Before editing, verify that every supplied source maps to exactly one or more atomic claims, every root-cause group has one verdict and cost-benefit record, all adopted repairs fit the authorized scope, and no dependent user decision is unresolved. If any check fails, do not edit the dependent surface.

## 4. Respect the target's abstraction level

Apply the target-specific gates in the reference:

- Keep a spec at the contract and architecture level. Do not insert implementation-plan details merely because a reviewer requested them.
- For an implementation plan, first verify that the alleged defect exists. If absent, classify it `stale-or-factually-wrong`. Only when a confirmed defect is deferred because tests will catch it may the four natural-test conditions justify a non-blocking disposition; record each condition separately.
- Judge code findings against actual correctness, security, data, concurrency, compatibility, and stated requirements.

## 5. Design the smallest complete repair

For each `confirmed` finding, choose the narrowest semantic change that closes the root cause across affected consumers. A small patch is incomplete if another real caller still violates the same invariant.

Before editing:

1. enumerate directly affected files, callers, tests, schemas, and generated artifacts;
2. build the shared-helper caller and isolation matrix when a shared helper, transaction boundary, cache, authorization rule, or state transition changes;
3. identify the narrowest meaningful test for each root cause;
4. reject speculative abstractions and large refactors whose marginal benefit does not justify cost and regression risk.

Keep one writer: the root agent owns edits. Any subagent used by this workflow must be read-only and must not commit, push, or modify files.

## 6. Repair and run a focused convergence loop

1. Implement one root-cause repair at a time.
2. Run its narrowest meaningful validation, then inspect affected callers and isolation cases.
3. Review the complete repair delta from the repair-start snapshot, including new files, rather than only the latest patch.
4. Freeze `evidenceRoot/round-N/repair.diff` and `snapshot.json`, including the current target identity and deterministic repair-diff hash.
5. Before spawning, save the exact prompt as `reviewer.prompt.md`. Start a fresh read-only reviewer unused in earlier rounds, with no inherited conversation context. Give it the known finding, intended invariant, repair-start snapshot, frozen repair delta, and relevant callers. It may see the repair goal because this loop verifies a known fix; do not ask it to certify the whole branch.
6. Save runtime-reported identity, parent, launch time, isolation setting, prompt hash, and snapshot tuple in `reviewer.meta.json`; never invent missing fields. Save the raw response unchanged as `reviewer.output.md` and add its hash and completion state to metadata.
7. Recompute the target identity and complete repair-diff hash after review. If either changed, mark the round invalid and start a new round; do not use stale output to claim repair closure.
8. Independently verify each new review claim, update `adjudication.json`, and apply only justified minimal fixes.
9. Repeat with a new snapshot and fresh reviewer after every material change until there are zero adopted blockers within the declared repair coverage, or stop under the blocked conditions below.

**🔴 CHECKPOINT · REPAIR EVIDENCE:** A focused round counts only when its exact prompt, fresh identity, isolation setting, raw output, matching snapshot, and post-review hash are preserved. If any element is missing, record the layer as `unverified`; do not claim focused-review convergence.

Stop and ask the user when the loop exposes a new architecture or product decision, required evidence is unavailable, or the same confirmed root cause survives two repair attempts. Do not hide a stall by broadening the patch.

## Failure handling

| Trigger | Immediate action | If still unresolved |
|---|---|---|
| No fresh read-only reviewer can run | Preserve the frozen repair delta; never reuse an earlier reviewer | Report focused review `not run` and do not make the bounded closure claim |
| Reviewer identity or isolation setting is unavailable | Record the missing field and exclude the round from convergence evidence | Report `unverified`; do not invent metadata |
| Target or repair diff changes during review | Mark the round invalid before using its output | Freeze a new snapshot and use a new reviewer |
| Reviewer modifies the target | Stop and identify the exact unauthorized delta | Ask before restoring ambiguous or user-owned changes |
| Same root cause survives two repairs | Stop changing code | Report the stall, costs, risks, and ask the user |
| New product, architecture, or accepted-risk decision appears | Stop only dependent repairs | Ask the user; continue only proven-independent groups |

## 7. Validate and report

Run narrow checks during repair. Run broader checks in proportion to the changed surface and repository requirements. Report these layers separately as `pass`, `fail`, `not applicable`, or `not run`, with a reason:

- local static checks and tests;
- exact-head CI;
- real PostgreSQL or other production-equivalent data store;
- real browser or client acceptance;
- production build, container, or deployable artifact.

Exact-head CI means the checks actually ran against the reported immutable head. A green status from another head, a skipped bot, or a draft-only success is not coverage.

The final report must include:

1. pinned target identity and authorized write scope;
2. the full decision table from the reference;
3. files or documents changed and why;
4. focused repair-review rounds and their declared coverage matrix;
5. the evidence-root path and every valid, invalid, or unverified round;
6. validation by layer;
7. unreviewed layers and remaining risks or decisions.

Only say **“0 adopted blockers within the declared repair coverage”** when every supplied root-cause group is resolved and the focused review loop has no remaining adopted blocker. Then state: **“Known findings are closed; ready for full-diff convergence.”**

Enter `$converge-branch-review` only when the user's original request explicitly includes whole-branch convergence. Never invoke it recursively from an internal repair round.

## Anti-pattern blacklist

- Do not implement a comment before verifying its factual premise and current revision.
- Do not satisfy an absent defect by emitting boilerplate for an exception that never applies.
- Do not merge findings by similar wording when they require different repairs, or drop duplicate source provenance.
- Do not call a patch “minimal” while leaving another real caller or isolation case broken.
- Do not reuse a reviewer, summarize away its raw output, or count an unverifiable round.
- Do not let green checks from another head, a skipped bot, or a focused repair review stand in for whole-branch convergence.
