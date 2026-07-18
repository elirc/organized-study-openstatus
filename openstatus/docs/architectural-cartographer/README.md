# Architectural Cartographer

This suite is a top-down onboarding guide for OpenStatus. Use it when you want to understand how the system is shaped before changing it. The repo is a pnpm monorepo because the root package declares pnpm as the package manager and workspaces under `apps/*`, `packages/**/*`, and `packages/config/*` (`package.json:33-38`; `pnpm-workspace.yaml:1-4`). The primary teaching path follows status pages and status reports because that feature crosses dashboard UI, tRPC, service functions, Drizzle tables, and public status-page rendering (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`; `packages/api/src/router/statusReport.ts:58-222`; `packages/services/src/status-report/create.ts:24-99`; `packages/db/src/schema/status_reports/status_reports.ts:15-89`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

Recommended order:

1. Read `00-reading-map.md` to build the first mental model.
2. Read `01-junior-engineer.md` while opening the referenced files beside it.
3. Read `02-mid-level-engineer.md` when you can already explain the status-report create path.
4. Read `03-senior-engineer.md` when you are ready to critique risks, not just follow flows.
5. Use `04-reference-suite.md` as the permanent handbook for reviews, debugging, and explaining the system.

For checkpoints, answer out loud first, then compare yourself against the "How to self-grade" section. The goal is not memorization. The goal is to practice the same questions a senior engineer asks while reading unfamiliar product code.

