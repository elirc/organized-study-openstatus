# Flashcards

Sixty retrieval cards. Format: **Q** → answer hidden behind the fold. Use them spaced (10 min/day, shuffle, say answers *out loud*), not in one sitting — recognition feels like knowledge and isn't. Cards marked ⚡ are the ones interviewers actually ask; cards marked 📍 are anchor cards (answer = file:line) — for those, the standard is "I can open the file to the right region without searching."

## A. System shape

1. **Q:** Name the five deployables that matter and each one's runtime. <details><summary>A</summary>dashboard (Next.js), status-page (Next.js), server (Bun+Hono), workflows (Bun+Hono on Fly), checker (Go/Gin on Fly/Koyeb/Railway).</details>
2. **Q:** What do `apps/` own vs `packages/`? <details><summary>A</summary>Apps = deployables, own nothing reusable; packages = all shared contracts (db, services, api, tinybird, notifications, ui).</details>
3. ⚡ **Q:** Why is the checker written in Go? <details><summary>A</summary>Tiny static binary deployable to any region in seconds + per-request control of timeouts/redirects/DNS — probe quality is the product.</details>
4. **Q:** What crosses the TS↔Go boundary? <details><summary>A</summary>One JSON payload, typed independently on both sides. No shared code, no codegen — drift fails silently (unpaid debt: no CI contract check).</details>
5. 📍 **Q:** Where does the cron fan-out live? <details><summary>A</summary>`apps/workflows/src/cron/checker.ts:245-356`.</details>
6. **Q:** Three trust boundaries and their credentials? <details><summary>A</summary>Session cookie (dashboard/NextAuth), API key `x-openstatus-key` (server), CRON_SECRET (machine hops).</details>
7. ⚡ **Q:** What's wrong with the machine-hop auth? <details><summary>A</summary>One static shared secret across all three hops, no rotation/scoping — leak = whole pipeline. Successor: Cloud Tasks OIDC.</details>
8. **Q:** Why must `packages/services` avoid `node:*` imports? <details><summary>A</summary>Dashboard tRPC runs on Next.js Edge. Evidence: hand-rolled `deepEqual` in `audit/emit.ts:14-24`.</details>

## B. Services layer & auth

9. ⚡ **Q:** Recite the mandatory chain for a new write verb. <details><summary>A</summary>`requireScope("write")` → `Input.parse` → `withTransaction` → workspace-scoped writes (tx handle) → `emitAudit` in same tx.</details>
10. **Q:** One file per ___ under `packages/services/src/<entity>/`? <details><summary>A</summary>Verb (create.ts, update.ts, remove.ts), re-exported via index.ts.</details>
11. **Q:** What does `ServiceContext` carry? <details><summary>A</summary>workspace, actor, optional db/transaction.</details>
12. 📍 **Q:** The canonical verb to show someone? <details><summary>A</summary>`packages/services/src/monitor/create.ts:16-63`.</details>
13. **Q:** The seven-ish actor kinds in the union? <details><summary>A</summary>user, apiKey, mcp, slack, system, webhook, subscriber (`context.ts:13-20`).</details>
14. ⚡ **Q:** `requireScope` is a no-op for which actors, and what's the honest gap? <details><summary>A</summary>No-op for user/system/slack/webhook/subscriber; active for apiKey/mcp. Gap: no member roles — any workspace member can do anything (docblock at require-scope.ts:19).</details>
15. ⚡ **Q:** Why does every service verb re-parse input the router already validated? <details><summary>A</summary>Five+ transports call it; the domain owns domain invariants — validation belongs to whoever owns the invariant.</details>
16. **Q:** Cross-tenant read returns which error, and why not 403? <details><summary>A</summary>NOT_FOUND — existence non-disclosure; a 403 confirms the resource exists. (Known inconsistency: tag validation returns FORBIDDEN.)</details>
17. **Q:** The anti-IDOR machinery, in three items? <details><summary>A</summary>Workspace filter on every query; fetch-or-throw `getXInWorkspace` helpers (`internal.ts:28-47`); ownership-validated join-table ids (L63-84).</details>
18. **Q:** The fourth, subtle auth mode? <details><summary>A</summary>Capability tokens — subscriber verify/unsubscribe URLs carry a random UUID with 7-day expiry; possession = authorization; stripped from audit snapshots.</details>
19. **Q:** Where do ServiceErrors become transport errors? <details><summary>A</summary>At each transport's edge — `toTRPCError` in `packages/api/src/service-adapter.ts:32-87`; TRPCError never appears under packages/services.</details>

## C. Data

