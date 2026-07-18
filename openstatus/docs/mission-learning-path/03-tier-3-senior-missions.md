# Tier 3: Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions
**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Infer why the system is split into apps, API, services, and DB packages.
**The Concept:** Architecture is an incident postmortem before the incident: every boundary prevents a future failure.
**Design Intent Before You Read the Code:** Routers should handle transport, services should handle business rules, DB package should own persistence.
**Find It In The Code:** Open `packages/api/src/router/statusReport.ts:21-23`, `packages/api/src/service-adapter.ts:13-24`, `packages/services/src/context.ts:22-47`, `packages/services/package.json:8-81`.

```ts
// tRPC-side input schemas. These preserve the existing wire contract exactly;
// each procedure adapts between this shape and the canonical service schema.
// Decision: avoid breaking UI callers while extracting reusable services.
```

**The Aha Moment:** The service layer exists so business rules survive beyond one API transport.
**Socratic Checkpoint:** 5 questions.
1. What does the router preserve?
2. What does the service own?
3. Why does `ServiceContext` include actor and workspace?
4. Why export services by domain?
5. What future API surface benefits from this split?

How to self-grade: Strong answers cite all mission line ranges and name transport independence as the core decision.
**Connects To:** Mission 19 and Mission 23.

### Mission 19: Find the Bugs Before They Happen
**Tier:** Senior
**Time Estimate:** 55 minutes
**Goal:** Predict likely bugs from ownership and boundary gaps.
**The Concept:** Senior debugging starts before the outage: inspect where promises cross boundaries.
**Design Intent Before You Read the Code:** Risk hides where one layer performs another layer's job.
**Find It In The Code:** Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-55`, `packages/services/src/auth/require-scope.ts:11-30`, `packages/api/src/router/statusPage.ts:364-379`.

```tsx
if (!statusReports || !page) return null;
// Missing loading/error UI can turn backend trouble into a blank user experience.
```

```ts
if (actor.type !== "apiKey" && actor.type !== "mcp") {
  return;
}
// User actors skip scope enforcement; do not assume member-role checks exist here.
```

**The Aha Moment:** The riskiest code often looks reasonable until you ask who owns the failure.
**Socratic Checkpoint:** 5 questions.
1. Which blank-state bug can users see?
2. Which auth gap is explicitly documented?
3. Which sort code deserves a test?
4. Which client-side side effect can partially fail?
5. What would you log or monitor?

How to self-grade: Strong answers cite the three mission ranges and propose one concrete test for each risk.
**Connects To:** Mission 20 and Mission 22.

### Mission 20: The Bug Injection Challenge
**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Design bug scenarios from user-visible symptoms.
**The Concept:** Injecting bugs mentally trains you to recognize real incidents faster.
**Design Intent Before You Read the Code:** Do not edit production code. Write scenarios and tests.
**Find It In The Code:** Open `packages/api/src/router/statusReport.test.ts:77-207`, `packages/services/src/status-report/__tests__/status-report.test.ts:116-208`.

```ts
await expect(
  createStatusReport({
    ctx: { ...freeCtx, db: tx },
    input: { pageId: testPageId, pageComponentIds: [] },
  }),
).rejects.toBeInstanceOf(NotFoundError);
// Test describes the user-visible security expectation: other workspaces look missing.
```

**The Aha Moment:** A good bug scenario names symptom, setup, action, and expected guardrail.
**Socratic Checkpoint:** 5 questions.
1. What symptom proves cross-workspace leakage?
2. What setup creates another workspace page?
3. What API should reject it?
4. Why is `NOT_FOUND` safer than `FORBIDDEN` here?
5. Which service test pattern would you copy?

How to self-grade: Strong answers cite `packages/api/src/router/statusReport.test.ts:77-207` and `packages/services/src/status-report/__tests__/status-report.test.ts:165-180`.
**Connects To:** Mission 23 and Mission 24.

### Mission 21: Performance X-Ray
**Tier:** Senior
**Time Estimate:** 55 minutes
**Goal:** Identify hot paths and avoid accidental N+1 behavior.
**The Concept:** Performance tracing is uptime monitoring for code paths.
**Design Intent Before You Read the Code:** A list endpoint should not issue one query per row.
**Find It In The Code:** Open `packages/services/src/status-report/list.ts:66-161`, `packages/api/src/router/statusPage.ts:49-55`, `packages/api/src/router/statusPage.ts:80-113`.

```ts
const allUpdates = await db
  .select()
  .from(statusReportUpdate)
  .where(inArray(statusReportUpdate.statusReportId, reportIds))
  .orderBy(desc(statusReportUpdate.date))
  .all();
