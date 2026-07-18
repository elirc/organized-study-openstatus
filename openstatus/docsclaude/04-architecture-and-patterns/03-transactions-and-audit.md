# 03 — Transactions and the Audit Log Invariant

Atomicity and auditability are the two non-negotiables of a serious multi-tenant app. Skip either and a senior reviewing your PR should hold the merge button.

## Why transactions are mandatory for multi-row writes

A status report creation writes to three tables:

1. `status_report` (the report header).
2. `status_report_update` (the initial update).
3. `status_report_to_page_component` (association rows).

Plus an audit row in `audit_log`.

Without a transaction, any failure in steps 2–4 leaves a half-created entity. The header exists with no update; the affected components are wrong; the audit log is missing the record of what happened. None of those failure modes is acceptable; all of them are catastrophic in different ways:

- A header with no update is silent — it appears in the UI as an empty report.
- Wrong component associations show the wrong "affected" services on the public page.
- A missing audit row means a SOC2 reviewer asks "who created this?" and you can't answer.

Wrap related writes in a transaction. Always.

## `withTransaction` — composable atomicity

[`packages/services/src/context.ts:40-47`](../../packages/services/src/context.ts#L40-L47):

```ts
export async function withTransaction<T>(
  ctx: ServiceContext,
  fn: (tx: DB) => Promise<T>,
): Promise<T> {
  const db = ctx.db ?? defaultDb;
  if (isTx(db)) return fn(db);                         // already in a tx — reuse
  return (db as DrizzleClient).transaction(fn);
}
```

Two behaviors:

- If the caller's `ctx.db` is already a transaction (a higher-level service started one), reuse it. Nested calls join the existing tx instead of starting a new one — which would either fail (libsql HTTP doesn't support savepoints) or break atomicity.
- Otherwise, open a fresh transaction.

This is what lets one service call another and still have the whole thing be atomic. The outer service opens; the inner sees `ctx.db = tx` and *uses* it.

Don't open transactions any other way. Don't write `db.transaction(...)` inside a service. Always `withTransaction(ctx, ...)`.

## The audit log — fail-closed by design

