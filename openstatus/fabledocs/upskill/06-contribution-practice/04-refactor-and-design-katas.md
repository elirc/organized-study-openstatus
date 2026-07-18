# Refactor & Design Katas

Eight judgment exercises. Output is writing + small diffs, not big PRs. Self-grade each against its criteria; redo any kata graded Basic a week later.

## Kata 1: Extract the quorum decision into a pure function
From [checker/index.ts:137-348](../../../apps/workflows/src/checker/index.ts#L137-L348), design `decideTransition(evidence): { flip?: Status; openIncident: boolean; notify?: NotifType }` — inputs: current aggregate status, per-region statuses, incoming report. Write the function signature, 10 table-driven test cases, and the diff plan (what stays in the handler: I/O only).
Self-grade — Basic: signature + cases. Solid: cases cover single-region, shrunk-region-list, and same-status-noop. Strong: the handler diff shows *zero* behavior change and you can say how you'd prove it (golden tests from production wide-event payloads).

## Kata 2: Design the outbox migration (write the RFC only)
Take Project S1 and produce just the RFC using the [07/02 template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md): problem, two designs compared (status-on-trigger vs separate outbox table), chosen tradeoffs, staged rollout, kill-switch.
Strong: includes the *duplicate-send* consequence and per-provider dedupe capabilities table.

## Kata 3: Kill the boundary leak in one v1 route
M1 as a kata: plan (don't implement) migrating [put.ts](../../../apps/server/src/routes/v1/monitors/put.ts) to services. List every observable behavior of the current route (status codes, validation order, response shape, side effects present/absent) *from reading it + its test*, then map each to the service verb, flagging mismatches as decisions-for-maintainers.
Strong: your mismatch list includes audit rows appearing where none existed (a behavior *addition* that needs a changelog note).

## Kata 4: Type-safety kata — de-duplicate the provider dispatch
[alerting.ts:144-239](../../../apps/workflows/src/checker/alerting.ts#L144-L239) repeats the retry-send block three times (J13's target). Write the typed helper so `method: "sendAlert" | "sendRecovery" | "sendDegraded"` stays narrowed against `providerToFunction`'s value type without casts.
Strong: no `any`/`as`; you can explain why the naive `providerToFunction[p][m]` loses narrowing and how a generic constraint restores it.

## Kata 5: Schema migration kata — NOT NULL workspace_id
Write the full plan (SQL + deploy order across dashboard/server/workflows) for making `monitor.workspace_id` NOT NULL ([monitor.ts:37](../../../packages/db/src/schema/monitors/monitor.ts#L37)): violating-rows query, remediation decision tree (orphans: delete? assign?), additive constraint timing, and rollback story (forward-only migrations — what does "rollback" mean here?).
Strong: names SQLite's ALTER limitations (table rebuild) and how drizzle-kit generates it.

## Kata 6: Reduce N+1 in the notification loop
Design the batched version of [alerting.ts:70-121](../../../apps/workflows/src/checker/alerting.ts#L70-L121) (workspace + SMS counts fetched once, not per notification). Constraint: behavior identical including the quota edge (`>` vs `>=` — decide and document which is correct per the limit's meaning).
Strong: your design notes that SMS is rare, so measure first — and the honest conclusion may be "not worth it"; a kata where *declining the refactor* is the senior answer.

## Kata 7: Review a flawed PR, fully written
Take Kata 6 from the [review katas](../04-code-reading-gym/04-review-katas.md) (async audit) and write the complete review: three comments (blocking/important/optional), each with anchor, reason, and suggested alternative; plus the summary paragraph that keeps the author motivated.
Grade against [07/01](../07-career-and-collaboration/01-code-review-mindset.md)'s language rules.

## Kata 8: Write the ADR the repo never wrote
Reverse-engineer the decision record for "audit log is fail-closed and in-transaction" from the code and comments ([emit.ts:92-101](../../../packages/services/src/audit/emit.ts#L92-L101), CLAUDE.md). Sections: context, decision, alternatives considered (fail-open async; CDC/log-based audit), consequences (availability coupling; Edge-safe diffing), revisit triggers.
Strong: consequences include the *asymmetry* you discovered in [03/01](../03-architecture-and-patterns/01-boundaries-and-layers.md) — pipeline mutations aren't covered — and whether the ADR should scope the invariant to "user-facing mutations" honestly.

---

Meta: katas 1, 4, 6 produce small PR-able diffs; 2, 3, 5, 8 produce documents. Both artifact types belong in your portfolio — interviewers increasingly ask for a design doc sample, and Kata 2/8 are exactly that.
