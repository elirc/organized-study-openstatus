# User Story Build Path

This suite gives you ten real feature stories to build inside OpenStatus. The stories are grounded in the status-page/status-report workflow because that workflow uses the dashboard route, form components, typed tRPC, service layer, Drizzle schema, tests, and public rendering (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`; `apps/dashboard/src/components/forms/status-report/form.tsx:79-417`; `packages/api/src/router/statusReport.ts:58-222`; `packages/services/src/status-report/create.ts:24-99`; `packages/db/src/schema/status_reports/status_reports.ts:15-89`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

Difficulty progression:

- Stories 1-3 are Easy. You change existing UI or fields in 1-2 files.
- Stories 4-5 are Medium-Easy. You add UI that reads existing data in 3-4 files.
- Stories 6-7 are Medium. You add an API or state path in 4-6 files.
- Stories 8-9 are Hard. You touch frontend, backend, and database-facing behavior in 6-10 files.
- Story 10 is Expert. You design an architecture-level change around reliability/performance/authorization.

A story is done when every acceptance criterion is independently verified, the UI has sensible loading/error states, the relevant typed contract is updated, and a reviewer can trace the change from UI to service or public rendering using file paths.

How to get unstuck without outsourcing the learning:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z.
Don't give me the solution. Ask me questions that help me figure it out.
Use the OpenStatus files I mention as context and help me trace the next boundary.
```

Use AI as a code-reading coach, not a code vending machine. If you ask for direct code too early, you skip the exact mental reps these stories are designed to build.

