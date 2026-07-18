# Junior Feature Ticket Book

This folder is a junior-friendly backlog of 25 additional feature stories for this repo. It is written like a tech lead handing you real tickets: each story tells you what to build, where to look first, how to make the change in small steps, what fake code shape to aim for, and how to know whether you are done.

Use this folder when you want to practice implementation, not just reading. The earlier story suite in `docs/user-story-build-path/01-stories.md` focuses heavily on the status-report flow. This suite widens the training field across dashboard pages, page components, subscribers, notifications, monitors, public status pages, maintenance windows, and API/service boundaries.

## Recommended Order

1. `01-easy-ui-stories.md` - copy, empty states, visible context, and tiny derived UI.
2. `02-dashboard-state-stories.md` - client state, filters, URL state, and existing data.
3. `03-api-and-service-stories.md` - router inputs, service calls, validation, and tests.
4. `04-public-page-and-notification-stories.md` - public UX, subscriber management, notification safety.
5. `05-full-stack-growth-stories.md` - deeper features with database, API, UI, and migration thinking.

## How To Work A Story

1. Read the story once without editing.
2. Open every file listed in "Read These Anchors First."
3. Write down the current data flow in your own words.
4. Follow the implementation plan step by step.
5. Use the fake-code sketch as a shape guide, not something to paste blindly.
6. Run the suggested checks.
7. Write a short PR summary with: what changed, why it is safe, and what you tested.

## What "Done" Means

A story is done when the user-visible behavior works, TypeScript understands the change, the query or mutation still uses the repo's existing tRPC patterns, and the relevant tests or manual checks pass.

For this repo, a junior dev should especially watch:

- Dashboard queries usually come from `useTRPC()` plus React Query, as shown in `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:37-49` and `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:43-49`.
- Protected dashboard router procedures adapt to service functions through `toServiceCtx(ctx)`, as shown in `packages/api/src/router/page.ts:92-105` and `packages/api/src/router/monitor.ts:131-148`.
- Public status-page data is rendered from `statusPage.get`, with client-side filtering and mapping in `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-136` and `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`.
- Subscriber flows have both public procedures and protected dashboard procedures in the same router, so do not mix the permission model. See `packages/api/src/router/pageSubscriber.ts:52-98` for public self-signup and `packages/api/src/router/pageSubscriber.ts:230-357` for dashboard-owned operations.

## How To Ask For Help Without Losing The Learning

Use this prompt with Claude, ChatGPT, or a teammate:

```text
I'm working on Story [number] from docs/junior-feature-ticket-book.
The behavior I need is: [acceptance criterion].
The files I opened are: [paths].
The step I'm stuck on is: [specific step].
Here is what I tried: [short summary].
Do not give me the solution. Ask me 3-5 debugging or design questions that help me discover the next move.
```

## Ticket Difficulty

- Easy stories touch mostly one UI area and use data that is already present.
- Medium stories introduce client state, derived values, filter state, or a small router input.
- Hard stories cross UI, API, service, tests, and sometimes database schema.
- Expert stories ask you to make an architectural choice before editing.

