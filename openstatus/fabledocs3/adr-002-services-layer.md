# ADR-002: All business logic in a services layer, one verb per file

**Status:** Accepted; adoption in progress — v1 REST handlers are acknowledged legacy still being migrated (`apps/server/src/routes/v1/index.ts:143-147`).

## Context

The same business capability (create a monitor, with quota check, tenancy scoping, and audit) is reachable from at least five transports: tRPC (dashboard), REST v1, MCP, Slack, and background jobs. Early code put logic in routers; the moment a second transport needed it, the logic either got duplicated (drift) or the router got imported from weird places (coupling). Additional constraint: the dashboard runs tRPC on Next.js **Edge**, so shared logic must not import `node:*`.

## Decision

Every workspace-scoped business operation is a **service verb**: one file per verb under `packages/services/src/<entity>/` (`create.ts`, `update.ts`, `remove.ts`), uniform signature `verbEntity({ ctx, input })` where `ctx` carries workspace, actor, and optional transaction. The verb owns the full invariant chain: `requireScope` → `Input.parse` → `withTransaction` → workspace-scoped reads/writes → `emitAudit`. Routers are thin adapters: validate wire input, call the verb, translate errors. The repo's CLAUDE.md codifies this as the house rule; inline DB access in a router is a named smell.

## Evidence

- `packages/services/src/monitor/create.ts:16-63` — the canonical verb: scope, parse, transaction, quota, insert, audit.
- `packages/services/src/context.ts:13-20` — the `Actor` union (7 caller kinds) that makes verbs transport-agnostic.
- `packages/api/src/service-adapter.ts:32-87` — error translation lives in the transport, not the service.
- `packages/services/src/audit/emit.ts:14-24` — hand-rolled `deepEqual` instead of `node:util`: the Edge-safety constraint made visible.
- `apps/server/src/routes/v1/monitors/post.ts:44-90` — the counter-evidence: legacy inline Drizzle with duplicated quota logic, marked transitional in `v1/index.ts:143-147`.

## Alternatives considered

1. **Fat routers per transport.** Rejected: already tried implicitly; produced the duplicated quota logic still visible in v1. Five transports × N entities of duplicated invariants doesn't scale.
2. **Repository pattern / entity classes (DDD-style aggregates).** Rejected (inferred): heavier ceremony than a function-per-verb; verbs map 1:1 to product actions and to audit action names (`monitor.update`), which keeps the audit taxonomy trivially derivable.
3. **Database-enforced invariants only** (constraints, triggers). Rejected: quota math and scope checks need application context (plan limits, actor kind); SQLite's trigger/constraint toolkit is also weak (ADR-005).

## Consequences

**Positive:** invariants (tenancy, quota, scopes, audit) are written once and hold for every transport, including future ones; the verb file is the single place to review for any mutation; Edge-safety is preserved by construction as long as code stays in the layer; tests target the verb and pin the invariant for everyone (ADR-010).

**Negative:** a migration tax — until v1 finishes moving, the repo contains two implementations of some capabilities and the "every mutation is audited" invariant is only true for the migrated ones. Edge-safety is enforced by review, not tooling: nothing stops a `node:crypto` import until a bundle breaks. Schema duplication with the wire layer is deliberate but hand-maintained (ADR-006/007). And the pipeline (`apps/workflows`) bypasses the layer entirely for throughput, so its writes are unaudited — a precision cost when stating the invariant (see ADR-008).
