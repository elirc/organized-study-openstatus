# ADR-001: Monorepo with apps-as-deployables, packages-as-contracts

**Status:** Accepted, stable.

## Context

One product, several very different runtimes: two Next.js frontends (authenticated dashboard, public status page), a Bun/Hono external API, a Bun/Hono pipeline service on Fly, and a Go probe binary deployed to ~28 regions. They share a database schema, business rules, notification providers, and UI components. The team is small; the alternative costs are coordination overhead (polyrepo) or coupling (monolith).

## Decision

A pnpm-workspaces + Turborepo monorepo. `apps/*` are deployables and own nothing reusable; `packages/**/*` are internal libraries and own all shared contracts: `packages/db` (schema + Drizzle client, no business rules), `packages/services` (all business logic), `packages/api` (tRPC routers), `packages/tinybird`, `packages/notifications/*` (one sub-package per provider), `packages/ui`, etc. Turbo encodes the dependency pipeline, including the developer-experience chain `dx → seed → migrate → env`.

## Evidence

- `package.json:38` — workspace globs `apps/*`, `packages/**/*`.
- `turbo.json` — build/dev/test/dx pipeline with task dependency edges.
- `packages/db/src/db.ts:6-12` (shared client), `packages/services/src/` (one directory per entity), `packages/notifications/` (per-provider sub-packages).
- The Go app (`apps/checker/`) sits inside the monorepo but outside the pnpm graph — it shares *no code*, only a JSON contract (see ADR-003).

## Alternatives considered

1. **Polyrepo** (one repo per app + published internal packages). Rejected: version-bumping `@openstatus/db` through npm for four consumers on every schema change is pure friction at this team size; atomic cross-app changes (schema + service + router + UI in one PR) are the common case here, not the exception.
2. **Single Next.js monolith** (API routes + cron in one app). Rejected: the probe/alerting workload has different runtime, scaling, and latency needs than the CRUD UI (see ADR-003/004); a monitoring product whose pipeline shares a failure domain with its dashboard is self-defeating.
3. **Nx or Bazel** instead of Turborepo. Rejected (inferred): Turbo's task-graph model is enough; nothing here needs Bazel-grade hermeticity, and the Go app opts out of the JS toolchain entirely anyway.

## Consequences

**Positive:** atomic cross-cutting PRs; one clone gives a working dev environment (`pnpm dx`); package boundaries make dependency direction enforceable (e.g. `packages/services` cannot import `TRPCError` without adding a dependency edge someone would see in review).

**Negative:** the monorepo hides that these are *independently deployed* services sharing one database — deploy-order discipline (additive migrations, one-version compatibility) is invisible in the repo structure and lives only in convention. `turbo test` runs `cache: false` because tests aren't hermetic, so CI time grows linearly with the repo. The Go app's presence is structural but not tooling-integrated: nothing in CI validates the TS↔Go payload contract.
