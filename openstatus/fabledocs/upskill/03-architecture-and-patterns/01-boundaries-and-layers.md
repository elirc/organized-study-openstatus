# Boundaries & Layers

## The intended layering (dashboard path)

```
UI (React) → tRPC router → service verb → Drizzle → libSQL
                 │              │
            transport map   audit + scopes + tenancy (the invariants live HERE)
```

What each layer owns — and must not own:

| Layer | Owns | Must not own |
| --- | --- | --- |
| UI components | presentation, optimistic state, cache invalidation | authorization decisions, quota math |
| tRPC routers | wire schema, error translation, analytics meta | business rules, direct DB writes |
| Services | invariants: tenancy, quota, audit, scopes | HTTP/status codes, React-isms |
| db package | schema, migrations, the client | business rules |
| Workflows/checker | pipeline orchestration | (currently they *do* own direct DB writes — see leaks) |

## Boundaries done well

1. **Service ↔ transport.** Services throw `ServiceError` codes; each transport translates ([service-adapter.ts:32-87](../../../packages/api/src/service-adapter.ts#L32-L87)). The service never imports `TRPCError` — dependency direction enforced by package boundaries.
2. **Actor abstraction.** Seven caller kinds funnel into one `Actor` union ([context.ts:13-20](../../../packages/services/src/context.ts#L13-L20)); downstream code (audit attribution, scope checks) branches on the union, never on "which route called me."
3. **Pure decision core in the proxy.** `composePageAction` takes all inputs (cookies, IP, session, page row) and returns an action value ([proxy.ts:64-86](../../../apps/status-page/src/proxy.ts#L64-L86)) — the I/O shell around a pure gate. Testable without HTTP.
4. **TS ↔ Go boundary is data-only.** No shared code; the contract is a JSON payload typed independently on both sides ([cron/checker.ts:249-320](../../../apps/workflows/src/cron/checker.ts#L249-L320) ↔ Go structs). Cost: drift risk; benefit: independent deploys and languages.

## Boundary leaks (real, with evidence)

1. **v1 REST handlers do inline Drizzle** — quota logic duplicated, audit skipped ([post.ts:44-90](../../../apps/server/src/routes/v1/monitors/post.ts#L44-L90)). Acknowledged in-code as transitional ([v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147)).
2. **Workflows writes domain state directly** — `/updateStatus` mutates `monitor.status` and inserts incidents around the service layer ([checker/index.ts:227-316](../../../apps/workflows/src/checker/index.ts#L227-L316)); the free-plan pause flips `monitor.active` the same way ([cron/monitor.ts:292-303](../../../apps/workflows/src/cron/monitor.ts#L292-L303)). Consequence: those mutations produce **no `audit_log` rows** and re-implement scoping ad hoc. Defensible (throughput, system actor), but it means "every mutation is audited" is only true for *user-facing* mutations — precision matters when you state invariants.
3. **Dashboard ↔ service schema duplication** — wire and domain schemas deliberately decoupled ([monitor.ts:39-61](../../../packages/api/src/router/monitor.ts#L39-L61)); a *managed* leak: the boundary exists but must be maintained by hand.

## Transferable: how to find layer leaks in any repo

Grep for the persistence import outside its layer (`from "@openstatus/db"` in `apps/server/src/routes/` — hits are leaks or migrations-in-progress), and for transport types inside domain code (`TRPCError` under `packages/services/` — should be zero; verify).

## Drill

Pick `packages/services/src/status-report/` (not covered elsewhere in these docs). Map its verbs to the layer table above: does every write follow scope → parse → tx → guard → write → audit? Note any deviation and classify it: leak, managed exception, or bug-suspicion.
Self-grade — Basic: verb list. Solid: per-verb conformance table. Strong: one paragraph on whether status-report's shape justifies any deviation (e.g. subscriber notification side effects).

## Interview angle

- "Describe the layering of an app you know" — recite the table, then *volunteer a leak and its justification*. Mid-level candidates describe clean architecture; strong candidates describe real architecture including its managed exceptions ([08/03 Q7](../08-interview-prep/03-api-and-data-modeling-questions.md)).
