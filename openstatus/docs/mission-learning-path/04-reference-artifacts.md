# Reference Artifacts

### Mission 25: Write the Docs That Don't Exist

Use these action references while doing real work.

## Doc 1: Junior Onboarding Checklist

- Install Node 20+, pnpm, Bun, and Turso CLI because the README lists those requirements (`README.md:93-101`).
- Run `pnpm install` (`README.md:110-114`).
- Start DB with `turso dev --db-file openstatus-dev.db` (`README.md:116-122`).
- Run `pnpm dx`; Turbo wires `dx` to `seed`, `migrate`, and `env` (`README.md:124-128`; `turbo.json:51-64`).
- Start a surface with `pnpm dev:dashboard`, `pnpm dev:status-page`, or `pnpm dev:web` (`package.json:15-17`).
- First PR checklist: identify the route, API procedure, service function, DB schema, UI cache update, and test touched by your change.

## Doc 2: Architecture Guide for New Engineers

Navigate by ownership:

- UI routes: `apps/dashboard/src/app` and `apps/status-page/src/app` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).
- API: `packages/api/src/router` plus context in `packages/api/src/trpc.ts` (`packages/api/src/trpc.ts:59-119`; `packages/api/src/edge.ts:25-49`).
- Services: `packages/services/src/<domain>` (`packages/services/package.json:8-81`).
- Persistence: `packages/db/src/schema` (`packages/db/src/schema/index.ts:1-27`).

## Doc 3: Code Review Checklist

- Does the route use `protectedProcedure` when user/workspace data is modified? (`packages/api/src/trpc.ts:255-264`)
- Does the service check workspace ownership before writes? (`packages/services/src/status-report/create.ts:32-46`)
- Does the mutation map errors with `toTRPCError`? (`packages/api/src/service-adapter.ts:32-87`)
- Does the UI update/invalidate the correct React Query cache? (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:46-49`)
- Is there a test for cross-workspace rejection? (`packages/api/src/router/statusReport.test.ts:77-207`)

## Doc 4: Debugging Playbook

For failed create:

1. Check form branch and payload (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`).
2. Check tRPC input schema (`packages/api/src/router/statusReport.ts:24-32`).
3. Check workspace middleware (`packages/api/src/trpc.ts:148-200`).
4. Check service page lookup and component validation (`packages/services/src/status-report/create.ts:32-52`).
5. Check DB tables (`packages/db/src/schema/status_reports/status_reports.ts:15-63`; `packages/db/src/schema/page_components/page_components.ts:124-157`).

## Doc 5: Change Playbook

Branch, implement, test, review, merge:

1. Make a branch named after the user-visible change.
2. Start at UI route or service depending on whether this is presentation or behavior.
3. Update service schema before tRPC if the domain contract changes (`packages/services/src/status-report/schemas.ts:12-92`).
4. Keep tRPC backward-compatible where comments say legacy consumers exist (`packages/api/src/router/statusReport.ts:21-23`).
5. Add service/router tests around workspace boundaries (`packages/services/src/status-report/__tests__/status-report.test.ts:116-208`; `packages/api/src/router/statusReport.test.ts:77-207`).
6. Run targeted package tests, then broader `pnpm test` if practical (`package.json:20`).

## Doc 6: Senior Ownership Notes

Monitor these:

- Public status-page latency around `statusPage.get` (`packages/api/src/router/statusPage.ts:49-113`).
- Create-report notification failures caused by UI-owned email side effect (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
- Member-role enforcement gap for dashboard user actors (`packages/services/src/auth/require-scope.ts:11-30`).
- Any regression in batched report enrichment (`packages/services/src/status-report/list.ts:66-161`).

## Doc 7: Interview Walkthrough

Practice answer:

"OpenStatus is a pnpm/Turbo monorepo for status pages and uptime monitoring (`package.json:3-20`; `README.md:144-153`). The dashboard uses typed tRPC clients generated from `AppRouter` (`packages/api/src/root.ts:1-6`; `apps/dashboard/src/lib/trpc/client.tsx:9-12`). A status-report create starts in the dashboard page, goes through tRPC, enters a service transaction, writes Drizzle rows, and appears on the public page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`; `packages/api/src/router/statusReport.ts:58-81`; `packages/services/src/status-report/create.ts:32-98`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`)."

