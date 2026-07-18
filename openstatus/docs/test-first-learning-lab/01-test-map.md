# Test Map

## Test Surfaces

```text
packages/api/src/router/*.test.ts
  tests tRPC router behavior, procedure calls, error codes, cross-workspace guards

packages/services/src/**/__tests__/*.test.ts
  tests reusable domain services, transactions, audit rows, scopes, notifications

packages/utils/src/*.test.ts
  tests pure utilities and safety helpers

apps/**/src/lib/**/*.test.ts
  tests app-specific helpers and proxy/domain logic
```

Evidence:

- `statusReport.test.ts` creates an `edgeRouter` caller with `createInnerTRPCContext` (`packages/api/src/router/statusReport.test.ts:11-13`; `packages/api/src/router/statusReport.test.ts:77-99`).
- Service tests use `withTestTransaction`, seeded workspaces, and context helpers (`packages/services/src/status-report/__tests__/status-report.test.ts:19-39`; `packages/services/src/status-report/__tests__/status-report.test.ts:116-208`).
- SSRF tests cover allowed/blocklisted URL shapes (`packages/utils/src/ssrf.test.ts:26-160`).

## What To Test At Which Layer

| Behavior | Best Layer | Why |
| --- | --- | --- |
| tRPC input shape and error code | Router test | Procedure contract lives in router (`packages/api/src/router/statusReport.ts:24-222`) |
| Workspace ownership and transaction writes | Service test | Business rule lives in service (`packages/services/src/status-report/create.ts:24-99`) |
| Pure event computation | Unit test | `getEvents` is a pure-ish helper over arrays (`packages/api/src/router/statusPage.utils.ts:128-258`) |
| URL safety | Utility test | SSRF guard is a utility boundary (`packages/utils/src/ssrf.test.ts:26-160`) |
| Public integrated page shape | Router/integration test | `statusPage.get` computes public read model (`packages/api/src/router/statusPage.ts:65-400`) |

## Test Helper Map

`withTestTransaction` wraps a test body and always rolls back writes (`packages/services/test/helpers.ts:18-50`).

`loadSeededWorkspace` loads seeded workspace rows and parses them with `selectWorkspaceSchema` (`packages/services/test/helpers.ts:102-119`).

`makeUserCtx`, `makeApiKeyCtx`, `makeMcpCtx`, `makeSlackCtx`, and `makeSystemCtx` construct `ServiceContext` actors (`packages/services/test/helpers.ts:121-204`).

`expectAuditRow` asserts audit rows were written (`packages/services/test/helpers.ts:240-264`).

Illustrative fake service test skeleton:

```ts
// Illustrative fake code.
test("service writes audit row", async () => {
  await withTestTransaction(async (tx) => {
    const workspace = await loadSeededWorkspace();
    const ctx = { ...makeUserCtx(workspace), db: tx };
    const result = await serviceUnderTest({ ctx, input });
    await expectAuditRow({
      workspaceId: workspace.id,
      action: "thing.create",
      entityType: "thing",
      entityId: result.id,
      db: tx,
    });
  });
});
```

