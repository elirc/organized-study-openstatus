# Senior Build Projects

Six projects, 2 days–4 weeks. Each is realistic — a maintainer could plausibly accept it (or its RFC). Do at most two; do them completely: design doc, tests, rollout plan, and a written retro.

---

## Project S1: Notification outbox (at-least-once alerting)
Problem: alert delivery is at-most-once — a crash between trigger insert and provider send suppresses a page forever ([alerting.ts:131-176](../../../apps/workflows/src/checker/alerting.ts#L131-L176); analysis in [03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)). ~1–2 weeks.
Product value: "we never miss a page" is the product's core promise.
Design checklist: extend `notification_trigger` with `status (pending|sent|failed)`, `attempts`, `lastError`; write pending in the same breath as today; dispatcher loop (interval or queue-driven) sends + marks; provider-side idempotency keys where supported (PagerDuty dedup_key).
Architecture decisions: dispatcher location (workflows app — already owns providers); poison-message policy (max attempts → failed + Sentry); ordering (per monitor, not global).
Likely files: `packages/db/src/schema/notifications/*` (migration), [alerting.ts](../../../apps/workflows/src/checker/alerting.ts), new `apps/workflows/src/checker/dispatcher.ts`, tests.
Migration plan: additive columns with defaults → dual-mode flag (send-inline vs dispatch) → flip → remove inline.
Test plan: kill-between-steps simulation; duplicate-send tolerance test; provider-failure retry test.
Security/perf: no new inputs; dispatcher batch size bounded.
Rollout/rollback: env-flag per mode; rollback = flip flag.
Open questions: do recovery notifications need the same guarantee? (Weaker — argue it.)
Stretch: dead-letter UI in dashboard.
Interview story potential: a complete at-least-once redesign with staged rollout — a senior system-design story you *implemented*.

## Project S2: Member roles (workspace RBAC)
Problem: any workspace member can do anything; `requireScope` no-ops for users, documented as a separate project ([require-scope.ts:19](../../../packages/services/src/auth/require-scope.ts#L19)). ~3–4 weeks.
Design checklist: roles (owner/admin/member/viewer) on `users_to_workspaces`; extend `Actor` user variant with role (compiler surfaces every consumer — [context.ts:13-20](../../../packages/services/src/context.ts#L13-L20)); map verbs → minimum role; enforcement inside `requireScope` (one choke point, keeping first-line placement).
Migration: backfill everyone as owner (no behavior change) → invite flow sets roles → enforce behind flag per workspace.
Test plan: role matrix test per entity (extend the `'rejects read-only actor'` convention into `'rejects viewer'`); invitation flow tests.
Likely files: schema users/, services auth/, invitation service, dashboard settings UI, [trpc.ts](../../../packages/api/src/trpc.ts#L148-L200) context enrichment.
Open questions: does the audit log record the role at time-of-action? (It should — snapshot semantics.)
Interview story: authorization system design + zero-downtime enforcement rollout.

## Project S3: Level-triggered reconciliation for monitor state
Problem: the pipeline is edge-triggered; missed edges strand state (scenario 2 in [debugging](../05-quality-engineering/03-systematic-debugging.md)); quorum math races (R1). ~1–2 weeks.
Design: periodic reconciler recomputes `monitor.status` from `monitor_status` evidence (and optionally recent TB data), corrects drift, emits audit on correction, counts corrections as a *health metric* (drift rate ≈ pipeline correctness KPI).
Architecture decision: reconciler is authoritative or advisory? Start advisory (log-only), promote.
Likely files: new workflows cron route; reuse quorum function — which forces extracting it pure from [checker/index.ts:216-348](../../../apps/workflows/src/checker/index.ts#L216-L348) (a refactor with its own value).
Test plan: property-style: for random evidence tables, reconciler verdict == quorum spec.
Interview story: "edge-triggered vs level-triggered" — the distributed-systems concept most mid-level candidates can't ground; you'll have built both.

## Project S4: Self-host queue abstraction
Problem: the pipeline hard-codes GCP Cloud Tasks ([cron/checker.ts:56-63](../../../apps/workflows/src/cron/checker.ts#L56-L63), [update.go](../../../apps/checker/checker/update.go#L29-L93)), complicating true self-hosting (a stated project value — README/Docker/Coolify). ~2–4 weeks.
Design checklist: `TaskQueue` interface (enqueue with name/scheduleTime/target); adapters: CloudTasks (current), Redis-backed (BullMQ-style semantics but keep it dependency-light), in-process for dev; task-name dedupe must be preserved per adapter (the invariant!).
Migration: adapter extraction with CloudTasks as default = zero behavior change; SELF_HOST uses Redis adapter.
Test plan: adapter-contract test suite (same tests run against every adapter — the technique to name in interviews).
Open questions: Go checker's UpdateStatus also enqueues — does the interface live in both languages, or does Go call a workflows HTTP endpoint instead (simplification!)?
Interview story: hexagonal ports/adapters with an invariant-preserving contract-test suite.

## Project S5: Status-page data-layer authorization (defense in depth, R6)
Problem: page access control lives in middleware; the public tRPC data path's protection depends on output-schema stripping (*investigate* status — [statusPage.ts:64-113](../../../packages/api/src/router/statusPage.ts#L64-L113), [proxy.ts:64-86](../../../apps/status-page/src/proxy.ts#L64-L86)). ~1 week including the investigation.
Phase 1 (the real deliverable): **prove or disprove the leak** — attempt direct `statusPage.get` calls for a password-protected fixture page; document findings.
Phase 2: if leaky, thread the access decision into the procedure (signed short-TTL access token minted by the proxy, verified in the router); if not leaky, add the regression test that keeps it that way + a doc note.
Test plan: fixture pages per accessType × direct-call matrix.
Interview story: a genuine security investigation with evidence-first methodology — outstanding for security-conscious interviews. (Also: responsible-disclosure etiquette if it's real and you're contributing upstream.)

## Project S6: Probe result buffering in the checker (R3)
Problem: Tinybird send failures lose the product's core data, logged only ([checker.go:255-257](../../../apps/checker/handlers/checker.go#L255-L257)). ~1–2 weeks, Go.
Design checklist: bounded in-memory ring buffer + retry flusher; overflow policy (drop-oldest + counter — never block probes); flush on shutdown (SIGTERM handling); loss-rate metric exposed.
Architecture decision: why *not* a disk WAL (checker instances are ephemeral/regional; bounded loss acceptable — argue the SLA).
Test plan: Go tests with a failing fake TB server; overflow behavior; shutdown flush.
Interview story: backpressure + bounded-buffer design with explicit loss budgets — rare, concrete reliability story.

---

Selection guide: S1 or S3 for backend-reliability roles; S2 for product-platform roles; S5 for security interest; S4/S6 to demonstrate range (infra + Go). Whatever you pick, the **design doc is half the value** — write it as if the maintainers will read it ([07/02 has the RFC template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)).
