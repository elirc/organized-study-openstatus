# 03 — Exercises

## Runtimes

1. Open [`packages/db/src/db.ts`](../../packages/db/src/db.ts). Explain why `drizzle-orm/libsql/http` is used. Try changing the import to `drizzle-orm/libsql` and run `pnpm build` of the dashboard. What error?
2. Add a `process.env.MY_SECRET` read to a `"use client"` file. What does Next.js do? Now try `NEXT_PUBLIC_MY_SECRET`. Trace it through the bundle.
3. List every router on the lambda side and explain why each can't be on edge. (Hint: imports Stripe? imports Resend?)
4. Pretend you're adding a Slack notification feature. Decide: edge or lambda? Defend your choice in three sentences.

## HTTP and tRPC plumbing

5. Open both `apps/dashboard` and `apps/status-page`'s tRPC route handlers (e.g., `apps/.../api/trpc/edge/[trpc]/route.ts`). What's different? Why two?
6. Walk through `protectedProcedure` line by line ([`packages/api/src/trpc.ts:148-241`](../../packages/api/src/trpc.ts#L148-L241)). For each control-flow branch, name a real scenario that hits it.
7. Add a tRPC middleware that logs procedure path + duration. Mount it once. Confirm in the terminal.
8. Read the "test escape hatch" in `enforceUserIsAuthed`. Write a sentence explaining the specific test failure it prevents.

## Drizzle

9. Pick the largest list query in the repo (likely `statusPage.get` in [`packages/api/src/router/statusPage.ts`](../../packages/api/src/router/statusPage.ts)). Count the DB round-trips. Identify which are parallel (`Promise.all`) and which are serial.
10. Compare the relational style (`db.query.x.findMany({ with: {...} })`) and builder style (`db.select().from(x).leftJoin(...)`) for the same query. Which is more readable for *this* query?
11. Find an `eq(table.workspaceId, ctx.workspace.id)` filter. Remove it locally. Run the relevant test. Document the result. Revert.
12. Identify a table without an index that you'd expect to be queried by a foreign key. Add the index in a new migration. Inspect the generated SQL.
13. Wrap a multi-write service verb's outermost call in two `withTransaction` blocks (deliberately). What happens? Why?

## Auth and multi-tenancy

14. Read [`packages/api/src/router/statusReport.test.ts:77-207`](../../packages/api/src/router/statusReport.test.ts#L77-L207). For each test, name the attack it prevents.
15. Find every write service. Verify each one has `requireScope(ctx, "write")` as line 1. Note exceptions.
16. Construct a malicious request: an authenticated user trying to create a status report on another workspace's page. Trace the rejection through router → service → DB. Where does it stop?
17. Read [`packages/services/src/auth/require-scope.ts`](../../packages/services/src/auth/require-scope.ts). For each actor type, list its scope behavior. Identify one gap.
18. Find a `notify` or `dispatch` service (e.g., for notifications). Decide if it counts as a "write" — does it have `requireScope(ctx, "write")`?

## Synthesis

19. Pick a new feature: "auto-resolve a status report 24 hours after the last update." Sketch the layers: where in the DB? where in services? router? UI? Background job? Where would auth enforcement live?
20. Find one place in this repo where a senior would say "this should be on Lambda, not Edge" or vice versa. Defend or refute the current placement.

## Self-grading

You're senior-shaped on Node + DB + auth when:
- You can name the runtime of any file in 5 seconds.
- You can spot an N+1 in a code review at a glance.
- You can defend or attack any `WHERE workspace_id` you read.
- You know which middleware runs in what order on any procedure.
- You can sketch the transaction boundary for any multi-row write.
