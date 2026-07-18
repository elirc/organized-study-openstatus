# Fake-Code Contrasts

Ten bad-vs-better pairs. **Every snippet here is fake and labeled** — the "better" versions mirror real repo patterns (anchored). The bad versions are the instincts a junior brings from tutorials; learn to *see* them.

---

## 1. Business logic in the router

```ts
// Illustrative fake code: not from this repo
export const monitorRouter = createTRPCRouter({
  create: protectedProcedure.input(input).mutation(async ({ ctx, input }) => {
    const count = await ctx.db.select().from(monitor)
      .where(eq(monitor.workspaceId, ctx.workspace.id));
    if (count.length >= 10) throw new TRPCError({ code: "FORBIDDEN" });
    return ctx.db.insert(monitor).values({ ...input, workspaceId: ctx.workspace.id });
  }),
});
```

Problems: hardcoded quota, no audit, no reuse for REST/Slack/MCP, `FORBIDDEN` for a billing condition, loads all rows to count.
Better shape: thin router calling the service verb — the real thing at [monitor.ts:77-93](../../../packages/api/src/router/monitor.ts#L77-L93) + [create.ts:16-63](../../../packages/services/src/monitor/create.ts#L16-L63).

## 2. Missing tenant filter (IDOR)

```ts
// Illustrative fake code: not from this repo
export async function getMonitor({ input }: { input: { id: number } }) {
  const row = await db.select().from(monitor).where(eq(monitor.id, input.id)).get();
  if (!row) throw new NotFoundError("monitor", input.id);
  return row; // whoever asked, gets it
}
```

Problem: any authenticated caller reads any tenant's monitor by iterating ids.
Better: fetch-or-throw scoped helper — [internal.ts:28-47](../../../packages/services/src/monitor/internal.ts#L28-L47) (`AND workspaceId AND deletedAt IS NULL`, 404 for foreign rows: no existence oracle).

## 3. N+1 in a fan-out loop

```ts
// Illustrative fake code: not from this repo
for (const m of monitors) {
  const statuses = await db.select().from(monitorStatusTable)
    .where(eq(monitorStatusTable.monitorId, m.id)).all(); // 1 query × N monitors
  buildTasks(m, statuses);
}
```

Better: one `inArray` batch + in-memory Map — exactly what the repo did, self-labeled "N+1 fix" at [cron/checker.ts:127-151](../../../apps/workflows/src/cron/checker.ts#L127-L151). (Counter-drill: find the *remaining* per-notification queries in [alerting.ts:70-121](../../../apps/workflows/src/checker/alerting.ts#L70-L121) — the pattern isn't applied everywhere.)

## 4. Retry without idempotency

```ts
// Illustrative fake code: not from this repo
await retry(() => sendPagerDuty(alert), { times: 3 });
await retry(() => queue.enqueue(task), { times: 3 }); // task has random id
```

Problem: retries of non-idempotent effects multiply pages and jobs.
Better: derive identity from business facts *first*, then retry freely — deterministic task names ([cron/checker.ts:336](../../../apps/workflows/src/cron/checker.ts#L336)) and trigger-row dedupe ([alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139)).

## 5. Swallowed errors

```ts
// Illustrative fake code: not from this repo
try {
  await emitAudit(tx, ctx, entry);
} catch {
  // audit is best-effort, don't block the user
}
```

Problem: silently converts an invariant ("every mutation audited") into a hope. Nobody notices until compliance asks.
Better: fail-closed on purpose, documented — [emit.ts:92-101](../../../packages/services/src/audit/emit.ts#L92-L101). Swallowing is legal only when *explicit and intentional*, like the idempotent delete at [monitor.ts:87-92](../../../packages/api/src/router/monitor.ts#L87-L92) — note it swallows exactly one typed error, with a comment.

## 6. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
// page.tsx renders row.assertions (raw JSON text) and row.regions.split(",")
<td>{JSON.parse(monitor.assertions)[0]?.type}</td>
<td>{monitor.regions.split(",").join(", ")}</td>
```

Problem: every consumer re-implements deserialization; a storage-format change breaks N screens.
Better: transform once at the read boundary — `selectMonitorSchema` turns storage text into typed arrays so consumers see domain shapes ([create.ts:61](../../../packages/services/src/monitor/create.ts#L61); schema transforms in `packages/db/src/schema/monitors/validation.ts`).

## 7. Secrets in snapshots/logs

```ts
// Illustrative fake code: not from this repo
await emitAudit(tx, ctx, { action: "page_subscriber.create", after: insertedRow }); // row includes `token`
console.log("validated key", key);
```

Better: strip capabilities before persisting snapshots — `const { token: _t, ...after } = …` at [upsert.ts:216](../../../packages/services/src/page-subscriber/upsert.ts#L216); log key *ids*, never key material ([require-scope.ts:42-45](../../../packages/services/src/auth/require-scope.ts#L42-L45) logs keyId).

## 8. `any` at the boundary

```ts
// Illustrative fake code: not from this repo
const payload: any = await c.req.json();
await upsertMonitorStatus({ monitorId: payload.monitorId, status: payload.status, region: payload.region });
```

Problem: a malformed queue message writes garbage into `monitor_status` and you learn about it days later.
Better: parse to a typed value, reject with 422 — [checker/index.ts:94-110](../../../apps/workflows/src/checker/index.ts#L94-L110).

## 9. Casual public-contract change

```ts
// Illustrative fake code: not from this repo
// "cleanup": rename response field in v1
const MonitorSchema = z.object({ identifier: z.number(), /* was: id */ ... });
```

Problem: every external client breaks; OpenAPI consumers pin generated types.
Better: additive evolution + explicit versioning — the repo runs `/v1` (deprecation-noted in its OpenAPI description, [v1/index.ts:52-54](../../../apps/server/src/routes/v1/index.ts#L52-L54)) alongside a v2 RPC surface; wire schemas are their own artifacts precisely so internal refactors don't leak ([monitor.ts:39](../../../packages/api/src/router/monitor.ts#L39)).

## 10. Stale-closure state sync

```tsx
// Illustrative fake code: not from this repo
useEffect(() => {
  setColumnFilters([{ id: "status", value: [searchParams.status] }]);
}, []); // runs once; later URL changes silently ignored
```

Problem: mount-only sync means back/forward navigation and shared links desync from the table.
Real cousin: the repo does exactly this, *knowingly*, with a TODO and lint suppression ([client.tsx:80-90](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L80-L90)). Better shape: derive filters from URL state on every render (nuqs values are already reactive), or subscribe to param changes. The lesson isn't "the repo is wrong" — it's *recognize the compromise and know the exit*.

---

Drill: for contrasts 2, 4, and 5, write (from memory, in 3 lines each) the review comment you'd leave if the bad version appeared in a PR. Kind, specific, with the anchor. Compare tone against [07/01-code-review-mindset.md](../07-career-and-collaboration/01-code-review-mindset.md).