// One batched query for all updates, not one query per report.
```

**The Aha Moment:** Batched enrichment is a deliberate performance architecture.
**Socratic Checkpoint:** 5 questions.
1. Which three relation groups are batched?
2. Why does dashboard list pressure matter?
3. Which public query admits performance work is needed?
4. Where might caching help?
5. What metric would you capture?

How to self-grade: Strong answers cite `packages/services/src/status-report/list.ts:66-161` and `packages/api/src/router/statusPage.ts:49-113`.
**Connects To:** Mission 22 and Mission 23.

### Mission 22: The Security Audit
**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Audit auth, workspace isolation, and untrusted URL patterns.
**The Concept:** A status platform must protect tenant boundaries the way it protects incident trust.
**Design Intent Before You Read the Code:** Security should be enforced server-side and tested.
**Find It In The Code:** Open `apps/dashboard/src/lib/trpc/shared.ts:17-29`, `packages/api/src/trpc.ts:148-200`, `packages/services/src/auth/require-scope.ts:25-48`, `packages/utils/src/ssrf.test.ts:26-160`.

```ts
export function guardTRPCSource(req: Request): Response | null {
  const source = req.headers.get("x-trpc-source");
  if (source !== "server" && source !== "client") {
    return new Response(null, { status: 401 });
  }
  return null;
}
// Comment says this is spoofable; real auth must be elsewhere.
```

**The Aha Moment:** Security is layered; a convention guard is not an authorization model.
**Socratic Checkpoint:** 5 questions.
1. What is spoofable?
2. Where is session required?
3. Where is workspace resolved?
4. Which actors are scope-checked?
5. What URLs do SSRF tests block?

How to self-grade: Strong answers cite all mission ranges and separate authentication, authorization, tenant isolation, and SSRF validation.
**Connects To:** Mission 23 and Mission 24.

### Mission 23: Write the Test That Doesn't Exist
**Tier:** Senior
**Time Estimate:** 70 minutes
**Goal:** Design a missing test for risky behavior.
**The Concept:** A missing test is an unmonitored service: you may not notice it is down until users do.
**Design Intent Before You Read the Code:** Tests should pin behavior at the lowest layer that owns the rule.
**Find It In The Code:** Open `packages/api/src/router/statusPage.ts:364-379`, `packages/services/src/status-report/__tests__/status-report.test.ts:116-208`, `packages/api/src/router/statusReport.test.ts:77-207`.

```ts
const statusReports = _page.statusReports.sort((a, b) => {
  if (a.statusReportUpdates.length === 0) return -1;
  if (b.statusReportUpdates.length === 0) return -1;
  return bLastUpdate - aLastUpdate;
});
// Suspicious comparator: empty b also returns -1, so write a sort-order test.
```

**The Aha Moment:** Test the behavior users rely on, not just the line you distrust.
**Socratic Checkpoint:** 5 questions.
1. What behavior needs pinning?
2. Which layer owns it?
3. What fixtures are needed?
4. What order should the assertion expect?
5. What would fail if the comparator is wrong?

How to self-grade: Strong answers propose a runnable Bun test and cite `packages/api/src/router/statusPage.ts:364-379`.
**Connects To:** Mission 24.

### Mission 24: The Git History Tells a Story
**Tier:** Senior
**Time Estimate:** 45 minutes
**Goal:** Practice inferring evolution from commit messages and current code shape.
**The Concept:** Git history is a status feed for architecture decisions.
**Design Intent Before You Read the Code:** Use present code to infer why past changes happened.
**Find It In The Code:** Open `packages/api/src/router/statusReport.ts:21-23`, `packages/services/src/status-report/list.ts:66-161`, `apps/dashboard/src/lib/trpc/shared.ts:38-76`, `packages/api/src/router/statusReport.test.ts:77-207`.

```ts
// Legacy wire contract in router + canonical schema in service
// implies a migration from router-owned business logic to service-owned logic.
```

**The Aha Moment:** Architecture history is visible in comments, adapters, tests, and compatibility code.
**Socratic Checkpoint:** 5 questions.
1. What migration does the status-report router comment imply?
2. What performance problem does batched enrichment imply?
3. Why might edge/lambda split have been introduced?
4. What security event could have motivated IDOR tests?
5. What commit message would you write for adding role enforcement?

How to self-grade: Strong answers cite all mission ranges and connect code comments to architectural forces.
**Connects To:** Reference artifacts, because ownership means writing durable operational guidance.

