# Debugging Forensics Lab

This suite is for problem-first learners. Instead of starting with architecture, it starts with symptoms: blank screens, missing notifications, wrong public status, failed writes, and suspicious performance. You will learn the codebase by investigating what could have gone wrong.

Primary investigation path:

```text
Symptom -> reproduce -> locate UI owner -> identify query/mutation -> inspect tRPC contract -> inspect service rule -> inspect DB shape -> inspect public rendering or cache refresh
```

Core evidence:

- Dashboard status report page owns report list, create mutation, notification side effect, and cache refresh (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`).
- tRPC status-report router owns runtime input validation, protected procedures, service adaptation, and error mapping (`packages/api/src/router/statusReport.ts:24-222`; `packages/api/src/service-adapter.ts:32-87`).
- Service create owns workspace validation, transaction writes, associations, and audit rows (`packages/services/src/status-report/create.ts:24-99`).
- Public status-page query computes user-visible status, open events, last events, trackers, reports, and maintenances (`packages/api/src/router/statusPage.ts:65-400`).
- Public client renders banner, component cards, and feed from `statusPage.get` output (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

Use this suite when the question starts with "Why did the app do that?"

