# 02 — Writing Tests in This Repo

Concrete recipes for the test patterns you'll use most. Each section maps directly to existing code so you can copy-paste-modify.

## Setup

Tests use Bun's built-in runner. Run them with:

```sh
pnpm test                       # all packages
pnpm --filter @openstatus/services test
pnpm --filter @openstatus/api test
```

Before running, you need a local Turso DB running:

```sh
turso dev --db-file openstatus-dev.db
pnpm --filter @openstatus/db migrate
pnpm --filter @openstatus/db seed
```

Seeding produces predictable workspaces (free plan = id 2, team plan = id 1) and seeded users.

## Service test template

[`packages/services/src/status-report/__tests__/status-report.test.ts`](../../packages/services/src/status-report/__tests__/status-report.test.ts) is the canonical example. The skeleton:

```ts
import { afterAll, beforeAll, beforeEach, describe, expect, test } from "bun:test";
import { db, eq } from "@openstatus/db";
import { /* tables */ } from "@openstatus/db/src/schema";

import {
  SEEDED_WORKSPACE_FREE_ID,
  SEEDED_WORKSPACE_TEAM_ID,
} from "../../../test/fixtures";
import {
  expectAuditRow,
  loadSeededWorkspace,
  makeApiKeyCtx,
  makeUserCtx,
  withTestTransaction,
} from "../../../test/helpers";
import type { ServiceContext } from "../../context";
import { ForbiddenError, NotFoundError } from "../../errors";
import { createMyThing } from "../create";

let teamCtx: ServiceContext;
let freeCtx: ServiceContext;
let testParentId: number;

beforeAll(async () => {
  const team = await loadSeededWorkspace(SEEDED_WORKSPACE_TEAM_ID);
  const free = await loadSeededWorkspace(SEEDED_WORKSPACE_FREE_ID);
  teamCtx = makeUserCtx(team, { userId: 1 });
  freeCtx = makeUserCtx(free, { userId: 2 });
  // Insert any shared fixtures that rollback can't help (they're for *other* tests).
  testParentId = (await db.insert(parentTable).values({...}).returning().get()).id;
});

afterAll(async () => {
  // Clean up shared fixtures.
  await db.delete(parentTable).where(eq(parentTable.id, testParentId)).catch(() => undefined);
});

describe("createMyThing", () => {
  test("creates row and emits audit", async () => {
    await withTestTransaction(async (tx) => {
      const ctx = { ...teamCtx, db: tx };
      const result = await createMyThing({ ctx, input: { ... } });
      expect(result.id).toBeGreaterThan(0);

      await expectAuditRow({
        db: tx,
        workspaceId: teamCtx.workspace.id,
        action: "my_thing.create",
        entityId: result.id,
      });
    });
  });

  test("rejects when actor lacks write scope", async () => {
    const readOnlyCtx = makeApiKeyCtx(teamCtx.workspace, {
      keyId: "k", userId: 1, scopes: ["read"],
    });
    await expect(
      createMyThing({ ctx: readOnlyCtx, input: { ... } })
    ).rejects.toBeInstanceOf(ForbiddenError);
  });

  test("rejects cross-workspace parent reference", async () => {
    await withTestTransaction(async (tx) => {
      const ctx = { ...freeCtx, db: tx };  // free workspace
      await expect(
        createMyThing({ ctx, input: { parentId: testParentId /* team workspace */, ... } })
      ).rejects.toBeInstanceOf(NotFoundError);
    });
  });
});
```

Three tests per write verb, minimum: happy path with audit, scope rejection, cross-workspace rejection. If any are absent, add them before merging.

## Router test template

Router tests verify the wire boundary. The example: [`packages/api/src/router/statusReport.test.ts`](../../packages/api/src/router/statusReport.test.ts).

```ts
import { afterAll, beforeAll, expect, test } from "bun:test";
import { db, eq } from "@openstatus/db";
import { TRPCError } from "@trpc/server";

import { edgeRouter } from "../edge";
import { createInnerTRPCContext } from "../trpc";

let otherWorkspaceResourceId: number;

beforeAll(async () => {
  otherWorkspaceResourceId = (await db.insert(resource).values({
    workspaceId: 2, ...
  }).returning().get()).id;
});

afterAll(async () => {
  await db.delete(resource).where(eq(resource.id, otherWorkspaceResourceId));
});

test("resource.create rejects cross-workspace reference", async () => {
  const ctx = createInnerTRPCContext({
    req: undefined,
    session: { user: { id: "1" } },
    // @ts-expect-error - minimal workspace for test
    workspace: { id: 1 },                       // caller is in workspace 1
  });
  const caller = edgeRouter.createCaller(ctx);

  try {
    await caller.resource.create({ parentId: otherWorkspaceResourceId, ... });
    throw new Error("Should have thrown");
  } catch (e) {
    expect(e).toBeInstanceOf(TRPCError);
    expect((e as TRPCError).code).toBe("NOT_FOUND");
  }
});
```

Notice: the test bypasses `protectedProcedure`'s normal workspace resolution by pre-populating `ctx.workspace`. This is what the "test escape hatch" in `enforceUserIsAuthed` is for ([`packages/api/src/trpc.ts:154-169`](../../packages/api/src/trpc.ts#L154-L169)). Without it, the middleware would overwrite your test's workspace with whatever the DB resolves.

