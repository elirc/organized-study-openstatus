# Code Reading Gym

This suite is for hands-on learners. It trains you to read real code through drills, trace tables, comparison exercises, and review katas.

The goal is not to memorize OpenStatus. The goal is to build the reflexes you need when you open an unfamiliar file: locate the owner, name the inputs, follow the output, identify side effects, and ask what breaks.

Primary anchors:

- Dashboard status report page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`)
- Status report form (`apps/dashboard/src/components/forms/status-report/form.tsx:62-417`)
- tRPC status report router (`packages/api/src/router/statusReport.ts:24-222`)
- Service create/list functions (`packages/services/src/status-report/create.ts:24-99`; `packages/services/src/status-report/list.ts:66-226`)
- DB schema (`packages/db/src/schema/status_reports/status_reports.ts:8-89`; `packages/db/src/schema/page_components/page_components.ts:124-157`)

Fake code examples in this suite are intentionally small and marked as illustrative. Use them to build the mental model, then return to the real code.

