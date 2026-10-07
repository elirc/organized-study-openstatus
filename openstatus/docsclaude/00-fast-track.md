# 00 — Fast Track (Weekend Version)

If you have one weekend, do this. You won't be senior afterwards, but you'll have a working senior-shaped mental model of OpenStatus and the JS/TS/React/Node stack it uses.

## Saturday morning (2h) — Get the system running

1. Read [`README.md`](README.md) — this guide's index.
2. Run the manual setup from the repo root [`README.md`](README.md):
   ```sh
   pnpm install
   turso dev --db-file openstatus-dev.db   # separate terminal
   pnpm dx                                 # env -> migrate -> seed
   pnpm dev:dashboard
   ```
3. Open `http://localhost:3002` (or whatever port `pnpm dev:dashboard` prints). Click around the dashboard until you understand: workspaces, status pages, monitors, status reports, notifications, subscribers.
4. Open `http://localhost:3003` for a public status page (use a seeded slug).

If anything breaks, do not move on. Reading code for a system you can't run is half-blind. The fastest way to unblock is `pnpm dx` failures — almost always `DATABASE_URL` or a missing `.env` from `packages/db/.env.example`.

## Saturday afternoon (3h) — Trace one feature end-to-end

The status report create flow is the best one to trace because it touches everything: route handler, tRPC, service layer, transaction, audit log, query invalidation, public read model.

Open these files **in this order** with the comments in [`02-react-deep-dive/06-react-query-and-trpc.md`](02-react-deep-dive/06-react-query-and-trpc.md) and [`04-architecture-and-patterns/01-layering-and-services.md`](04-architecture-and-patterns/01-layering-and-services.md) handy:

1. [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx) — UI submits, then invalidates caches on success.
2. [`apps/dashboard/src/components/forms/status-report/form.tsx`](../apps/dashboard/src/components/forms/status-report/form.tsx) — react-hook-form + Zod resolver.
3. [`apps/dashboard/src/lib/trpc/client.tsx`](../apps/dashboard/src/lib/trpc/client.tsx) and [`apps/dashboard/src/lib/trpc/shared.ts`](../apps/dashboard/src/lib/trpc/shared.ts) — typed client + edge/lambda link routing.
4. [`packages/api/src/router/statusReport.ts`](../packages/api/src/router/statusReport.ts) — protected procedure, input validation, service adapter.
5. [`packages/api/src/service-adapter.ts`](../packages/api/src/service-adapter.ts) — `toServiceCtx` and `toTRPCError` are the boundary.
6. [`packages/services/src/status-report/create.ts`](../packages/services/src/status-report/create.ts) — `requireScope`, workspace check, transaction, audit emit.
7. [`packages/services/src/context.ts`](../packages/services/src/context.ts) — `withTransaction` and the `Actor` discriminated union.
8. [`packages/db/src/schema/status_reports/status_reports.ts`](../packages/db/src/schema/status_reports/status_reports.ts) — Drizzle schema.

Now write a one-page explanation in your own words: what happens between "user clicks Create" and "DB row exists." Don't skip this. Writing forces precision.

## Saturday evening (2h) — Read the language modules

Skim [`01-javascript-typescript-core/`](01-javascript-typescript-core/README.md). Don't try to absorb everything — read the *pitfalls* sections in each file. Those are where your existing knowledge is most likely wrong.

## Sunday morning (3h) — Make one real change

Pick one from [`06-practice-tracks/02-modify-tickets.md`](06-practice-tracks/02-modify-tickets.md). The "Add a column to the reports table" ticket is the most rewarding for a weekend: it touches a service file, a router (sometimes), the data table column definition, and tests.

The pattern you should follow on every change:
1. Read the entire path you're about to modify before writing anything.
2. Make the change.
3. Run `pnpm test` (or just the touched package's test).
4. Run `pnpm lint` if biome is configured.
5. Use the feature in the browser — type checks don't catch UX regressions.
6. Write a 3-bullet PR description: what, why, what you tested.

## Sunday afternoon (2h) — Read with a critical eye

Read [`CRITIQUE-codex-vs-claude.md`](../CRITIQUE-codex-vs-claude.md) (moved to the repo root in this snapshot). Then re-open one of your favorite files from earlier and ask: *what would a senior change about this?* If you can find one real critique with a concrete fix, the weekend was worth it.

## What this doesn't cover

A weekend can't teach you: the React render model from first principles, advanced TypeScript types, profiling, security review reflexes, or how to design a system. The full curriculum exists for that. The fast track exists so you don't freeze at the door.