20. ⚡ **Q:** Why two data planes? <details><summary>A</summary>~80k rows/day/monitor of probe telemetry would let the firehose threaten the CRUD plane; split means it structurally can't. libSQL = state, Tinybird = telemetry.</details>
21. ⚡ **Q:** The consequence of the split you must design UIs around? <details><summary>A</summary>Cross-plane consistency is eventual and unenforced — incident row can coexist with healthy-looking pings.</details>
22. **Q:** What's `ping_response__v8` and why the suffix? <details><summary>A</summary>The Tinybird table/pipe for probe results; name-suffix versioning because analytics schemas evolve destructively.</details>
23. ⚡ **Q:** The sharpest durability fact about telemetry? <details><summary>A</summary>Checker's Tinybird sends are fire-and-forget — failure is logged, row lost, loss rate unmeasured (`checker.go:255-257`, risk R3).</details>
24. **Q:** Three lifecycle axes on monitor, and the combinatorial worry? <details><summary>A</summary>`active` (should probe), `status` (verdict), `deletedAt` (soft delete) → 8 combined states, not all handled everywhere (ghost-probe drill).</details>
25. 📍 **Q:** The partial index and its lesson? <details><summary>A</summary>`WHERE deleted_at IS NULL` on workspaceId, `monitor.ts:68-72` — index the filtered working set, not the raw column.</details>
26. ⚡ **Q:** The schema decision the authors would take back? <details><summary>A</summary>Nullable `workspace_id` — tenancy by disciplined query filters instead of structural NOT NULL + FK.</details>
27. **Q:** Timestamp convention and its foot-gun? <details><summary>A</summary>Unix seconds via SQLite strftime; JS is milliseconds — `getScheduledTime` (cron/monitor.ts:393-407) mixes conventions.</details>
28. **Q:** The zod-beside-table convention and its maintenance rule? <details><summary>A</summary>Every entity dir pairs table.ts with validation.ts; reads are parsed. Rule: zod sibling ships in the same PR as any schema change or migrations become runtime 500s.</details>
29. ⚡ **Q:** The zero-downtime schema-change litany here? <details><summary>A</summary>Additive-nullable first → backfill → tighten; never rename in one step; forward-only migrations ("rollback" = new forward migration); four apps share the DB so one-version compatibility.</details>
30. **Q:** Where are transactions used and where deliberately not? <details><summary>A</summary>User mutations: always (`withTransaction`, quota+insert+audit atomic). Pipeline `/updateStatus`: separate statements, no tx — idempotency backstops instead.</details>
31. **Q:** The open isolation question? <details><summary>A</summary>Client is `drizzle-orm/libsql/http`; tx isolation over the HTTP driver is uncharacterized (R7) — don't rely on serializable semantics without verifying.</details>

## D. API contracts

32. ⚡ **Q:** Why tRPC for the dashboard instead of dogfooding REST? <details><summary>A</summary>UI-shaped queries would ossify into public contracts; tRPC is versionless because client+server deploy together, so breaking changes are free.</details>
33. **Q:** What makes REST v1 "contract-first"? <details><summary>A</summary>zod schemas inside `createRoute` generate the OpenAPI doc — docs structurally can't describe unenforced behavior (`monitors/post.ts:15-42`).</details>
34. ⚡ **Q:** The zod wire-format wart? <details><summary>A</summary>Default parse strips unknown fields silently — typoed field = 200 + no-op; `.strict()` now = breaking change. Strictness is a v1 decision.</details>
35. 📍 **Q:** Where does the code admit v1 is legacy? <details><summary>A</summary>Comment at `apps/server/src/routes/v1/index.ts:143-147` — inline Drizzle handlers, migration to services in progress.</details>
36. **Q:** Wire vs domain schemas — shared or duplicated, and why? <details><summary>A</summary>Deliberately duplicated (managed decoupling): sharing welds public contract to internal model. Cost: hand-maintained, no drift detection.</details>

## E. Reliability & pipeline

37. ⚡ **Q:** The three idempotency layers and their keys? <details><summary>A</summary>Deterministic Cloud Task names (enqueue); unique notification-trigger rows (send); Redis keys+TTL (email).</details>
38. ⚡ **Q:** The two words you never say apart? <details><summary>A</summary>Retries + idempotency — retries are only safe because of the idempotency layer beneath.</details>
39. **Q:** Evidence vs verdict? <details><summary>A</summary>Per-region `monitor_status` rows = evidence; global `monitor.status` flips only on quorum (`checker/index.ts:216`) — structural anti-flapping.</details>
40. ⚡ **Q:** State the trigger-before-send flaw in one breath. <details><summary>A</summary>Trigger row (dedupe record) inserts before the provider call; crash between them → retry hits its own dedupe → alert suppressed forever → at-most-once alerting in a paging product.</details>
41. ⚡ **Q:** And the fix + accepted new failure mode? <details><summary>A</summary>Send-status on the trigger row + retry sweep (or full outbox); accept rare duplicate sends, deduped provider-side (PagerDuty dedup_key etc.). Customer prefers two pages to zero.</details>
42. **Q:** The R1 race in one sentence? <details><summary>A</summary>Quorum flip is read-check-write across awaits with no tx — concurrent regional reports can both act on pre-flip state → duplicate/missed notifications. Fixes: per-monitor serialization or one tx around flip+incident+trigger.</details>
43. **Q:** Compensation vs rollback here? <details><summary>A</summary>Can't roll back a page or a TB row — incidents auto-resolve with `autoResolved: true` (`checker/index.ts:51-84`), preserving honest history.</details>
44. **Q:** What has no backpressure? <details><summary>A</summary>`/updateStatus` fan-in during a mass outage — unbounded per-request DB work when every monitor×region flips at once.</details>
45. **Q:** Which mutations bypass the audit log? <details><summary>A</summary>Pipeline status flips (`checker/index.ts:227-231`) and free-plan auto-pause (`cron/monitor.ts:292-303`) — system-actor audit rows are the cheap unshipped fix.</details>

