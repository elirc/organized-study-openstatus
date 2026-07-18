# Validation, Auth & Permissions

## The validation layer map

| Layer | Mechanism | Anchor |
| --- | --- | --- |
| Wire (tRPC) | `.input(zod)` per procedure | [monitor.ts:80](../../../packages/api/src/router/monitor.ts#L80) |
| Wire (REST) | OpenAPI `createRoute` request schemas; zod errors via `defaultHook` | [post.ts:21-30](../../../apps/server/src/routes/v1/monitors/post.ts#L21-L30), [v1/index.ts:26](../../../apps/server/src/routes/v1/index.ts#L26) |
| Wire (queue) | zod parse of task payloads, 422 on failure | [checker/index.ts:96-99](../../../apps/workflows/src/checker/index.ts#L96-L99) |
| Domain | service re-parses input (`Input.parse` at verb top) | [create.ts:22](../../../packages/services/src/monitor/create.ts#L22) |
| Persistence-out | DB rows re-validated on read | [create.ts:61](../../../packages/services/src/monitor/create.ts#L61) |
| Response-out | `.output()` schemas strip fields | [statusPage.ts:76](../../../packages/api/src/router/statusPage.ts#L76) |

Why double-parse (wire + domain)? The service is called from tRPC, REST-in-future, Slack, MCP, and jobs — it can't assume any transport validated. **Validation belongs to whoever owns the invariant**, and the domain owns domain invariants.

## Authentication (who are you) — three front doors

1. **Dashboard session** (NextAuth): `enforceUserIsAuthed` → user + active workspace from `workspace-slug` cookie → context ([trpc.ts:148-200](../../../packages/api/src/trpc.ts#L148-L200)). Failure mode covered: cookie pointing at a workspace you left → resolver falls back and re-sets the cookie (L195–198).
2. **API keys**: `x-openstatus-key`; custom keys (prefix + hash verify + expiry) with Unkey fallback, super-admin sentinel, dev bypass — [auth.ts:124-261](../../../apps/server/src/libs/middlewares/auth.ts#L124-L261). Notice keys resolve to a **workspace** (ownerId), and attribution degrades loudly if no keyId ([auth.ts:100-119](../../../apps/server/src/libs/middlewares/auth.ts#L100-L119)).
3. **Machine secrets**: `CRON_SECRET` Basic header across cron/checker/updateStatus ([checker/index.ts:86-91](../../../apps/workflows/src/checker/index.ts#L86-L91), [checker.go:52](../../../apps/checker/handlers/checker.go#L52)). One static secret, no rotation story visible — flag it (risk register R4).

Plus a fourth, subtler one: **capability tokens** — subscriber verify/unsubscribe URLs carry a `crypto.randomUUID()` token with 7-day expiry ([upsert.ts:183-197](../../../packages/services/src/page-subscriber/upsert.ts#L183-L197)). Possession = authorization, which is why it's stripped from audit snapshots (L154–158).

## Authorization (what may you do)

Two active dimensions, one deliberate gap:
- **Tenancy** (row-level): every service read/write filters `workspaceId`; fetch-or-throw helpers make it structural ([internal.ts:28-47](../../../packages/services/src/monitor/internal.ts#L28-L47)); even join-table ids are ownership-validated ([validateTagIds → ForbiddenError](../../../packages/services/src/monitor/internal.ts#L63-L84)). This is the anti-**IDOR** machinery.
- **Scopes** (key-level RBAC): `read`/`write` on API keys; enforced in services (`requireScope` first line, [require-scope.ts:25-48](../../../packages/services/src/auth/require-scope.ts#L25-L48)) and at v1 route level while legacy handlers remain ([v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147)). MCP goes further: read-only keys never even *see* write tools (CLAUDE.md).
- **Member roles** (user-level): explicitly **absent** — `requireScope` is a no-op for `user` actors; the docblock says member-role enforcement is a separate project (L19). Any workspace member can do anything in that workspace today. State gaps precisely; don't discover them in review.

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| The second parse in services ("redundant!") | Which transports skip wire validation (Slack? jobs?) — the domain parse is load-bearing |
| 404 vs 403 semantics | Cross-tenant probes return NOT_FOUND (no existence leak) but tag validation returns FORBIDDEN — inconsistency to note |
| Access control on the status page "works" | It lives in middleware only ([proxy.ts:64-86](../../../apps/status-page/src/proxy.ts#L64-L86)); is the data layer also gated? (risk register R6) |
| The dev-mode key bypass is convenient | It grants `*` scopes ([auth.ts:246-261](../../../apps/server/src/libs/middlewares/auth.ts#L246-L261)) — verify it's unreachable in prod builds (guarded by NODE_ENV at L154) |

## Drills

1. Trace a read-only API key attempting `DELETE /v1/monitor/1` and a *user* attempting to delete another workspace's monitor via tRPC. Name the exact line each is stopped at. (Answers: `requireWriteScope` middleware / `getMonitorInWorkspace` NotFound.)
2. List every place `workspaceId` scoping could be forgotten in a *new* verb, and what convention catches each (helper usage, review checklist, `'rejects read-only actor'` + cross-tenant tests).
3. Write the one-paragraph security note you'd put in a PR adding a new public (unauthenticated) endpoint, using [upsert.ts](../../../packages/services/src/page-subscriber/upsert.ts#L52-L239) as the exemplar: plan gate, input scoping, capability token, idempotent no-op.

## Interview angle

- IDOR and multi-tenant isolation → [08/03 Q5](../08-interview-prep/03-api-and-data-modeling-questions.md); this page is the evidence base.
- "authn vs authz" → three front doors + two authz dimensions, with the honest gap.
- API key design → [08/03 Q6](../08-interview-prep/03-api-and-data-modeling-questions.md).
