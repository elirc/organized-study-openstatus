# API & Data-Modeling Question Cards

Twelve cards, all repo-anchored.

## Q1: REST vs RPC vs GraphQL — how do you choose?
Repo anchor: both live here: tRPC for the first-party dashboard ([root.ts](../../../packages/api/src/root.ts#L5-L6)), OpenAPI REST for external users ([v1/index.ts](../../../apps/server/src/routes/v1/index.ts#L25-L27)), plus ConnectRPC v2 ([server index.ts:187](../../../apps/server/src/index.ts#L187)).
Junior: recites definitions.
Mid adds: choose by *consumer relationship* — versionless inferred contracts for clients you deploy with; documented, versioned, stable contracts for clients you don't control.
Senior includes: the shared services layer beneath both is what makes running three surfaces affordable ([create.ts](../../../packages/services/src/monitor/create.ts#L16-L63)); contract governance (who may break what) is the real question, not the acronym.

## Q2: Model a multi-tenant SaaS schema. What's non-negotiable?
Repo anchor: `workspaceId` on every table ([monitor.ts:37](../../../packages/db/src/schema/monitors/monitor.ts#L37)); scoped fetch helpers ([internal.ts:28-47](../../../packages/services/src/monitor/internal.ts#L28-L47)).
Mid adds: tenant key on every row + every query; 404-not-403 for foreign rows; per-tenant quotas as data ([workspace.ts:22](../../../packages/db/src/schema/workspaces/workspace.ts#L22)).
Senior includes: enforcement layers ranked — app convention (here) < helper-enforced (here) < DB-enforced (RLS/FKs); notes this repo's nullable workspaceId as the kind of gap you'd fix with a staged NOT NULL migration.
Follow-up: "how do you *test* isolation?" → the cross-tenant rejection recipe ([05/02 Recipe 4](../05-quality-engineering/02-writing-tests-here.md)).

## Q3: Soft delete — tradeoffs?
Repo anchor: `deletedAt` + partial index ([monitor.ts:66-72](../../../packages/db/src/schema/monitors/monitor.ts#L66-L72)); every query must filter it ([internal.ts:41](../../../packages/services/src/monitor/internal.ts#L41)).
Mid adds: recovery/audit value vs the forever-tax of `WHERE deleted_at IS NULL`; partial indexes keep the live working set fast.
Senior includes: uniqueness interactions (can a deleted row's slug be reused?), cascade semantics for children, and the drill finding from [03/02](../03-architecture-and-patterns/02-data-model-and-persistence.md): does the *scheduler* filter deletedAt? A soft delete is only as good as its most forgetful reader.

## Q4: Pagination — offset vs cursor?
Repo anchor: audit list ordering by monotonic id ([helpers.ts:206-211](../../../packages/services/test/helpers.ts#L206-L211) documents why id, not timestamp); Tinybird pipes take date ranges ([client.ts:74-97](../../../packages/tinybird/src/client.ts#L74-L97)).
Mid adds: offset drifts under concurrent writes and gets slow deep; cursor = stable sort key (id, created_at+id).
Senior includes: cursor choice is a *uniqueness* argument (same-millisecond ties — exactly the repo's id-DESC reasoning); pagination is part of the public contract (adding it later is a breaking change).

## Q5: What's IDOR and how do you prevent it structurally?
Repo anchor: [internal.ts:28-47](../../../packages/services/src/monitor/internal.ts#L28-L47), join-row validation [internal.ts:63-84](../../../packages/services/src/monitor/internal.ts#L63-L84).
Mid adds: never trust ids from the client; scope every lookup; return 404 to avoid existence oracles.
Senior includes: the *structural* part — helpers make the safe path the default; tests make it a convention (`'rejects read-only actor'` + cross-tenant cases); relation ids (tags, notification ids in a monitor update) are the ones everyone forgets.

## Q6: Design API keys: storage, verification, permissions, rotation.
Repo anchor: [auth.ts:159-226](../../../apps/server/src/libs/middlewares/auth.ts#L159-L226) — prefix (`os_` + 8 hex) for lookup, hash for verify, expiry, scopes, debounced lastUsedAt, legacy-provider fallback with documented default scopes.
Mid adds: hash-not-store (like passwords), prefix for O(1) lookup + UX display, scopes for least privilege.
Senior includes: migration story (custom-first, Unkey fallback = provider transition without breaking keys); attribution (keyId into audit logs, fallback warned loudly at [auth.ts:106-114](../../../apps/server/src/libs/middlewares/auth.ts#L106-L114)); the `'*'` internal-only scope kept out of public APIs.

## Q7: Where does business logic live, and how do you keep it there?
Repo anchor: [CLAUDE.md](../../../CLAUDE.md) services doctrine; thin router [monitor.ts:78-93](../../../packages/api/src/router/monitor.ts#L78-L93) vs legacy inline handler [post.ts:44-90](../../../apps/server/src/routes/v1/monitors/post.ts#L44-L90).
Mid adds: the service signature (ctx + input), error taxonomy, adapter per transport.
Senior includes: *migration reality* — both styles coexist with an in-code comment declaring direction ([v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147)); how you move a live API without breaking it (ticket M1's byte-equality tests).

## Q8: Design middleware for an API server. What order and why?
Repo anchor: [server index.ts:105-181](../../../apps/server/src/index.ts#L105-L181) — sentry → requestId → wide event → routes → onError; then per-group auth → scope gate ([v1/index.ts:142-147](../../../apps/server/src/routes/v1/index.ts#L142-L147)).
Mid adds: ordering rationale: identity/tracing outermost, auth before handlers, error handler wraps everything.
Senior includes: the wide-event middleware *finalizes after* `await next()` — the onion shape enables tail sampling; per-route vs global middleware cost.

## Q9: Make a webhook/queue consumer safe under retries.
Repo anchor: `/updateStatus` — shared-secret auth, zod parse, upsert (idempotent), dedupe-by-unique-insert for the side effect ([checker/index.ts:86-126](../../../apps/workflows/src/checker/index.ts#L86-L126), [alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139)).
Mid adds: idempotency key from business facts; effects deduped, state upserted; 4xx vs 5xx signaling to the queue (retry only 5xx).
Senior includes: the at-most-once vs at-least-once choice hiding in insert-before-send ordering ([03/04's analysis](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)) — deliver this and you've won the round.

## Q10: How do you ship a breaking schema change with three services on one DB?
Repo anchor: drizzle-kit flow + CI order ([test.yml:51-57](../../../.github/workflows/test.yml#L51-L57)); multi-app constraint from the [system map](../01-codebase-cartography/01-system-map.md).
Mid adds: expand → migrate/backfill → contract; each deploy backward-compatible one step.
Senior includes: forward-only migrations ("rollback is a new migration"); zod read-schemas must move in lockstep with columns ([03/02 step 2](../03-architecture-and-patterns/02-data-model-and-persistence.md)); deploy *order* across dashboard/server/workflows stated explicitly in the PR.

## Q11: Transactions — when do you need one, and what belongs inside?
Repo anchor: quota+insert+audit atomically ([create.ts:24-62](../../../packages/services/src/monitor/create.ts#L24-L62)); counter-example: external sends inside a tx (review [Kata 3](../04-code-reading-gym/04-review-katas.md)).
Mid adds: invariants spanning multiple writes need atomicity; keep txs short; never do network I/O inside.
Senior includes: ambient-tx composition ([withTransaction](../../../packages/services/src/context.ts#L40-L47)); isolation isn't free or uniform (libsql-HTTP caveat, R7); some pipelines *choose* no-tx + idempotency instead — name when that's right.

## Q12: Cache design for a public read-heavy endpoint.
Repo anchor: status page: Tinybird pipe 5-min public cache ([client.ts:11-13](../../../packages/tinybird/src/client.ts#L11-L13)), Next revalidation, the heavy relational query with its TODO ([statusPage.ts:50](../../../packages/api/src/router/statusPage.ts#L50)).
Mid adds: cache key = (page, time bucket); staleness budget from product ("status must update within X min"); layered: CDN/ISR/data cache.
Senior includes: invalidation on report publish (event-driven purge vs TTL); *what must never be cached* (access-control decisions — the proxy runs per-request for a reason).

---

Protocol: pair each card with its module-03 page for depth. Cards Q6, Q9, Q10 are disproportionately asked — over-practice those.
