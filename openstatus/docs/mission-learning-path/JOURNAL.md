# Journal

## Mission Design Rationale

The missions begin with repo heartbeat and folder mapping because root scripts and workspaces explain how the system is operated (`package.json:3-20`; `package.json:33-38`; `turbo.json:4-66`). They then move into a single feature: status reports. That path was chosen because it begins in a user-facing dashboard page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`), crosses typed tRPC (`apps/dashboard/src/lib/trpc/client.tsx:11-64`; `packages/api/src/router/statusReport.ts:58-222`), lands in service transactions (`packages/services/src/status-report/create.ts:24-99`), persists through Drizzle tables (`packages/db/src/schema/status_reports/status_reports.ts:15-89`), and returns to public rendering (`packages/api/src/router/statusPage.ts:65-400`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:136-414`).

## Why Missions Are Ordered This Way

Tier 1 develops orientation: scripts, folders, TypeScript contracts, one component, one route, props, data entry, and navigation (`README.md:93-143`; `apps/dashboard/src/lib/trpc/client.tsx:39-64`; `apps/dashboard/src/app/(dashboard)/status-pages/[id]/layout.tsx:18-50`).

Tier 2 develops tracing: remote state, custom hooks, side effects, API contracts, middleware, end-to-end behavior, diff reading, composition, and inferred types (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-51`; `packages/ui/src/hooks/use-mobile.ts:7-23`; `packages/api/src/trpc.ts:148-264`).

Tier 3 develops ownership: architecture decisions, bug prediction, performance, security, missing tests, and history reasoning (`packages/services/src/auth/require-scope.ts:7-48`; `packages/api/src/router/statusPage.ts:49-55`; `packages/api/src/router/statusReport.test.ts:77-207`).

## What A Senior Engineer Does Differently

A junior often asks "what does this line do?" A senior asks "what contract does this line protect, and what breaks if the contract changes?" For example, `statusReport.create` preserves a legacy return shape for dashboard consumers (`packages/api/src/router/statusReport.ts:75-77`), while `createStatusReport` owns DB integrity and audit emission (`packages/services/src/status-report/create.ts:54-95`).

At checkpoints, a senior cites evidence, names risk, and identifies the next test they would write.