Every mutation emits an audit row inside the same transaction. From [`packages/services/src/status-report/create.ts:82-95`](../../packages/services/src/status-report/create.ts#L82-L95):

```ts
await emitAudit(tx, ctx, {
  action: "status_report.create",
  entityType: "status_report",
  entityId: newReport.id,
  after: newReport,
});

await emitAudit(tx, ctx, {
  action: "status_report_update.create",
  entityType: "status_report_update",
  entityId: initialUpdate.id,
  after: initialUpdate,
  metadata: { statusReportId: newReport.id },
});
```

Pattern:

- One audit row per mutation, *not* per request.
- Emitted **inside** the transaction. If the audit insert fails (FK violation, schema mismatch), the whole transaction rolls back. No write succeeds without an audit row.
- Action names follow `entity.verb` (`status_report.create`, `monitor.update`, `integration.delete`).
- The discriminated union of valid action names is defined in [`packages/db/src/schema/audit_logs/validation.ts`](../../packages/db/src/schema/audit_logs/validation.ts) (per [`CLAUDE.md`](../../CLAUDE.md)).

## `before` and `after` snapshots

For updates, pass both `before` (the pre-mutation row) and `after` (the result of `.returning()`):

```ts
const before = await tx.select().from(monitor).where(eq(monitor.id, id)).get();
const after  = await tx.update(monitor).set({...}).where(eq(monitor.id, id)).returning().get();
await emitAudit(tx, ctx, {
  action: "monitor.update",
  entityType: "monitor",
  entityId: id,
  before, after,
});
```

`emitAudit` diffs them and computes `changed_fields`. No-op updates are skipped.

For creates: only `after`. For deletes: only `before`.

## Strip secrets from snapshots

Audit rows are queryable. They're shown to admins, exported for compliance, archived for years. **Don't put secrets in them.**

Pattern from integration removal (cited in [`CLAUDE.md`](../../CLAUDE.md)):

```ts
const { credential, ...safeBefore } = integration;
await emitAudit(tx, ctx, { action: "integration.delete", before: safeBefore, ... });
```

Always-strip fields:
- API credentials.
- Bot tokens (Slack, Discord, PagerDuty).
- Raw API keys.
- Subscriber email addresses if your privacy policy treats them as PII.

Audit the audit log periodically. If you find a secret in there, you've already leaked.

## The `requireScope` order

```ts
export async function createStatusReport(args) {
  requireScope(ctx, "write");                    // 1
  const input = CreateStatusReportInput.parse(args.input);   // 2
  return withTransaction(ctx, async (tx) => {     // 3
    // ...
  });
}
```

Scope check → input parse → transaction open. Why this order?

- **Scope first** is the cheapest. A read-only API key trying to write should fail before we waste a parse or a connection.
- **Parse second** is also cheap and gives the caller useful error info.
- **Transaction last** because it holds a connection. We don't want to hold one for any longer than necessary.

If you find a service that opens the transaction first then checks scope, you're paying for a connection that you're about to throw away. Reorder.

## The "should this side effect be in the transaction?" question

Things that go *inside* the transaction:
- DB writes related to the entity.
- Association rows.
- The audit emit.

Things that go *outside* the transaction:
- Sending emails (can't roll back).
- HTTP webhooks (can't roll back).
- Pushing to a queue *for an external system*.
- Anything that touches a non-transactional system.

Pattern:

```ts
const result = await withTransaction(ctx, async (tx) => {
  // all DB work + audit emit
  return { ... };
});
// after commit: side effects
await sendNotification(result.id);
```

Or, better, the **outbox pattern**:

```ts
await withTransaction(ctx, async (tx) => {
  // ... write the entity
  await tx.insert(outbox).values({
    kind: "send_status_report_notification",
    payload: { reportId: newReport.id },
  });
});
// a background worker drains the outbox and reliably sends
```

The outbox row commits atomically with the write. A worker reads pending rows and dispatches. If the worker crashes, it retries. If dispatch fails permanently, the row is flagged for human review.

OpenStatus doesn't have a full outbox today (notification is triggered from the dashboard `onSuccess`). Moving to an outbox is on the TODO list ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:38-44`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L38-L44) — "TODO: move to server"). A senior engineer designing this feature today would build the outbox.

## What happens when the transaction fails

`db.transaction` semantics:

1. Drizzle opens a SQL transaction.
2. Your callback runs.
3. If it returns normally, Drizzle commits.
4. If it throws, Drizzle rolls back.

The thrown error propagates to the caller. The adapter maps it to a `TRPCError`. The client sees a typed failure.

If the *rollback itself* fails (rare), you have a half-state. Most modern DBs (including SQLite/libSQL) handle rollback robustly, but you should log unexpected errors from `withTransaction` aggressively.

## Read consistency inside a transaction

Reads inside the tx see the tx's own pending writes. Reads outside don't (until commit). This matters for the "validate then write" pattern:

```ts
return withTransaction(ctx, async (tx) => {
  const exists = await tx.select().from(page).where(eq(page.slug, slug)).get();
  if (exists) throw new ConflictError(...);
  await tx.insert(page).values({ slug, ... }).run();
});
```

Race condition: two requests run this concurrently, both see "doesn't exist," both insert. Result: duplicate (or unique constraint violation).

Two fixes:

1. **DB unique constraint** + catch the violation. Let the DB be the arbiter.
2. **`SELECT ... FOR UPDATE`** to lock. Doesn't work in SQLite/libSQL.

OpenStatus uses #1. The schema declares unique indexes; the service catches the violation and converts it to a `ConflictError`. Robust, no race window.

## Test transactions — `withTestTransaction`

OpenStatus's test helper ([`packages/services/test/helpers.ts:37-50`](../../packages/services/test/helpers.ts#L37-L50)):

```ts
export async function withTestTransaction<T>(fn: (tx: DrizzleTx) => Promise<T>): Promise<T> {
  let result!: T;
  try {
    await db.transaction(async (tx) => {
      result = await fn(tx);
      throw new RollbackSignal();    // intentional throw to rollback
    });
  } catch (e) {
    if (!(e instanceof RollbackSignal)) throw e;
  }
  return result;
}
```

Pattern: open a real transaction, run the test, throw a sentinel error, swallow it after rollback. The test's writes never commit to disk; the next test starts clean. No mocks, no test DB reset per test, fast and accurate.

This pattern requires that the code under test uses `withTransaction` properly — if it opens its own transaction inside, you get nested-tx errors. Hence the entire architecture's commitment to threading `ctx.db` everywhere.

## Pitfall checklist

1. **Multi-row write without a transaction.** Half-state on failure.
2. **`db.transaction(...)` directly** in a service instead of `withTransaction`. Breaks composition.
3. **Audit emit outside the tx.** Writes can succeed without an audit row.
4. **Audit emit before the write returns.** Now you've audited something that doesn't exist yet.
5. **Secret in audit `before`/`after`.** Permanent leak.
6. **Side effect inside the tx** (email, webhook). Can't roll back.
7. **`requireScope` after the tx opens.** Wasted connection.
8. **`SELECT then INSERT` race** without a unique constraint.
9. **Forgetting `.returning().get()`** after an insert — you can't audit what you didn't read back.
10. **Long-running tx.** Holds locks, blocks other writes. Keep tx bodies fast.

## Drills

- Find every `emitAudit` call. Verify it's inside a transaction.
- Find a delete service. Verify the `before` snapshot is captured *before* the delete runs.
- Pick a mutation that has a side effect (notification). Sketch an outbox refactor: what's the new table, what's the worker, how do retries work?
- Pull up [`packages/services/src/status-report/delete.ts`](../../packages/services/src/status-report/delete.ts) (if it exists) and identify: scope check, ownership check, audit, cascade behavior.
- Find a "validate then insert" pattern that relies on a unique constraint. Identify the race window the DB closes.
