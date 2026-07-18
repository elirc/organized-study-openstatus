# Router Test Workouts

## Workout 1: tRPC Create Rejects Other Workspace Page

Existing test:

```ts
// packages/api/src/router/statusReport.test.ts:77-99
test("statusReport.create rejects pageId from another workspace", async () => {
  const ctx = createInnerTRPCContext({
    req: undefined,
    session: { user: { id: "1" } },
    workspace: { id: 1 },
  });
  const caller = edgeRouter.createCaller(ctx);
  // caller.statusReport.create(...) rejects with NOT_FOUND
});
```

Why router test: it confirms the public API surface maps service rejection to `TRPCError` with code `NOT_FOUND` (`packages/api/src/router/statusReport.test.ts:96-99`; `packages/api/src/service-adapter.ts:41-45`).

Practice extension: assert the error message if stable enough, but avoid overfitting if the message is intentionally generic.

## Workout 2: tRPC Create Succeeds For Own Workspace

Existing test: `statusReport.create succeeds for own workspace page` calls the router, expects a result, then cleans up the inserted row (`packages/api/src/router/statusReport.test.ts:102-124`).

Learning target: router tests can be higher-level than service tests. They do not need to reassert every association if service tests already cover that. They should prove the API contract works.

Illustrative fake test:

```ts
// Illustrative fake code.
const result = await caller.statusReport.create(validPayload);
expect(result).toBeDefined();
expect(result.notifySubscribers).toBeUndefined(); // if caller did not send it
```

Check the actual router create return shape merges `notifySubscribers` into the initial update (`packages/api/src/router/statusReport.ts:75-77`).

## Workout 3: Component Injection Rejection

Existing router tests reject page components from another workspace during create and update (`packages/api/src/router/statusReport.test.ts:181-207`; `packages/api/src/router/statusReport.test.ts:210-220`).

Why it matters: the dangerous bug is not only "wrong page id." A user might choose a valid page id in their workspace but inject an affected component id from another workspace. The service validates component ids after page ownership lookup (`packages/services/src/status-report/create.ts:42-52`).

Illustrative fake payload:

```ts
// Illustrative fake code.
await caller.statusReport.create({
  pageId: ownPageId,
  pageComponents: [componentIdFromOtherWorkspace],
  title: "Cross-workspace injection",
  status: "investigating",
  date: new Date(),
  message: "Should fail",
});
```

## Workout 4: Router Contract For List Filters

Current router list input accepts optional period, order, and pageId (`packages/api/src/router/statusReport.ts:189-197`). It then passes a fixed `statuses: []`, `limit: 10_000`, and `offset: 0` to the service (`packages/api/src/router/statusReport.ts:199-211`).

Test-first story idea:

```text
Before adding UI status filters, write a router test:
Given reports with statuses investigating and resolved
When caller.statusReport.list({ pageId, statuses: ["resolved"] })
Then only resolved reports return
```

This test will fail today because tRPC list input does not accept `statuses` and passes `statuses: []` to the service (`packages/api/src/router/statusReport.ts:189-211`). The service already supports statuses (`packages/services/src/status-report/schemas.ts:79-87`; `packages/services/src/status-report/list.ts:172-181`).

Illustrative fake failing test:

```ts
// Illustrative fake code for future feature.
const reports = await caller.statusReport.list({
  pageId,
  statuses: ["resolved"],
});
expect(reports.every((report) => report.status === "resolved")).toBe(true);
```

