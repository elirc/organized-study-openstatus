# Annotation Drills

Ten real excerpts. For each: open the anchor, and annotate **inputs, outputs, dependencies, invariants, side effects, failure modes** before reading the answer key notes. Write your annotations down — reading is not the drill.

Self-grading rubric (applies to all drills):
- **Basic:** correct inputs/outputs; names the obvious side effect.
- **Solid:** + dependencies (what must be true before this runs), at least one non-obvious failure mode.
- **Strong:** + the invariant the code protects, and what breaks elsewhere if this code is wrong.

---

## Drill 1 — [create.ts:16-63](../../../packages/services/src/monitor/create.ts#L16-L63) (`createMonitor`)

Annotate all six categories.
Key notes to check yourself against: invariant = *no monitor without workspace scoping and an audit row, atomically*. Dependencies = seeded plan limits on `ctx.workspace.limits`; failure modes = LimitExceeded, zod, audit-insert failure (rolls back the create — is that in your list?). Side effects = 2 inserts, nothing external.

## Drill 2 — [emit.ts:102-149](../../../packages/services/src/audit/emit.ts#L102-L149) (`emitAudit`)

Trick question in the middle: when does a call *insert nothing and that's correct*?
Check: both-snapshots-empty-diff-no-metadata → skip (L126–132). Failure mode most people miss: `auditEntrySchema.parse` throwing on an unknown action name — meaning **adding a new verb requires updating the action union** in `@openstatus/db/src/schema/audit_logs/validation.ts` (per CLAUDE.md) or every call fails at runtime.

## Drill 3 — [checker/index.ts:216-249](../../../apps/workflows/src/checker/index.ts#L216-L249) (recovery branch)

Annotate with special attention to *dependencies on earlier lines* (the `monitor` snapshot was read at L128–134 — before the quorum count).
Check: invariant *attempted* = "notify once per real transition." Failure mode = two concurrent recoveries both see stale `monitor.status === "error"`; both pass the guard at L219; dedupe insert is the only stop. Side effects = monitor.status write, incident resolution write, provider notifications, audit publishes.

## Drill 4 — [alerting.ts:70-139](../../../apps/workflows/src/checker/alerting.ts#L70-L139) (notification loop head + SMS quota)

Check: inputs include *implicit* ones — `notificationsToMonitors` join rows and workspace limits fetched mid-loop. Failure modes: N+1 workspace/SMS-count queries per notification (performance), quota check uses `>` against `sms-limit` (off-by-one? decide and defend), `insertNotificationTrigger` catch treats *any* error as "already sent" ([alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139)) — a connection error suppresses a page. That last one is the Strong-level find.

## Drill 5 — [auth.ts:159-226](../../../apps/server/src/libs/middlewares/auth.ts#L159-L226) (`validateKey`, production branch)

Check: dependencies = `os_` prefix length math (3+8=11, L161); invariants = hash never compared without prefix match; expiry checked *after* hash verify (order matters? why or why not); side effect = debounced `lastUsedAt` write — a **write inside an auth read path** (what does that do under read replicas or high concurrency?). Failure mode = Unkey outage → all legacy keys 401.

## Drill 6 — [withTestTransaction, helpers.ts:37-50](../../../packages/services/test/helpers.ts#L37-L50)

Check: the sentinel-error rollback trick; invariant = *no committed writes from tests*; failure modes = nesting (no savepoints over HTTP), reads via committed `db` inside the callback missing in-tx writes (documented at L29–35), and a `result` variable assigned before an intentional throw (`let result!: T` — why the `!`?).

## Drill 7 — [cron/checker.ts:153-183](../../../apps/workflows/src/cron/checker.ts#L153-L183) (task-input assembly)

Check: inputs = monitors + statusMap + regionDict; outputs = TaskInput[] including the *enqueued-with* status (this becomes `req.Status` in Go — the edge-trigger baseline). Failure modes: unknown region skipped with log; deprecated region skipped; 30s monitors get a second task at +30s (doubling semantics). Invariant = a task exists per (active monitor × valid region × tick).

## Drill 8 — [proxy.ts:15-96](../../../apps/status-page/src/proxy.ts#L15-L96) (status-page gate)

Check: inputs include four credential channels (cookie, `?pw`, session email, client IP). Output = a three-way action union — note how making the decision a *pure function* (`composePageAction`) moves all testable logic out of the middleware. Failure modes: DB lookup per request (no cache), `console.log` of access metadata (PII-ish in logs), passthrough on schema-validation failure (fail-open? what does that mean for a page mid-migration?). That fail-open observation is the Strong find.

## Drill 9 — [checker.go:120-183](../../../apps/checker/handlers/checker.go#L120-L183) (probe op closure)

Check: the closure mutates `called` and `result` from the enclosing scope — annotate which variables cross the closure boundary. Failure modes: `json.Marshal` errors ignored for assertions string (L98–104, `_`), retry-until-success semantics conflate "assertion failed" with "network error" (both return error → retry). Invariant = one Tinybird row per *final* attempt outcome (early failed attempts don't emit — confirm by reading where `SendEvent` sits relative to the retry loop; note the last-attempt failure path at L276–313 emits separately).

## Drill 10 — [upsert.ts:92-146](../../../packages/services/src/page-subscriber/upsert.ts#L92-L146) (existing-subscriber branch)

Check: three-way branch (verified / pending / new); invariant = verified subscribers never modified by re-signup; side effects on the pending path = join-table inserts with `onConflictDoNothing`, expiry refresh, audit with metadata. Failure mode: `componentIds` merge is additive-only — a re-signup can widen scope but never narrow it (feature or bug? defend).

---

After all ten: write one paragraph titled "What this codebase considers non-negotiable" (tenancy scoping, atomic audit, idempotent effects, parse-at-boundary). If your paragraph says those four things, you've extracted the architecture from the code — which is exactly the skill "walk me through a codebase you know" interview questions measure.