## F. Error handling & audit

46. ⚡ **Q:** Fail-closed audit — meaning and accepted cost? <details><summary>A</summary>Audit insert in same tx as mutation; audit failure rolls back the mutation. Cost: availability coupling — audit-table trouble blocks writes. Chosen so absence of a row *means* it didn't happen.</details>
47. **Q:** What gets auto-diffed and what gets stripped in audit entries? <details><summary>A</summary>`changed_fields` auto-diffed from before/after (no-op updates skipped); secrets stripped from snapshots (see integration/remove.ts).</details>
48. **Q:** Fail-open vs fail-closed — the per-surface examples? <details><summary>A</summary>Status-page proxy fails open on schema-invalid page row (public availability wins, but uncounted); scope checks fail closed. Both correct for their surface.</details>
49. **Q:** Where do the audit action names' legal values live? <details><summary>A</summary>Discriminated union in `@openstatus/db/src/schema/audit_logs/validation.ts` — `{entity}.{verb}`.</details>

## G. Testing

50. ⚡ **Q:** Why integration-first at the service layer? <details><summary>A</summary>The killers (cross-tenant leak, quota bypass, missing audit) live in code×SQL×schema interaction — mocked-ORM tests pass while all three burn.</details>
51. ⚡ **Q:** The isolation strategy and its two documented constraints? <details><summary>A</summary>Shared seeded DB + rollback transactions (`withTestTransaction`, helpers.ts:37-50): no nesting (no savepoints over libsql HTTP), and every write must use the `tx` handle.</details>
52. ⚡ **Q:** The classic failure when a test ignores constraint two? <details><summary>A</summary>Committed rows leak into the shared DB and poison *other* suites (quota tests count rows) — flaky by ordering, only in CI. Why `cleanQuotaGatedTables` exists.</details>
53. **Q:** The five standard cases per entity suite? <details><summary>A</summary>Happy path, validation failure, cross-tenant rejection, quota rejection, read-only-scope rejection — plus `expectAuditRow`.</details>
54. **Q:** Why is `turbo test` cache:false? <details><summary>A</summary>Tests aren't hermetic; cached greens would lie. Honest-slow over fast-lying.</details>
55. ⚡ **Q:** The testing confession? <details><summary>A</summary>`apps/workflows` — the highest-risk code — has zero tests; blocker is non-injectable collaborators (clock, tasks, providers). Fix = seam refactor (Lab 04-A).</details>
56. **Q:** The one-line flake prevention worth quoting? <details><summary>A</summary>Audit assertions sort `id DESC` because same-millisecond timestamps tie (helpers.ts:206-211).</details>

## H. Observability

57. ⚡ **Q:** The observability philosophy in one line? <details><summary>A</summary>One wide event per request, enriched by handlers, tail-sampled (keep all errors/5xx/slow, sample the rest 20%), shipped to Axiom.</details>
58. ⚡ **Q:** Tail vs head sampling — why tail? <details><summary>A</summary>Decision made after the response = outcome-aware, so errors are never dropped. Head sampling discards errors at the same rate as successes.</details>
59. **Q:** The two hard observability lessons this repo teaches? <details><summary>A</summary>Absence is the hardest signal (cron silence looks like health — no dead-man's-switch exists); instrument the degradation, not just the failure (the keyId-fallback warn, auth.ts:106-114).</details>
60. **Q:** Where does request correlation die, and what's the weak substitute? <details><summary>A</summary>At the Cloud Tasks boundary — requestId doesn't propagate; only `cronTimestamp` in payloads weakly joins the hops.</details>

## Suggested schedule

Day 1-3: sections A-C. Day 4-6: D-F. Day 7: G-H. Then shuffle-all daily; retire a card after three consecutive clean recalls, resurrect it in week 4. The ⚡ cards double as your final pre-interview sweep — if you can't answer all of those cold in one sitting, you're not done with this repo.
