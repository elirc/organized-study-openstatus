# Reverse-engineered ADRs

Nobody wrote these down at the time — this folder reconstructs the 10 most significant architecture decisions the codebase's authors implicitly made, in standard ADR format (Context / Decision / Alternatives Considered / Consequences / Status), each grounded in specific file evidence. "Reverse-engineered" means the *decision* is inferred from the code; the *evidence* is not — every citation is a real file you can open. Where a rationale is inference rather than something the code states (in a comment, a helper name, a test), it's phrased as inference.

| # | ADR | Domain |
| --- | --- | --- |
| 001 | [Monorepo with apps-as-deployables, packages-as-contracts](adr-001-monorepo-layering.md) | Layering / structure |
| 002 | [All business logic in a services layer, one verb per file](adr-002-services-layer.md) | Layering / structure |
| 003 | [Go probe fleet behind a data-only boundary, GCP Cloud Tasks as the spine](adr-003-go-checker-and-queue.md) | Distributed systems |
| 004 | [Two data planes: libSQL for state, Tinybird for telemetry](adr-004-two-data-planes.md) | Data access |
| 005 | [libSQL/Turso as system of record, zod-beside-table, serialized columns](adr-005-libsql-and-schema-style.md) | Data access |
| 006 | [Split contract strategy: versionless tRPC + contract-first REST v1](adr-006-split-api-contracts.md) | API contract design |
| 007 | [Validation owned by the invariant owner; ServiceError + Actor union at the edges](adr-007-validation-and-errors.md) | API / error handling |
| 008 | [Fail-closed, in-transaction audit log](adr-008-fail-closed-audit.md) | Error handling philosophy |
| 009 | [Reliability via layered idempotency instead of transactions or an outbox](adr-009-idempotency-over-outbox.md) | Distributed systems |
| 010 | [Integration-first testing on a shared seeded DB with rollback isolation](adr-010-integration-first-testing.md) | Testing |

Then: **[Interview translation](interview-translation.md)** — each ADR as a 90-second spoken answer plus the follow-up question an interviewer is most likely to ask next.

Companion docs: the deep-dive curriculum lives in `../fabledocs/upskill/` (risk register: `09-reference/risk-register.md`); the narrative onboarding version of much of this material is `../fabledocs2/`.
