# Writing PRs and RFCs

## PR descriptions — the five sections

**What** (one sentence) · **Why** (the problem, with anchors) · **How tested** (commands + what they prove) · **Risks** (what could break, blast radius) · **Follow-ups** (consciously deferred).

Worked example for ticket J13:

> **What:** Extract the triplicated retry-send block in `triggerNotifications` into one typed helper.
> **Why:** `alerting.ts:144–239` repeats identical Effect-retry logic for alert/recovery/degraded; three copies have already drifted risk (log fields) and triple every future change.
> **How tested:** `bun test` in apps/workflows *(none existed — added table-driven tests for the helper with a fake provider)*; manual diff of emitted log fields before/after (paste).
> **Risks:** Low — pure refactor; typed method union prevents dispatch typos; no behavior change intended, retry counts asserted in tests.
> **Follow-ups:** provider-call timeouts are still unbounded (out of scope, filed #—).

Why this works: a reviewer can review *risk-first*, and "no behavior change intended" + evidence is the strongest sentence a refactor PR can contain.

## Commit messages

Imperative subject ≤ 65 chars; body = why + notable decisions, never a code walkthrough (the diff shows *what* — same principle as the repo's comment discipline in [CLAUDE.md](../../../CLAUDE.md)). One logical change per commit; migrations and behavior changes never share a commit with refactors.

## When to RFC

RFC before code when any of: schema migration touching shared tables; public API surface change; new infrastructure dependency; changing an invariant (audit semantics, quorum, idempotency keys). PR-only when: bounded, reversible, pattern-following.

## RFC template (tailored to this repo)

```md
# RFC: <title>
Status: draft | discussed | accepted
## Problem
What breaks/costs today. Anchors: file:line evidence, wide-event/Sentry data if available.
## Constraints
Invariants that must survive (e.g. "every user-facing mutation audits in-tx",
"Edge-safe services", "v1 wire contract frozen", "task-name idempotency").
## Design A / Design B
Sketch each; per design: schema delta, deploy order across
dashboard/server/workflows/checker, failure modes introduced.
## Decision & rationale
Which and why; who pays what cost.
## Rollout
Additive → shadow/flagged → enforce; kill switch; migration ordering
(three apps share the DB — state the order explicitly).
## Test strategy
The test that would fail today and pass after; contract tests if the
TS↔Go payload changes.
## Open questions
```

The Constraints section is the differentiator — it proves you know the house invariants before proposing to move them. Fill it from [03-architecture-and-patterns](../03-architecture-and-patterns/README.md).

## Drill

Write the full PR description for ticket M3 (system-actor audit rows) *before* implementing anything. If the Risks section is thin, you don't understand the ticket yet — go re-read [emit.ts:92-101](../../../packages/services/src/audit/emit.ts#L92-L101) and decide fail-open vs fail-closed for pipeline paths.

## Interview angle

"Walk me through a technical decision you documented" — the RFC template *is* the answer structure: problem → constraints → alternatives → decision → rollout. Interviewers grade the constraints and rollout sections hardest; juniors skip both.
