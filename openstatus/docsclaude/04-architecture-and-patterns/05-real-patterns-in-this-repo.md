# 05 — Real Patterns in This Repo

A catalog of the most reusable patterns you'll see in OpenStatus, with file anchors and a sentence each on why they exist. The goal is recognition: when you see a similar problem in a different codebase, you'll reach for the same shape.

## Pattern: Discriminated union for actor identity

**Where:** [`packages/services/src/context.ts:13-20`](../../packages/services/src/context.ts#L13-L20).

```ts
export type Actor =
  | { type: "user"; userId: number }
  | { type: "apiKey"; keyId: string; userId?: number; scopes: Scope[] }
  | { type: "mcp"; keyId: string; userId?: number; scopes: Scope[] }
  | { type: "slack"; teamId: string; slackUserId: string; userId?: number }
  | { type: "system"; job: string }
  | { type: "webhook"; source: string; externalId?: string }
  | { type: "subscriber"; subscriberId: number };
```

**Why:** the service layer needs to authorize many actor types — humans, API keys, integrations. A discriminated union encodes "exactly one of these" and lets TypeScript verify exhaustive handling (see [`packages/services/src/context.ts:60-76`](../../packages/services/src/context.ts#L60-L76) for the exhaustive switch).

**Use it when:** you have N+1 things that are *kinds* of the same concept and behavior branches per kind.

## Pattern: Adapter at the transport boundary

**Where:** [`packages/api/src/service-adapter.ts`](../../packages/api/src/service-adapter.ts).

```ts
export function toServiceCtx(ctx: AuthedContext): ServiceContext { ... }
export function toTRPCError(err: unknown): never { ... }
```

**Why:** keeps services transport-agnostic. Every router can call `toServiceCtx(ctx)` and `toTRPCError(err)`, and the service doesn't know tRPC exists.

**Use it when:** you want to be able to expose the same service through multiple transports (tRPC, REST, MCP, background workers).

## Pattern: `withTransaction` for composable atomicity

**Where:** [`packages/services/src/context.ts:40-47`](../../packages/services/src/context.ts#L40-L47).

```ts
export async function withTransaction<T>(ctx: ServiceContext, fn: (tx: DB) => Promise<T>): Promise<T> {
  const db = ctx.db ?? defaultDb;
  if (isTx(db)) return fn(db);
  return (db as DrizzleClient).transaction(fn);
}
```

**Why:** a service that calls another service can share the outer transaction. The inner sees `ctx.db = tx` and reuses it; otherwise it opens a fresh tx.

**Use it when:** you have nested business operations that must be atomic together.

## Pattern: Fail-closed audit emit inside the transaction

**Where:** every mutation. Example: [`packages/services/src/status-report/create.ts:82-95`](../../packages/services/src/status-report/create.ts#L82-L95).

**Why:** the audit row commits with the mutation. If the audit insert fails, the mutation rolls back. There can be no successful write without a record of it.

**Use it when:** you have compliance/audit requirements, or want to be able to answer "who did what, when, in what context."

## Pattern: One file per service verb

**Where:** [`packages/services/src/status-report/`](../../packages/services/src/status-report/).

```
status-report/
  create.ts
  list.ts
  update.ts
  add-update.ts
  remove.ts
  notify.ts
  resolve.ts
  schemas.ts
  internal.ts
  index.ts
```

**Why:** small files, focused tests, easy to find. The `index.ts` re-exports. Consumers `import { createStatusReport } from "@openstatus/services/status-report"`.

**Use it when:** an entity has more than ~3 mutations. Keeps each file readable and pushes shared helpers to `internal.ts`.

## Pattern: Schema as the single source of truth

**Where:** [`packages/db/src/schema/status_reports/status_reports.ts:8-13`](../../packages/db/src/schema/status_reports/status_reports.ts#L8-L13).

```ts
export const statusReportStatus = ["investigating", "identified", "monitoring", "resolved"] as const;
```

Imported by: DB schema, service schema, form, audit log validation, public render.

**Why:** rename or extend a value in one place, get a TypeScript error wherever it doesn't match.

**Use it when:** the same set of values appears in more than two places.

## Pattern: Wire input differs from domain input

**Where:** [`packages/api/src/router/statusReport.ts:24-32, 64-74`](../../packages/api/src/router/statusReport.ts#L24-L74).

The router accepts `pageComponents` (legacy dashboard name); the service expects `pageComponentIds`. The router adapts.

**Why:** the dashboard contract is older than the service contract. Renaming the service is cheap; renaming the wire field would require coordinated changes to every dashboard caller (and any other clients).

**Use it when:** you're modernizing a domain layer while keeping a stable external API.

## Pattern: `requireScope` at the top of every write verb

**Where:** [`packages/services/src/auth/require-scope.ts`](../../packages/services/src/auth/require-scope.ts).

```ts
requireScope(ctx, "write");
```

**Why:** RBAC at the service layer means every transport (tRPC, REST, MCP) gets the same enforcement. Putting it in routers would mean re-checking in every router.

**Use it when:** you have actor-level permissions (API key scopes, role-based auth).

## Pattern: `getReadDb` for read-side queries

**Where:** [`packages/services/src/context.ts:56-58`](../../packages/services/src/context.ts#L56-L58).

```ts
export function getReadDb(ctx: ServiceContext): DB {
  return ctx.db ?? defaultDb;
}
```

**Why:** reads inside a write transaction should see in-flight writes. Reads outside don't need to open a tx. One helper covers both cases.

**Use it when:** your services have both read and write verbs and you want reads to compose cleanly inside writes.

## Pattern: Batched enrichment instead of N+1

**Where:** [`packages/services/src/status-report/list.ts:72-162`](../../packages/services/src/status-report/list.ts#L72-L162).

Load all reports, then run three batched queries: updates, components, pages. Build maps. Stitch in JS.

**Why:** O(1) queries regardless of list size. Critical for dashboards that list many entities.

**Use it when:** you have a list endpoint that needs nested data and you want predictable performance.

## Pattern: `fetchQueryOrNotFound` for resource-gated layouts

**Where:** [`apps/dashboard/src/lib/trpc/server.tsx:103-117`](../../apps/dashboard/src/lib/trpc/server.tsx#L103-L117).

A server component's layout fetches a resource; if missing, calls Next's `notFound()`. The child route only renders for valid resources.

**Why:** centralizes the "does this exist?" check at the route level instead of every leaf page checking.

**Use it when:** you have nested dashboard routes under a parent resource.

## Pattern: Server prefetch + client hydrate

**Where:** [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx#L16-L35), [`apps/dashboard/src/lib/trpc/server.tsx:59-67`](../../apps/dashboard/src/lib/trpc/server.tsx#L59-L67).

Server component prefetches a tRPC query into a per-request `QueryClient`. `HydrateClient` ships the cache to the browser. The client component finds the cache pre-populated.

**Why:** zero network round-trip on first render. Best of SSR + SPA.

**Use it when:** you're using React Query with App Router and want fast first paint.

## Pattern: `enabled` to gate React Query

**Where:** [`apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:64-78`](../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L64-L78).

```ts
useQuery({ ...trpc.tinybird.globalMetrics.queryOptions({...}), enabled: monitorIds.length > 0 });
```

**Why:** don't fire a query until its inputs are valid.

**Use it when:** a query depends on data that may not be available on first render.

## Pattern: `useTransition` for non-blocking submit

**Where:** [`apps/dashboard/src/components/forms/status-report/form.tsx:108-137`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L108-L137).

```ts
startTransition(async () => {
  const promise = onSubmit(values);
  toast.promise(promise, { ... });
  await promise;
});
```

**Why:** the UI stays responsive while the network call flies; `isPending` drives the disabled state.

**Use it when:** your submit handler is async and may take a noticeable amount of time.

## Pattern: `toast.promise` for unified async feedback

**Where:** any form. Example above.

**Why:** loading toast → success or error toast, all from one promise. Consistent UX without per-state plumbing.

**Use it when:** you want a single status notification per async user action.

## Pattern: Test-time transaction rollback

**Where:** [`packages/services/test/helpers.ts:37-50`](../../packages/services/test/helpers.ts#L37-L50).

Open a real tx, run the test, throw a sentinel, roll back, swallow the sentinel.

**Why:** test isolation without per-test DB resets. Tests run against a real DB without polluting state.

**Use it when:** you have a real DB available in tests and want fast, accurate integration tests.

## Pattern: Custom tRPC link for runtime routing

**Where:** [`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77).

Inspect the procedure path and route to one of multiple HTTP endpoints (`/api/trpc/edge` or `/api/trpc/lambda`).

**Why:** different procedures need different runtimes; the client decides at call time.

**Use it when:** your backend is split across multiple deployment targets and you want one tRPC client to talk to all of them.

## Pattern: Centralized cast helper for unsafe type holes

**Where:** [`apps/dashboard/src/components/chat/use-chat-session.ts`](../../apps/dashboard/src/components/chat/use-chat-session.ts) (per [`CLAUDE.md`](../../CLAUDE.md)).

```ts
function asUIMessages(rows): UIMessage[] {
  // Safe because storedMessageSchema validates on write.
  return rows as unknown as UIMessage[];
}
```

**Why:** unavoidable casts get centralized in one named function with a comment explaining the runtime guarantee. Avoids scattered `as never`s.

**Use it when:** you need a cast and the safety reasoning is non-obvious.

## How to recognize a pattern in unfamiliar code

When you read a new codebase, ask:

1. Where is the boundary between transport and business logic?
2. What's the service contract shape?
3. How are transactions composed?
4. Where does authorization live?
5. How is the audit / change history captured?
6. How is multi-tenancy enforced?
7. What's the validation strategy at each layer?
8. How do mutations invalidate caches?
9. How is the read model derived?
10. How is the test isolation achieved?

If you can answer those ten questions in 30 minutes of reading, you understand the architecture. The shape of the answers — not the specific files — is what carries between projects.

## Drills

- For each pattern above, find one *more* example in this repo than the one I cited.
- Pick a different popular open-source TypeScript repo (cal.com, dub.co, supabase). Try to recognize the same patterns. Where do they diverge? Which divergences are better, which worse?
- Write a one-page "architecture document" for OpenStatus using these patterns as the structure. Could a new team member onboard from your doc in an afternoon?
