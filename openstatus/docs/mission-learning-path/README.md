# Mission Learning Path

This suite is a training campaign. You are not passively reading; you are practicing how to trace, annotate, debug, review, and explain OpenStatus.

The missions use the status-page/status-report path because it touches the main skills you need: Next App Router UI, typed React Query/tRPC calls, runtime validation, protected API context, service transactions, Drizzle schema, public rendering, and tests (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`; `packages/api/src/router/statusReport.ts:58-222`; `packages/services/src/status-report/create.ts:24-99`; `packages/db/src/schema/status_reports/status_reports.ts:15-89`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

Pacing:

- Tier 1: 1-2 missions per day. Your goal is orientation and vocabulary.
- Tier 2: 1 mission per day. Your goal is tracing behavior across boundaries.
- Tier 3: 1 mission every 1-2 days. Your goal is critique, risk detection, and ownership thinking.

Self-grade after every checkpoint. A strong answer includes file paths, line ranges, the boundary being crossed, and the failure mode if the code is changed poorly.

This suite differs from normal documentation because every section asks you to do something: open a file, annotate a block, answer questions, or simulate a review.

