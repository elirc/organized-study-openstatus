# 01 — Reading Katas

10 short reading exercises. For each one, do the reading, write a 5-sentence summary, then compare against the rubric. The summaries are *your* notes, not for anyone else; the point is to force precision.

## Kata 1 — The tRPC handler

**Read:** [`apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts`](../../apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts).

**Question:** what does this file do? In what order do the steps happen? What's *not* in this file that you might expect to see?

**Rubric:**
- A strong summary names: `guardTRPCSource`, `fetchRequestHandler`, `createTRPCContext`, `auth`, `onError`.
- A strong summary notes that auth lives in `createTRPCContext`, not in this file.
- A strong summary mentions the convention-not-security comment on `guardTRPCSource`.

## Kata 2 — The protected procedure

**Read:** [`packages/api/src/trpc.ts:148-241`](../../packages/api/src/trpc.ts#L148-L241).

**Question:** list every side effect and every control-flow branch.

**Rubric:**
- Side effects: throws on no session, throws on workspace resolution failure, sets cookie if slug doesn't match, schedules analytics via `after()`.
- Branches: NODE_ENV test escape hatch, resolved.ok check, cookie mismatch, result.ok in tracking branch.

## Kata 3 — The service create

**Read:** [`packages/services/src/status-report/create.ts`](../../packages/services/src/status-report/create.ts).

**Question:** in what order do the six steps run? What invariants does each enforce?

**Rubric:**
- Order: requireScope → parse → withTransaction → ownership check → component validation → insert report → associate components → insert update → audit emits.
- Invariants: scope before connection; input validated before any DB hit; ownership before any write; component IDs validated against workspace + page; transaction wraps everything; audit can't be skipped.

## Kata 4 — The list with batched enrichment

**Read:** [`packages/services/src/status-report/list.ts:66-207`](../../packages/services/src/status-report/list.ts#L66-L207).

**Question:** how many DB queries does a list of 100 reports cause? How does that scale?

**Rubric:**
- 1 (count) + 1 (rows) + 1 (updates) + 1 (associations) + 1 or 2 (pages + their components) = 5–6 queries regardless of list size.
- Naive: 1 + 100 (each report's updates) + 100 (each report's components) = O(N) — would be 201 queries.

## Kata 5 — The wire-to-domain adapter

**Read:** [`packages/api/src/router/statusReport.ts:21-81`](../../packages/api/src/router/statusReport.ts#L21-L81).

**Question:** name every difference between the tRPC input shape and the service input shape. Why does each difference exist?

**Rubric:**
- `pageComponents` vs `pageComponentIds` — legacy dashboard field name vs domain name.
- `notifySubscribers` — UI-only behavior, not part of the service create.
- Why: keep wire backwards-compatible while domain stays canonical.

## Kata 6 — The error adapter

**Read:** [`packages/api/src/service-adapter.ts:32-87`](../../packages/api/src/service-adapter.ts#L32-L87).

**Question:** list every error class handled. What unique mapping is there?

**Rubric:**
- Handled: `TRPCError` (re-throw), `ZodError`, `ServiceError` subclasses (`NOT_FOUND`, `FORBIDDEN`, `UNAUTHORIZED`, `CONFLICT`, `VALIDATION`, `LIMIT_EXCEEDED`, `PRECONDITION_FAILED`, `INTERNAL`).
- Unique mapping: `LIMIT_EXCEEDED` → `TOO_MANY_REQUESTS` (not FORBIDDEN). The comment explains why.

## Kata 7 — The withTransaction

**Read:** [`packages/services/src/context.ts:40-47`](../../packages/services/src/context.ts#L40-L47).

**Question:** explain the two branches. What's the consequence if a service writes `db.transaction(...)` directly instead?

**Rubric:**
- Branch 1: if ctx.db is already a transaction (`isTx`), reuse it.
- Branch 2: otherwise open a fresh transaction.
- Consequence of `db.transaction(...)` directly: libsql HTTP doesn't support savepoints, so a nested tx would fail. The "reuse if already in tx" branch prevents this.

## Kata 8 — The IDOR test

**Read:** [`packages/api/src/router/statusReport.test.ts:77-100`](../../packages/api/src/router/statusReport.test.ts#L77-L100).

**Question:** describe the attack this test prevents. Where exactly is the rejection enforced?

**Rubric:**
- Attack: a user in workspace 1 sending a `pageId` from workspace 2.
- Rejection: in [`packages/services/src/status-report/create.ts:32-40`](../../packages/services/src/status-report/create.ts#L32-L40) — the page lookup uses `eq(page.workspaceId, ctx.workspace.id)`, so a cross-workspace page returns undefined, the service throws `NotFoundError`, and the adapter maps to `NOT_FOUND` (404, not 403, to avoid revealing existence).

## Kata 9 — The dashboard mutation

**Read:** [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-52`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L33-L52).

**Question:** in `onSuccess`, what side effects and cache updates happen, and in what order? What's the reliability risk?

**Rubric:**
- Order: (optionally) await sendStatusReportUpdateMutation (only if notifySubscribers); then refetch() this query; then invalidate page.list queryKey.
- Risk: notification reliability is in the browser path. If the browser closes, network drops, or the email mutation fails, the create succeeds but the notification doesn't go. TODO comment acknowledges this.

## Kata 10 — The custom tRPC link

**Read:** [`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77).

**Question:** describe the routing logic. Identify a fragility.

**Rubric:**
- Routing: split procedure path by `.`; if first segment is in `lambdas` array, route to lambda HTTP endpoint, else edge.
- Fragility: hardcoded array. Adding a new lambda router means *two* edits (the lambda router registration in `packages/api/src/lambda.ts` *and* this array). Forget one and the client routes to a non-existent endpoint.

## Self-prompt for AI help

If you must ask an AI, use this prompt — it makes the AI a coach, not a vending machine:

```
I'm working on Kata N from docs/06-practice-tracks/01-reading-katas.md.
I read [file:lines]. My current summary is: [your summary].
Don't tell me the answer. Ask me 3-5 questions that would help me see what I'm missing.
```
