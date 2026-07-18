# 03 — Databases with Drizzle

The data layer is where junior bugs cost the most: silent N+1s, missing transactions, accidentally cross-tenant reads. Drizzle is a thin, type-safe layer over SQL — you still need to know SQL.

## Drizzle is not an ORM. It's a typed query builder.

You write something like SQL; you get types from your schema for free. There's no "lazy loading," no "active record." You think in SQL.

```ts
const rows = await db
  .select({ id: page.id, title: page.title })
  .from(page)
  .where(eq(page.workspaceId, workspaceId))
  .all();
```

That's literally `SELECT id, title FROM page WHERE workspace_id = ?`. Drizzle types `rows` as `{ id: number; title: string }[]` based on the schema. No surprises.

## Schema definition

[`packages/db/src/schema/status_reports/status_reports.ts`](../../packages/db/src/schema/status_reports/status_reports.ts):

```ts
export const statusReport = sqliteTable(
  "status_report",
  {
    id: integer("id").primaryKey(),
    status: text("status", { enum: statusReportStatus }).notNull(),
    title: text("title", { length: 256 }).notNull(),
    workspaceId: integer("workspace_id").references(() => workspace.id),
    pageId: integer("page_id").references(() => page.id, { onDelete: "cascade" }),
    createdAt: integer("created_at", { mode: "timestamp" }).default(
      sql`(strftime('%s', 'now'))`,
    ),
    updatedAt: integer("updated_at", { mode: "timestamp" }).default(
      sql`(strftime('%s', 'now'))`,
    ),
  },
  (t) => [
    index("status_report_workspace_created_idx").on(t.workspaceId, t.createdAt),
    index("status_report_page_id_idx").on(t.pageId),
  ],
);
```

Notice:
- Column types map to SQL types (`integer`, `text`).
- `mode: "timestamp"` tells Drizzle to convert between SQL `INTEGER` (Unix seconds) and JS `Date`.
- Foreign keys are declared with `.references(() => other.col, { onDelete: "cascade" })`.
- Indexes go in the second argument; they're SQL-level constructs the table needs for performance.
- Enums are *string literal unions* (`statusReportStatus`) — the same `as const` array used by the form and the audit log.

## Relations (read-side helpers)

```ts
export const StatusReportRelations = relations(statusReport, ({ one, many }) => ({
  statusReportsToPageComponents: many(statusReportsToPageComponents),
  page: one(page, { fields: [statusReport.pageId], references: [page.id] }),
  statusReportUpdates: many(statusReportUpdate),
  workspace: one(workspace, { fields: [statusReport.workspaceId], references: [workspace.id] }),
}));
```

Relations are a *query helper*, not a runtime constraint. They let you use `db.query.statusReport.findMany({ with: { statusReportUpdates: true } })` to load nested data. They're optional — you can always write joins by hand.

## The two query styles

**Builder style** (explicit, more control):

```ts
const rows = await db
  .select()
  .from(statusReport)
  .leftJoin(statusReportUpdate, eq(statusReportUpdate.statusReportId, statusReport.id))
  .where(eq(statusReport.workspaceId, wsId))
  .all();
// rows: { status_report: ..., status_report_update: ... }[]
```

**Relational style** (terser, uses relation helpers):

```ts
const rows = await db.query.statusReport.findMany({
  where: (r, { eq }) => eq(r.workspaceId, wsId),
  with: { statusReportUpdates: true, page: true },
});
// rows: full nested objects
```

Both are valid. Relational style is more readable; builder style is more flexible. The repo uses both depending on the case.

## `get`, `all`, `run` — terminators

Drizzle queries are lazy until you call a terminator:

