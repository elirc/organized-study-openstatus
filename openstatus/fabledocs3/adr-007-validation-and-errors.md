# ADR-007: Validation owned by the invariant owner; ServiceError + Actor union at the edges

**Status:** Accepted, stable. The 404-vs-403 inconsistency is a known cleanup; member-role authorization is an explicitly deferred gap.

## Context

Service verbs (ADR-002) are called from tRPC, REST, MCP, Slack, and background jobs. Each transport has its own wire validation, its own error vocabulary (TRPC codes vs HTTP statuses), and its own authentication (sessions, API keys, machine secrets, capability tokens). Without a policy, validation gets skipped on the transport that "surely already checked," and transport error types leak into domain code.

## Decision

Three rules. (1) **Validation belongs to whoever owns the invariant**: transports validate wire shape; every service verb *re-parses* its input at the top (`Input.parse`) because it cannot assume any transport validated — and DB reads are re-validated on the way out (persistence-out parse). (2) **Errors flow one direction**: services throw `ServiceError` subclasses (`NotFoundError`, `ForbiddenError`); each transport translates at its own edge (`toTRPCError` for tRPC, status codes for REST); `TRPCError` never appears under `packages/services/`. (3) **Callers are normalized into one `Actor` union** (user / apiKey / mcp / slack / system / webhook / subscriber); downstream code — audit attribution, `requireScope` — branches on the union, never on "which route called me." Scope RBAC (`read`/`write` on API keys) is enforced as the first line of every write verb; cross-tenant reads return NOT_FOUND (no existence leak).

## Evidence

- `packages/services/src/monitor/create.ts:22, 61` — domain parse at verb top; persistence-out parse on the returned row.
- `packages/api/src/service-adapter.ts:32-87` — ServiceError → TRPCError translation, `cause` preserved.
- `packages/services/src/context.ts:13-20` — the Actor union.
- `packages/services/src/auth/require-scope.ts:25-48` — scope enforcement; L19's docblock states the deliberate gap: no-op for `user` actors (member roles are a separate, unbuilt project — any workspace member can currently do anything in the workspace).
- `packages/services/src/monitor/internal.ts:28-84` — fetch-or-throw tenancy helpers (NOT_FOUND) beside tag validation returning FORBIDDEN: the noted inconsistency.
- `packages/services/src/page-subscriber/upsert.ts:183-197` — the fourth auth mode: capability tokens (possession = authorization), stripped from audit snapshots.

## Alternatives considered

1. **Validate once at the edge, trust internally.** Rejected: with five transports the "trusted" path multiplies; the domain parse is load-bearing the day someone adds a job that builds input programmatically.
2. **Shared error types across layers** (throw HTTP-ish errors from services). Rejected: welds domain code to one transport's vocabulary; the translation-at-the-edge pattern keeps the dependency arrow pointing inward.
3. **Per-route authorization middleware only.** Rejected as the *sole* mechanism: route middleware can't protect the Slack/job/MCP paths; putting `requireScope` and tenancy filters inside the verb protects every current and future caller. (REST v1 keeps route-level scope checks *as well* while legacy handlers remain — belt and suspenders during migration.)

## Consequences

**Positive:** no transport can reach domain logic un-validated; error semantics are consistent per-transport; IDOR resistance is structural (fetch-or-throw helpers + workspace filters + ownership-validated join ids); the Actor union makes "who did this" a first-class, auditable concept.

**Negative:** double-parsing looks redundant and gets flagged by every new reviewer (the answer must be re-taught); wire/domain schema duplication is hand-maintained with no drift detection; the 404/403 inconsistency confuses API consumers until unified; and the deferred member-role gap means the authorization story is honest but incomplete — precision is required when claiming "we have RBAC" (keys do; humans don't).
