# ADR-006: Split contract strategy — versionless tRPC for the dashboard, contract-first REST v1 for the public

**Status:** Accepted, stable; newer machine surfaces (ConnectRPC v2, MCP, Slack) extend the same principle: contract stability priced per consumer.

## Context

Two kinds of API consumer with opposite needs. The dashboard is a first-party client that deploys in lockstep with its server and wants dozens of UI-shaped queries. External integrators need a documented, stable, key-authenticated API whose breaking changes are versioning events. One API serving both either ossifies UI queries into public contracts or makes the public API unstable.

## Decision

Two surfaces with explicitly different stability promises. (1) **tRPC** (`packages/api`) for the dashboard: the `AppRouter` type is the contract, it is versionless, and breaking changes are free because client and server ship together. (2) **REST v1** (`apps/server/src/routes/v1/`) for the world: **contract-first** — every route declares request/response as zod schemas inside `createRoute`, and the OpenAPI document is *generated from the validators*, so docs cannot describe behavior the code doesn't enforce. Validation failures share one shape via a single `defaultHook`. API keys authenticate REST; sessions authenticate tRPC; both resolve to the same `Actor`/workspace context so the service layer (ADR-002) never knows which door was used.

## Evidence

- `packages/api/src/root.ts:5-6` — `AppRouter` as the (dashboard-only) contract.
- `apps/server/src/routes/v1/monitors/post.ts:15-42` — `createRoute` with zod request/response schemas.
- `apps/server/src/routes/v1/index.ts:26` — the shared `defaultHook` error shape; L142-161 — key-authed, OpenAPI-documented registration.
- `packages/api/src/router/monitor.ts:39-61` — wire schema deliberately distinct from the service's domain schema (see ADR-007).
- `apps/server/src/index.ts:187-258` — ConnectRPC/MCP/Slack surfaces mounted beside v1.

## Alternatives considered

1. **Dogfood the public REST API from the dashboard.** Rejected: UI-shaped queries would become permanent public contracts; every dashboard iteration would pay versioning tax. The cost — two implementations per capability — is what ADR-002's services layer exists to contain.
2. **GraphQL for both.** Rejected (inferred): solves the shaped-query problem but reintroduces a single public contract, plus resolver-authorization complexity across seven actor kinds; tRPC gives the type-safety win with zero schema infrastructure for the one client that needs it.
3. **Hand-written OpenAPI spec** (or docs-from-annotations). Rejected: drifts from the validator within weeks; generating the spec *from* the zod schemas makes drift structurally impossible.

## Consequences

**Positive:** dashboard iteration is frictionless; public docs are always true; one error shape across the public surface; wire validation errors are consistent and machine-parseable.

**Negative:** every capability exists twice until the services migration completes (v1 handlers still inline Drizzle — `v1/index.ts:143-147`); zod's default parse **silently strips unknown request fields**, so a typoed field name returns 200 and does nothing — fixing it with `.strict()` is now itself a breaking change, which is the argument for strictness in v1 of any API; and the tRPC surface's freedom depends on nobody-but-the-dashboard consuming it — the moment a third party reverse-engineers it, its versionlessness becomes a liability.
