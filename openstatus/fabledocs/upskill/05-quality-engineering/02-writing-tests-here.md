# Writing Tests Here — Recipes

Six recipes using the repo's actual helpers. Commands are __inferred__ (nothing executed while authoring — see [verification log](../09-reference/verification-log.md)). Prereq for all service tests: `turso dev` running, `pnpm migrate && pnpm seed` done in `packages/db`.

Run a single package: `cd packages/services && bun test` — narrow further with a path: `bun test src/monitor` or `bun test --test-name-pattern "audit"`.

## Recipe 1: Service happy path + audit assertion

The canonical shape, copied from [monitor.test.ts:79-108](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L79-L108):

```ts
// Illustrative fake code: not from this repo (adapted from the real test cited above)
test("creates X and emits audit", async () => {
  await withTestTransaction(async (tx) => {
    const ctx = { ...teamCtx, db: tx };          // ← thread the tx or you commit for real
    const row = await createX({ ctx, input: {...} });
    expect(row.workspaceId).toBe(teamCtx.workspace.id);
    await expectAuditRow({
      workspaceId: teamCtx.workspace.id,
      action: "x.create", entityType: "x", entityId: row.id,
      db: tx,                                     // ← audit reads must also use tx
    });
  });
});
```

The two `db: tx` lines are where everyone gets bitten (documented at [helpers.ts:29-35](../../../packages/services/test/helpers.ts#L29-L35)).

## Recipe 2: Validation failure

```ts
// Illustrative fake code: not from this repo
test("rejects invalid url", async () => {
  await withTestTransaction(async (tx) => {
    const ctx = { ...teamCtx, db: tx };
    expect(createMonitor({ ctx, input: { ...valid, url: "" } })).rejects.toThrow(); // ZodError
  });
});
```

Assert the error *type* when the contract matters: `.rejects.toBeInstanceOf(ZodError)` — services throw raw ZodError; routers translate ([service-adapter.ts:34-40](../../../packages/api/src/service-adapter.ts#L34-L40)).

## Recipe 3: Permission failure (read-only API key) — the house standard

Convention from CLAUDE.md, factory at [makeApiKeyCtx](../../../packages/services/test/helpers.ts#L132-L151):

```ts
// Illustrative fake code: not from this repo
test("rejects read-only actor", async () => {
  const ctx = makeApiKeyCtx(teamCtx.workspace, { keyId: "k", userId: 1, scopes: ["read"] });
  expect(deleteMonitor({ ctx, input: { id: 999999 } })).rejects.toBeInstanceOf(ForbiddenError);
});
```

No transaction needed — `requireScope` fires before any DB access ([require-scope.ts:21-23](../../../packages/services/src/auth/require-scope.ts#L21-L23) docblock), which is why the fake id is fine. If your version needs a real row, your verb checks scope too late: that's a *finding*, not a test problem.

## Recipe 4: Cross-tenant rejection

```ts
// Illustrative fake code: not from this repo
test("cannot touch another workspace's monitor", async () => {
  await withTestTransaction(async (tx) => {
    const rowInTeam = await createMonitor({ ctx: { ...teamCtx, db: tx }, input: valid });
    const freeCtx2 = { ...freeCtx, db: tx };
    expect(updateMonitorGeneral({ ctx: freeCtx2, input: { ...valid, id: rowInTeam.id } }))
      .rejects.toBeInstanceOf(NotFoundError);   // 404, not 403 — no existence leak
  });
});
```

Both fixture workspaces are pre-seeded ([monitor.test.ts:38-42](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L38-L42)).

## Recipe 5: Quota rejection without state pollution

Use the free workspace + hygiene helper (the reason it exists — [helpers.ts:52-63](../../../packages/services/test/helpers.ts#L52-L63)):

```ts
// Illustrative fake code: not from this repo
beforeAll(async () => { await cleanQuotaGatedTables(freeWorkspace.id); });
test("enforces page limit", async () => {
  await withTestTransaction(async (tx) => {
    const ctx = { ...freeCtx, db: tx };
    await createPage({ ctx, input: pageInput(1) });               // fills the free quota (1)
    expect(createPage({ ctx, input: pageInput(2) }))
      .rejects.toBeInstanceOf(LimitExceededError);
  });
});
```

## Recipe 6: HTTP contract test (Hono route)

Pattern used across `apps/server/src/routes/v1/**/*.test.ts` (open [get.test.ts](../../../apps/server/src/routes/v1/monitors/get.test.ts) and mirror it):

```ts
// Illustrative fake code: not from this repo
import { app } from "@/index";
test("401 without key", async () => {
  const res = await app.request("/v1/monitor/1");
  expect(res.status).toBe(401);
});
test("200 with dev key", async () => {
  const res = await app.request("/v1/monitor/1", { headers: { "x-openstatus-key": "1" } });
  expect(res.status).toBe(200);
});
```

Why `"1"` works in tests: dev/test mode treats the key as the ownerId with `*` scopes ([auth.ts:246-261](../../../apps/server/src/libs/middlewares/auth.ts#L246-L261)) — know this or the tests look like magic.

## Checklist before opening a PR with tests

- [ ] Every new mutation test asserts its audit row (house invariant)
- [ ] `db: tx` threaded everywhere inside `withTestTransaction`
- [ ] Negative tests assert error *types*, not message strings
- [ ] No sleeps; no wall-clock assertions (see Kata 8 for why)
- [ ] Suite runs green twice in a row locally (state-leak smoke test)
