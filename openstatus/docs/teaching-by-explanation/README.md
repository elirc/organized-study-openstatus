# Teaching By Explanation

This suite is for verbal, social, and teach-back learners. It gives you scripts, Socratic dialogues, flashcards, and whiteboard prompts so you can explain OpenStatus under pressure.

Use this suite when you want to practice saying the system out loud. The best way to know whether you understand a codebase is to explain a feature from user intent to database rows and back to user-visible output.

Primary explanation path:

```text
Dashboard creates status report -> tRPC validates/adapts -> service writes transaction -> DB stores rows -> public status page computes and renders event/feed data
```

Evidence for that path lives in the dashboard page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`), status-report router (`packages/api/src/router/statusReport.ts:58-222`), service create function (`packages/services/src/status-report/create.ts:24-99`), DB tables (`packages/db/src/schema/status_reports/status_reports.ts:15-89`; `packages/db/src/schema/page_components/page_components.ts:124-157`), and public page renderer (`packages/api/src/router/statusPage.ts:65-400`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

