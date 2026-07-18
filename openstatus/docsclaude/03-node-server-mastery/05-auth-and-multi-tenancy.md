# 05 — Auth and Multi-Tenancy

Multi-tenant SaaS lives and dies by tenant isolation. One missing `WHERE workspace_id = ?` and you've leaked data. This file covers how OpenStatus enforces isolation in layers and what to look for in reviews.

## The two auth systems

OpenStatus has *two* authentication surfaces:

- **Dashboard auth** — NextAuth (now Auth.js). Real users with accounts (email, GitHub, Google OAuth). Sessions stored as JWTs in cookies.
- **Status-page viewer auth** — a separate, simpler system for public/protected status pages. Viewers might subscribe to a private status page with an email link.

They live in [`apps/dashboard/src/lib/auth/`](../../apps/dashboard/src/lib/auth/) and [`apps/status-page/src/lib/auth/`](../../apps/status-page/src/lib/auth/) respectively. Don't conflate them.

## The session

NextAuth gives you a session: `{ user: { id, email, image, ... } }` or `null`. That's the *identity* of who's calling. It does **not** tell you which workspace they're acting in — a user can belong to many workspaces.

## Workspace resolution

The "active workspace" is encoded in a cookie (`workspace-slug`). Dashboard middleware ([`apps/dashboard/src/proxy.ts`](../../apps/dashboard/src/proxy.ts)) sets it; the tRPC context reader uses it ([`packages/api/src/trpc.ts:171-200`](../../packages/api/src/trpc.ts#L171-L200)):

```ts
const workspaceSlug = ctx.req?.cookies.get("workspace-slug")?.value;
const resolved = await resolveActiveWorkspace({
  userId: Number(ctx.session.user.id),
  workspaceSlug,
});
if (!resolved.ok) throw new TRPCError({ code: "UNAUTHORIZED", message: ... });
const { user, workspace } = resolved.value;
```

`resolveActiveWorkspace` does two things:
1. Verifies the user exists.
2. Verifies the user is a *member of* the workspace named by the slug. Falls back to a default workspace if the slug is missing or invalid.

The result is a typed `{ user, workspace }` that the middleware splices onto ctx. Every downstream procedure can trust those.

## `protectedProcedure` — the auth boundary

After the middleware runs, `ctx.user` and `ctx.workspace` are guaranteed non-null. The TypeScript type doesn't enforce this end-to-end (since the base `Context` has them as nullable), so the adapter cast happens in `toServiceCtx` ([`packages/api/src/service-adapter.ts:8-24`](../../packages/api/src/service-adapter.ts#L8-L24)):

```ts
type AuthedContext = Context & {
  user: { id: number };
  workspace: NonNullable<Context["workspace"]>;
};

export function toServiceCtx(ctx: AuthedContext): ServiceContext {
  return {
    workspace: ctx.workspace,
    actor: { type: "user", userId: ctx.user.id },
    requestId: ctx.req?.headers.get("x-request-id") ?? undefined,
  };
}
```

The cast is safe *because* the middleware ran. If a router uses `protectedProcedure`, `ctx.user` and `ctx.workspace` are populated. If you call `toServiceCtx(ctx)` from a public procedure, you'd be lying — TypeScript won't catch it unless `protectedProcedure` was the gate.

## Workspace scoping in the service layer

The middleware sets `ctx.workspace.id`. The service is then responsible for *using* it in every query and write.

Read pattern, [`packages/services/src/status-report/list.ts:172-182`](../../packages/services/src/status-report/list.ts#L172-L182):

```ts
const conditions: SQL[] = [eq(statusReport.workspaceId, ctx.workspace.id)];
if (input.statuses.length > 0) {
  conditions.push(inArray(statusReport.status, input.statuses));
}
// ... more conditions
const whereClause = and(...conditions);
```

The `workspaceId` filter is the *first* condition. Always. Every other filter is layered on top.

Write pattern, [`packages/services/src/status-report/create.ts:32-46`](../../packages/services/src/status-report/create.ts#L32-L46):

```ts
const page_ = await tx
  .select({ id: page.id })
  .from(page)
  .where(
    and(eq(page.id, input.pageId), eq(page.workspaceId, ctx.workspace.id)),
  )
  .get();
if (!page_) throw new NotFoundError("page", input.pageId);
```

Notice: the input gives a `pageId`, but the service doesn't trust it. It re-checks that the page belongs to the active workspace. **This is what prevents IDOR (Insecure Direct Object Reference).** If a malicious user sent the API a pageId from a different workspace, the lookup returns nothing, the service throws `NotFoundError`, and the adapter maps it to `NOT_FOUND` (404, *not* 403 — don't tell attackers whether the resource exists).

The test for this is intentional ([`packages/api/src/router/statusReport.test.ts:77-100`](../../packages/api/src/router/statusReport.test.ts#L77-L100)):

```ts
test("statusReport.create rejects pageId from another workspace", async () => {
  const ctx = createInnerTRPCContext({
    req: undefined,
    session: { user: { id: "1" } },
    // @ts-expect-error - minimal workspace for test
    workspace: { id: 1 },                     // user is in workspace 1
  });
  const caller = edgeRouter.createCaller(ctx);
  try {
    await caller.statusReport.create({
      pageId: otherWorkspacePageId,           // page in workspace 2
      // ...
    });
    throw new Error("Should have thrown");
  } catch (e) {
    expect(e).toBeInstanceOf(TRPCError);
    expect((e as TRPCError).code).toBe("NOT_FOUND");
  }
});
```

Every workspace-scoped entity should have one of these tests. If a new feature ships without one, push back in review.

## Scopes — API key RBAC

OpenStatus has *two* actor classes within the same service interface:

- **User actors** — dashboard sessions. Have full workspace access (limited by member role, which is a planned feature).
- **API key actors** — programmatic access via API keys. Have explicit scopes (`"read"` / `"write"`).

`requireScope` ([`packages/services/src/auth/require-scope.ts`](../../packages/services/src/auth/require-scope.ts)) gates write verbs:

```ts
export function requireScope(ctx: ServiceContext, required: Scope): void {
  const { actor } = ctx;
  if (actor.type !== "apiKey" && actor.type !== "mcp") return;   // user/system/etc. are no-op
  if (matchesScope(actor.scopes, required)) return;
  // log denial; throw
  throw new ForbiddenError(`API key lacks required scope: ${required}`);
}
```

Usage, [`packages/services/src/status-report/create.ts:24-30`](../../packages/services/src/status-report/create.ts#L24-L30):

```ts
export async function createStatusReport(args) {
  const { ctx } = args;
  requireScope(ctx, "write");                          // first line
  const input = CreateStatusReportInput.parse(args.input);
  return withTransaction(ctx, async (tx) => { ... });
}
```

The order matters: scope check *before* input parse, *before* opening a transaction. If the actor lacks scope, fail fast — don't waste DB connection time.

**Senior aside:** *`requireScope` returning early for `user` actors is a documented gap. When member-role enforcement lands for dashboard users, `requireScope` becomes the natural hook. The pattern is already in place; only the user branch needs filling in.*

## The audit log

Every mutation emits an audit row inside the same transaction ([`packages/services/src/status-report/create.ts:82-95`](../../packages/services/src/status-report/create.ts#L82-L95)):

```ts
await emitAudit(tx, ctx, {
  action: "status_report.create",
  entityType: "status_report",
  entityId: newReport.id,
  after: newReport,
});
```

Two guarantees:
1. **Fail-closed**: a failed audit insert rolls back the mutation. You can't have a successful write without a matching audit row.
2. **Workspace-scoped**: audit rows carry the workspace id; the audit reader can only see its own workspace's history.

This is the auditability layer a SOC2 reviewer asks about. Bake it in early; retrofitting is expensive.

## Cookies

OpenStatus uses three categories of cookies:

- **Session** (NextAuth) — `__Secure-authjs.session-token` or similar. HttpOnly, Secure, SameSite=Lax. Never read in client JS.
- **Workspace slug** (`workspace-slug`) — readable, used to remember the active workspace. Set by middleware and `<WorkspaceClientCookie />`.
- **Locale / theme** — preferences. Readable.

The session cookie is the only one that authenticates. Everything else is preference, and the server re-verifies any claim from a non-session cookie against the DB.

## NextAuth's `allowDangerousEmailAccountLinking`

If you skim [`apps/dashboard/src/lib/auth/providers.ts`](../../apps/dashboard/src/lib/auth/providers.ts), you'll see `allowDangerousEmailAccountLinking: true` on OAuth providers. This says: "if a GitHub login uses an email that already has an account, link them." It's a UX win and a phishing risk. OpenStatus accepts the tradeoff; review the decision per product if you're forking.

## Pitfall checklist

1. **Missing workspace filter** in a query. The IDOR test exists for a reason; add one for every new entity.
2. **Trusting input IDs.** The user can send any number. Always verify ownership.
3. **`protectedProcedure` skipped on a mutation.** Anonymous writes.
4. **Service called with a hand-crafted ctx** that bypasses auth checks. Don't.
5. **Returning the existence of cross-tenant resources via different error codes.** A 403 on a foreign resource tells the attacker it exists; a 404 doesn't. OpenStatus is consistent: 404 for both "not present" and "in another tenant."
6. **Forgetting `requireScope` on a write verb.** API keys with read-only scope can write.
7. **Audit emit outside the transaction.** Now a write can succeed without an audit row.
8. **Stripping secrets from audit `after`/`before` snapshots** — easy to forget; do it deliberately (see `integration/remove.ts`).
9. **Setting auth-relevant cookies in JS.** They become HttpOnly bypassable.
10. **Conflating dashboard and status-page auth.** Different worlds, different requirements.

## Drills

- Find every mutation service function. Verify the first line is `requireScope(ctx, "write")`. Note any exceptions and explain why.
- Find every read service function. Verify the query starts with a workspace filter. Note exceptions.
- Read the IDOR test in [`packages/api/src/router/statusReport.test.ts:77-207`](../../packages/api/src/router/statusReport.test.ts#L77-L207). Identify every scenario it covers. Suggest one it doesn't.
- Walk through what happens if a user manipulates the `workspace-slug` cookie to a workspace they don't belong to. Trace the rejection.
- Examine `emitAudit`. What happens if the audit insert throws? Trace the rollback path.
- Read [`packages/services/src/audit/`](../../packages/services/src/audit/) (if present). How are secrets stripped from `before`/`after` snapshots?
