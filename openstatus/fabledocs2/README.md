# Onboarding: from the person who built this

Welcome. You're joining a codebase that runs a synthetic monitoring product: users define monitors, a fleet of Go probes checks them from ~28 regions, results land in an analytics store, status flips trigger alerts and public status pages. It looks simple from the outside. It is not simple. This pack is me telling you where the bodies are buried before you trip over them.

## How to read this

Six parts, in order. Each part tells you (1) what the subsystem does and why it exists at all, (2) the hardest design decisions I faced — including the alternatives I rejected and the tradeoffs that still bother me, (3) the exact files to read as evidence, and (4) a war story: an incident or bug this design caused or prevented. Each ends with three questions. **Don't move to the next part until you can answer them out loud** — not "I read it," but "I could defend it in a design review."

| Part | Covers | Read with these files open |
| --- | --- | --- |
| [01 Architecture](01-architecture.md) | The monorepo, the five deployables, the trust boundaries | `turbo.json`, `apps/workflows/src/checker/index.ts` |
| [02 API design](02-api-design.md) | Four front doors: tRPC, REST v1, RPC/MCP, machine endpoints | `packages/api/src/trpc.ts`, `apps/server/src/routes/v1/` |
| [03 Data access](03-data-access.md) | Drizzle/libSQL, Tinybird, transactions, schema evolution | `packages/db/src/schema/monitors/monitor.ts`, `packages/services/src/monitor/create.ts` |
| [04 Testing strategy](04-testing-strategy.md) | Shared-DB integration tests, rollback isolation, the coverage hole | `packages/services/test/helpers.ts` |
| [05 Error handling](05-error-handling.md) | ServiceError, fail-closed audit, at-most-once alerting | `packages/services/src/audit/emit.ts`, `apps/workflows/src/checker/alerting.ts` |
| [06 Observability](06-observability.md) | Wide events, tail sampling, the absence problem | `apps/server/src/index.ts` |

## Honesty note

The "production incident" sections are written the way I'd tell them over coffee. Some describe failures the design provably prevents; some describe failure modes the code provably permits, narrated as incidents so you feel the consequence. Every one of them is anchored to real lines in this repo — when I say "a crash between the trigger insert and the provider call suppresses the alert forever," you can go read `alerting.ts` and watch it happen. Treat the narratives as design analysis with a pulse, and verify every claim against the code. That habit — *verify the war story* — is the actual onboarding.

## The three rules that survive everything else

If you forget everything in this pack, remember these; they're the invariants the rest hangs off:

1. **Business logic lives in `packages/services`, one verb per file.** Routers are transport. If you're writing Drizzle in a router, you're either wrong or working in acknowledged legacy (`apps/server/src/routes/v1/index.ts:143-147` — the comment says so).
2. **Every user-facing mutation is: `requireScope` → parse → `withTransaction` → workspace-scoped write → `emitAudit`, atomically.** Miss the audit and the transaction rolls back — that's deliberate. (`packages/services/src/audit/emit.ts:92-149`)
3. **Two data planes, never mixed.** Transactional state in libSQL; probe telemetry in Tinybird. Cross-plane consistency is eventual and unenforced — UI code must tolerate an incident row while Tinybird still shows healthy pings.