## Audit assertion helper

`expectAuditRow` (in [`packages/services/test/helpers.ts`](../../packages/services/test/helpers.ts)) asserts an audit row exists for a workspace + action + entity. Use it on every mutation test.

```ts
await expectAuditRow({
  db: tx,
  workspaceId: teamCtx.workspace.id,
  action: "status_report.create",
  entityType: "status_report",
  entityId: report.id,
});
```

If the audit row is missing, your service skipped `emitAudit` or emitted with the wrong action name. The test catches it before review does.

## Notification spy assertion

For services that dispatch notifications, the test framework installs a spy ([`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`](../../packages/services/src/status-report/__tests__/status-report.test.ts#L41-L56)):

```ts
const subscriptionSpies = (globalThis as Record<string, unknown>).__subscriptionSpies as { ... };

beforeEach(() => {
  subscriptionSpies?.dispatchStatusReportUpdate.mockClear();
});

test("notifyStatusReport dispatches to subscribers", async () => {
  await notifyStatusReport({ ctx, input: { id: report.id } });
  expect(subscriptionSpies?.dispatchStatusReportUpdate.mock.calls.length).toBe(1);
});
```

The spy is installed by `test/preload.ts` (Bun's preload hook). You don't need to set it up per-test; you just assert against it.

## Testing time-sensitive code

If your service reads `Date.now()`, you need either:

- A `now()` parameter (refactor the service to take it).
- A library that fakes time (Bun has `setSystemTime` available in tests, or you can monkey-patch).
- Looser assertions ("created within the last second").

Loose assertions are tempting; they're fragile. Prefer the refactor when feasible.

## Testing concurrency

Race conditions are hard to test deterministically. Patterns:

- **Run twice in parallel:** `await Promise.all([fn(), fn()])` and assert exactly one succeeded, one failed with a uniqueness error.
- **Inject delays:** wrap the DB driver in a stub that delays specific queries.
- **Replay a sequence:** record real production logs of a concurrency bug, replay against the new code.

OpenStatus doesn't have explicit concurrency tests in the OSS surface. If you add a feature with a race window, add the test that demonstrates uniqueness wins.

## Testing the form layer

The form is a React component using `react-hook-form` + `zodResolver`. You can test it with `@testing-library/react`:

```ts
import { render, screen, fireEvent } from "@testing-library/react";
import { FormStatusReport } from "./form";

test("submits with selected status", async () => {
  const onSubmit = vi.fn().mockResolvedValue(undefined);
  render(<FormStatusReport items={[]} onSubmit={onSubmit} />);

  fireEvent.change(screen.getByLabelText(/title/i), { target: { value: "Test" } });
  fireEvent.click(screen.getByRole("button", { name: /submit/i }));

  await waitFor(() => expect(onSubmit).toHaveBeenCalledWith(
    expect.objectContaining({ title: "Test", status: "investigating" })
  ));
});
```

OpenStatus doesn't ship form tests in the OSS repo; if you're investing in UI quality, this is where to start.

## Testing the dashboard (component tests vs e2e)

A component test renders a piece of UI in isolation, fakes the network, asserts on output. Fast, focused.

An e2e test drives a real browser through a real app. Slow, comprehensive.

For OpenStatus's surface area, a sensible split:
- Component tests for forms, complex tables, and the trickier hooks.
- 2–3 e2e tests for the highest-value flows (signup → first monitor; create incident → public page shows it).

## When to write a test before the code

TDD ("write the test first") is most useful when:

- You can name the requirement precisely.
- The implementation has multiple valid shapes.
- You want a fast iteration loop.

Skip TDD when:

- You're exploring (you don't know the requirement yet).
- The "test" would just restate the code.
- The code is mostly plumbing (move ten variables around).

Senior engineers use TDD selectively. The dogmatic versions of "always TDD" or "never TDD" both leave value on the table.

## Pitfall checklist

1. **Forgetting `withTestTransaction`.** Your test pollutes the DB; next run fails.
2. **Reusing the same row id across tests.** Order-dependent failures.
3. **Asserting on order** of a result set without an explicit ORDER BY.
4. **Mocking the DB.** You're now testing your mock, not your code.
5. **`await expect(promise).rejects.toThrow()` without checking error type.** Catches any throw, including bugs in setup.
6. **Tests that need a network.** Mock the boundary.
7. **Tests that assume a specific time.** Use clock injection or relative assertions.
8. **Tests that read from `process.env`.** Make env explicit in setup.
9. **Suite-level `beforeAll` mutations** that other suites depend on. Suite isolation breaks.
10. **Tests that pass when the code is deleted.** No assertions on the actually meaningful behavior.

## Drills

- Pick a service verb that doesn't have a cross-workspace rejection test. Write one.
- Find a write verb that uses `requireScope`. Verify it has a "rejects read-only actor" test. If not, add one.
- Write a test that explicitly verifies the audit row's `before` field is `null` for a create and `null` for a delete's `after`. Useful to catch swap bugs.
- Add a notification spy assertion to a test that exercises a notification path.
- Run `pnpm test` and time the suite. Identify the slowest 3 tests. Are they slow because they're doing real work, or because of accidental waits?
