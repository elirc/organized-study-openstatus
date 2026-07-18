# Socratic Dialogues

These are scripted practice conversations. Cover the answer first, then try to answer as the learner.

## Dialogue 1: "Where Does This Data Come From?"

Mentor: The status reports page renders a table. Where does its data come from?

Learner: From `useQuery`.

Mentor: Which query exactly?

Strong answer: `trpc.statusReport.list.queryOptions({ pageId: Number.parseInt(id) })` in the dashboard status-reports page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-32`).

Mentor: Where did `id` come from?

Strong answer: `useParams<{ id: string }>()` reads the dynamic Next route segment, then the code parses it into a number for API calls (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-32`).

Mentor: What does the backend do with `pageId`?

Strong answer: the tRPC router passes `pageId` to `listStatusReports`, which filters by `statusReport.pageId` when provided (`packages/api/src/router/statusReport.ts:189-212`; `packages/services/src/status-report/list.ts:172-181`).

## Dialogue 2: "Why Is There A Service Layer?"

Mentor: Why not put all create logic inside the tRPC router?

Learner: To keep it cleaner.

Mentor: Cleaner how?

Strong answer: the router preserves the dashboard wire contract and adapts it to the service schema (`packages/api/src/router/statusReport.ts:21-23`; `packages/api/src/router/statusReport.ts:64-74`). The service owns business rules like scope check, workspace page lookup, component validation, transaction writes, and audit emission (`packages/services/src/status-report/create.ts:24-99`).

Mentor: What future benefit does that create?

Strong answer: another transport or background job can reuse `createStatusReport` without copying tRPC-specific field names or request handling. The services package exports domain modules for reuse (`packages/services/package.json:8-81`).

## Dialogue 3: "What Is The Actual Security Boundary?"

Mentor: Is `x-trpc-source` a security boundary?

Learner: It blocks bad requests.

Mentor: Read the comment. What does it say?

Strong answer: it says the header is a convention filter for bots/crawlers, not a security boundary, because the header is trivially spoofable (`apps/dashboard/src/lib/trpc/shared.ts:17-29`).

Mentor: Then what is the security boundary?

Strong answer: `protectedProcedure` verifies session user id, resolves the active workspace, and passes user/workspace into context (`packages/api/src/trpc.ts:148-200`). Services still enforce resource ownership, such as checking page id and workspace id together (`packages/services/src/status-report/create.ts:32-40`).

## Dialogue 4: "What Happens After The DB Write?"

Mentor: After `createStatusReport` returns, what does the UI do?

Strong answer: if the returned object has `notifySubscribers`, it calls `sendStatusReportUpdateMutation`, then refetches the report list and invalidates `trpc.page.list` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).

Mentor: Why should a senior engineer pause there?

Strong answer: because notification is a side effect owned by the client path today. If notification fails, the report was already created. The code even says `TODO: move to server` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:38-44`).

## Dialogue 5: "How Does Public Status Get Computed?"

Mentor: Is public page status stored directly as one DB column?

Strong answer: not in the public path shown here. The router loads page relations, computes component statuses, derives page-wide events, open events, last events, trackers, and then returns a parsed public object (`packages/api/src/router/statusPage.ts:80-400`).

Mentor: Why does that matter when adding draft reports or severity?

Strong answer: because filtering or ranking only in the React client is too late. Public computed fields like `openEvents`, `lastEvents`, and `status` are built in the router first (`packages/api/src/router/statusPage.ts:222-400`).