- `.get()` — returns the first row or `undefined`.
- `.all()` — returns all rows.
- `.run()` — runs (for inserts/deletes that don't return).
- `.returning()` — for inserts/updates, returns the affected rows.

```ts
const newReport = await tx
  .insert(statusReport)
  .values({ ... })
  .returning()
  .get();
// newReport is the inserted row with auto-assigned id
```

## Transactions — atomicity is mandatory for multi-row writes

A status report create writes to three tables:
1. `status_report` (the report header).
2. `status_report_update` (the initial update).
3. `status_report_to_page_component` (component associations).

If any of those fail mid-way without a transaction, you have a half-created report polluting the DB. Wrap the writes in `db.transaction(...)`:

```ts
await db.transaction(async (tx) => {
  const report = await tx.insert(statusReport).values({...}).returning().get();
  await tx.insert(statusReportUpdate).values({...}).run();
  // any throw rolls back everything
});
```

OpenStatus wraps this in `withTransaction` ([`packages/services/src/context.ts:40-47`](../../packages/services/src/context.ts#L40-L47)):

```ts
export async function withTransaction<T>(
  ctx: ServiceContext,
  fn: (tx: DB) => Promise<T>,
): Promise<T> {
  const db = ctx.db ?? defaultDb;
  if (isTx(db)) return fn(db);                         // already inside a tx — reuse it
  return (db as DrizzleClient).transaction(fn);
}
```

Pattern: if the caller passed a transaction (e.g., a higher-level service composing this one), reuse it. Otherwise, open a fresh one. This composability is what lets you call services from within services without losing atomicity.

## The N+1 problem and how this repo avoids it

Naive:

```ts
const reports = await db.select().from(statusReport).all();
for (const r of reports) {
  r.updates = await db.select().from(statusReportUpdate).where(eq(..., r.id)).all();
}
// 1 query + N queries = N+1 queries
```

If `reports.length` is 100, that's 101 round-trips.

The OpenStatus pattern, from [`packages/services/src/status-report/list.ts:72-162`](../../packages/services/src/status-report/list.ts#L72-L162):

```ts
async function enrichReportsBatch(db, rows) {
  if (rows.length === 0) return [];
  const reportIds = rows.map((r) => r.id);

  // One query: all updates for all reports
  const allUpdates = await db
    .select()
    .from(statusReportUpdate)
    .where(inArray(statusReportUpdate.statusReportId, reportIds))
    .orderBy(desc(statusReportUpdate.date))
    .all();

  // Build an in-memory map: reportId -> updates[]
  const updatesByReport = new Map();
  for (const u of allUpdates) {
    const arr = updatesByReport.get(u.statusReportId);
    if (arr) arr.push(u); else updatesByReport.set(u.statusReportId, [u]);
  }
  // ... same for component associations
}
```

Three batched queries regardless of how many reports. The mapping happens in JS. This is the right pattern for any list that needs nested data — *especially* when the list is "effectively unlimited" like this one (limit 10,000).

**Senior aside:** *N+1 is almost always silent. It works in dev (small data, fast DB). It crashes in prod (real data, network latency). Every list query you write should have a "how many round-trips does this make" answer before merging.*

## Indexes

An index is a sorted data structure for fast lookups. Without an index, a `WHERE workspace_id = ?` scans every row. With an index on `workspace_id`, it's O(log n).

Rules of thumb:

- Index every foreign key you query by.
- Index the columns in your `WHERE` and `ORDER BY` clauses.
- Composite indexes match left-to-right: `index(workspaceId, createdAt)` helps `WHERE workspace_id = ? ORDER BY created_at` but not `WHERE created_at = ?`.
- Too many indexes slow down writes. Don't index speculatively.

OpenStatus's `statusReport` table has two indexes ([`packages/db/src/schema/status_reports/status_reports.ts:35-38`](../../packages/db/src/schema/status_reports/status_reports.ts#L35-L38)) — one composite on `(workspaceId, createdAt)` for the dashboard list, one on `pageId` for the public read path. Those are exactly the queries that need them.

## libSQL / Turso specifics

OpenStatus uses Turso (a hosted libSQL — SQLite over HTTP/WebSockets).

- **HTTP driver** (`drizzle-orm/libsql/http`) — works in Edge runtime. Stateless. Each query is a fetch. Slightly higher per-query latency.
- **Native driver** (`drizzle-orm/libsql`) — TCP-based, lower latency, doesn't work in Edge.

Production deploys the dashboard's API to Edge → must use the HTTP driver. Hence [`packages/db/src/db.ts:1`](../../packages/db/src/db.ts#L1):

```ts
import { drizzle } from "drizzle-orm/libsql/http";
```

Implication for transactions: libSQL over HTTP doesn't support savepoints. Nested transactions don't work. OpenStatus's `withTransaction` reuses an outer tx exactly because *opening a nested one would fail*. The test helper ([`packages/services/test/helpers.ts:37-50`](../../packages/services/test/helpers.ts#L37-L50)) uses a "throw to rollback" pattern for the same reason.

## Migrations

Drizzle generates migrations from schema diffs:

```sh
pnpm db:generate    # produces sql migration files
pnpm db:migrate     # applies them
```

In OpenStatus, the [`packages/db/`](../../packages/db/) package owns the schema and migrations. The Turbo `dx` task chains `env → migrate → seed` ([`turbo.json`](../../turbo.json)) so a fresh checkout gets a working DB in one command.

Best practices:

- One PR = one migration. Don't lump multiple schema changes together.
- Make migrations backwards-compatible when possible (additive columns first, drop columns later).
- Test migrations against a copy of prod data before applying.
- *Always* have a rollback plan, even if it's "restore from backup."

## Pitfall checklist

1. **Missing workspace filter** in a query. Cross-tenant leak. Always start `WHERE` clauses with `eq(table.workspaceId, ctx.workspace.id)`.
2. **N+1 in a list.** Use `inArray` + a map.
3. **Forgotten transaction** around related writes. Half-written data.
4. **`.returning()` without `.get()` or `.all()`.** Drizzle returns a lazy builder; you need a terminator.
5. **`new Date()` in a default at JS time** instead of at DB time. Different timestamps per row vs one timestamp at insert. Use `sql\`(strftime('%s', 'now'))\`` for DB-side defaults.
6. **Timestamp columns without `mode: "timestamp"`.** You'll get raw integers and have to convert manually everywhere.
7. **Schema changes without migrations.** Dev works (regenerated DB); CI/prod doesn't.
8. **Composite index in the wrong order.** Doesn't help your query.
9. **Querying inside a loop.** N+1.
10. **`onDelete: "cascade"` reflexively.** Cascading deletes can wipe out audit data, notification history, etc. Be explicit about what you want.

## Drills

- Read [`packages/services/src/status-report/list.ts:72-162`](../../packages/services/src/status-report/list.ts#L72-L162). Count the DB queries it makes. Compare with a naive implementation that loads each report's relations separately.
- Find every table in the schema that has `workspaceId`. Verify the related service file always filters by it.
- Add a missing index to a table that you think needs one. (Hint: find a service that filters by a foreign key without an index on it.) Generate a migration. Inspect the SQL.
- Inspect [`packages/db/src/db.ts`](../../packages/db/src/db.ts). What happens if you change `drizzle-orm/libsql/http` to `drizzle-orm/libsql`? Where would it break?
- Pick a relational `findMany` with `with:` clauses (e.g., `statusPage.get`). Rewrite it in builder style with explicit joins. Compare ergonomics.
