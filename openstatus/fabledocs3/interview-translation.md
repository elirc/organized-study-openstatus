# Interview translation

Each ADR compressed into a ~90-second spoken answer — first person, past tense, tradeoffs volunteered before the interviewer asks — followed by the follow-up question you should expect, with a one-line note on how to meet it. Practice these out loud; the written version always sounds fine and the spoken version always runs long. Cut until it doesn't.

---

## ADR-001 — Monorepo, apps vs packages

**90 seconds:** "The system was one product with five very different deployables — two Next.js frontends, two Bun/Hono services, and a Go probe binary — all sharing a schema and business rules. We used a pnpm-plus-Turborepo monorepo with a hard rule: `apps` are deployables and own nothing reusable, `packages` own every shared contract — the Drizzle schema, the services layer, the notification providers. The payoff was atomic cross-cutting changes: a schema change, its service logic, the router, and the UI land in one PR, which at our team size beat publishing internal packages through a registry. The tradeoff I'd flag is that the monorepo *looks* like one system but deploys as independent services sharing one database — so deploy-order discipline, additive-only migrations, one-version compatibility, lived in convention, not structure. And our Go app sat in the repo but outside the JS toolchain, so nothing in CI validated the JSON contract between TypeScript and Go. That's the gap I'd close first today."

**Likely follow-up:** "How did you keep independently-deployed apps from breaking each other through the shared database?" — Answer with the migration litany: additive-nullable first, backfill, tighten later; never rename in one step; and admit it's discipline, not tooling.

---

## ADR-002 — Services layer, one verb per file

**90 seconds:** "The same operation — say, create a monitor with quota, tenancy, and audit — was reachable from five transports: tRPC, REST, MCP, Slack, and jobs. Early on logic lived in routers, and the second transport forced a choice between duplication and coupling; we actually had duplicated quota logic in the REST handlers as evidence. So we extracted a services layer: one file per verb, uniform signature taking a context with workspace, actor, and optional transaction. The verb owns the whole invariant chain — scope check, input parse, transaction, workspace-scoped writes, audit emission — and routers became thin adapters. One extra constraint made it interesting: our dashboard ran tRPC on the Edge runtime, so the services package had to stay free of Node built-ins — we literally hand-rolled a deepEqual rather than import node:util. The cost was a long migration tail: legacy REST handlers coexisted with the new layer, so for a while our 'every mutation is audited' invariant was only true for migrated code. I learned to state invariants precisely because of that."

**Likely follow-up:** "How do you *enforce* that routers stay thin — what stops the next engineer writing DB calls in a route?" — Review convention plus a grep-able smell (`@openstatus/db` imports under routes), and honestly: nothing structural; lint rules on import boundaries would be the upgrade.

---

## ADR-003 — Go probes + Cloud Tasks

**90 seconds:** "Probing is the product: tight control over timeouts, redirects, and DNS from twenty-eight regions, at volumes far above the CRUD traffic. We wrote the prober in Go — a static binary you can drop on any cheap region in seconds, with a per-request HTTP client honoring per-monitor timeouts — and kept the TypeScript-to-Go boundary data-only: one JSON payload, typed independently on both sides, no shared code, no codegen. For fan-out we used GCP Cloud Tasks rather than self-hosting a broker, and the detail I always highlight is deterministic task names — the task name encodes monitor, region, and tick, so a double-enqueued cron tick is a no-op. That's idempotency by construction, free from the queue. Two tradeoffs still bother me: the JSON contract can drift silently because nothing validates it in CI, and the machine hops authenticated with a single shared secret — no rotation, no per-edge scoping. Cloud Tasks supports OIDC natively; that's the successor decision we never shipped."

**Likely follow-up:** "You picked a managed queue in an open-source product — how did self-hosters run it?" — A config-driven alternative path existed (Coolify/compose docs); concede it's the honest cost of buying operational wins in the critical path.

---

## ADR-004 — Two data planes

**90 seconds:** "We split storage by workload. Transactional state — workspaces, monitors, incidents — lived in libSQL with full ACID and tenancy. Probe telemetry never touched SQL: a thirty-second monitor across twenty-eight regions is about eighty thousand rows a day *per monitor*, so the Go checker appended results straight to Tinybird — managed ClickHouse with a typed pipe layer — and every read went through versioned pipes. The design guarantee was that the firehose could never take down the plane users click on. The two costs I'd volunteer: first, cross-plane consistency is eventual and unenforced — an incident row can exist while the telemetry still shows healthy pings, and the UI has to be written to tolerate that. Second, and this is the sharp one: telemetry writes were fire-and-forget — a failed send was logged and the row was simply lost. Which means the uptime data, the product itself, had the weakest durability in the system, and we never measured the loss rate. If I owned it again, a loss-rate metric ships in week one."

**Likely follow-up:** "Why not just partition a ping table in Postgres and roll it up?" — Viable early; the argument is failure-domain isolation plus analytics query shape, and the honest answer includes "at small scale, Postgres would have been fine."

---

## ADR-005 — libSQL + schema conventions

**90 seconds:** "The OLTP side was read-mostly and tenant-scoped, and our dashboard API ran at the edge, so we chose libSQL — Turso — over HTTP: near-zero ops, replicas near users. We made three schema-style bets. First, zod-beside-table: every entity directory pairs the Drizzle table with a zod schema, and reads are parsed back into rich types — so drift fails loudly at parse time. Second, cheap serialization: regions as comma-joined text, assertions as JSON-in-text, validated in the app layer — that traded normalization for shipping speed and I'd do it again. Third bet I would *not* repeat: the tenancy column was nullable. Every query compensated with an explicit workspace filter and fetch-or-throw helpers, and it held — but structural beats disciplined, and the NOT NULL migration stayed hypothetical partly because SQLite turns constraint-tightening into table rebuilds. Also worth confessing: unix-seconds timestamps next to JavaScript milliseconds is a foot-gun we did get bitten by in scheduling math."

**Likely follow-up:** "Walk me through making that tenancy column NOT NULL in production with zero downtime." — Query for violators, backfill, deploy code tolerating both, add the constraint (table rebuild), all additive-first — and name the multi-app deploy-order constraint.

---

## ADR-006 — Split API contracts

**90 seconds:** "We ran two API surfaces with explicitly different stability promises. The dashboard used tRPC — versionless by design, because client and server deploy together, so breaking changes are free and the UI gets dozens of UI-shaped queries that should never become public contracts. External integrators got REST v1, built contract-first: every route declares its request and response as zod schemas, and the OpenAPI docs are *generated from the validators* — the docs structurally cannot describe behavior the code doesn't enforce. I rejected dogfooding our own public API from the dashboard, which sounds virtuous but ossifies UI queries into permanent contracts. The cost was two implementations per capability, which is exactly what the services layer existed to contain. The wart I always mention: zod's default parse silently strips unknown fields, so a typoed field name returns 200 and does nothing — and fixing that later is itself a breaking change. Strictness is a v1 decision; we missed it."

**Likely follow-up:** "How would you version REST v1 when a breaking change becomes unavoidable?" — URL-versioned `/v2` beside `/v1`, both thin adapters over the same service verbs — the services layer is what makes dual-running cheap.

---

## ADR-007 — Validation ownership + error translation

**90 seconds:** "Our rule was: validation belongs to whoever owns the invariant. Transports validated wire shape, but every service verb re-parsed its input at the top — which every new reviewer flagged as redundant, and it wasn't: the verb is called from five transports plus background jobs and can't assume any of them validated. Errors flowed one direction — services threw domain errors like NotFound or Forbidden, and each transport translated at its own edge; the tRPC error type never appeared in domain code, enforced by package boundaries. And every caller — user session, API key, MCP, Slack, machine — was normalized into one Actor union, so authorization and audit attribution branched on *who you are*, never on which route called. Two honest gaps I'd volunteer: cross-tenant reads returned 404 to avoid existence leaks but one legacy path returned 403 — an inconsistency, not a policy; and key-level scopes were enforced while human member roles were explicitly deferred — any workspace member could do anything. I'd rather state that precisely than have an interviewer discover it."

**Likely follow-up:** "Why 404 instead of 403 for cross-tenant access?" — Existence non-disclosure: a 403 confirms the resource exists in someone else's tenant; 404 gives an enumerating attacker nothing.

---

## ADR-008 — Fail-closed audit

**90 seconds:** "Audit logging is usually best-effort — emit after commit, swallow failures — which gives you a log with holes exactly when things go wrong. We inverted it: every service mutation emits its audit row *inside the same transaction*, and it's fail-closed — if the audit insert fails, the business mutation rolls back. The reasoning: our customers used the trail for their own compliance, so the log had to be trustworthy in the negative — no row means it didn't happen. Conventions rode along: entity-dot-verb action names in a discriminated union, before/after snapshots with auto-diffed changed fields, secrets stripped before emitting, and a test helper so every entity suite asserts the audit side-effect. I accepted an availability coupling — an audit-table failure blocks writes — deliberately; I'd rather have that outage than silent gaps. The asymmetry I'd confess: the background pipeline bypassed the services layer for throughput, so its mutations were unaudited — a support engineer asking why a monitor got auto-paused found nothing. The actor union already had a 'system' kind; that fix was cheap and just unprioritized."

**Likely follow-up:** "Wasn't fail-closed audit risky for availability? When is best-effort actually correct?" — Distinguish compliance-grade trails (fail-closed) from product analytics (best-effort, post-response) — the repo runs both patterns and the contrast is the answer.

---

## ADR-009 — Idempotency instead of an outbox

**90 seconds:** "The alerting pipeline crossed process and vendor boundaries, so transactions couldn't span it. Consistency rested on deterministic identity at three layers: queue task names encode monitor-region-tick, so double-enqueue is a no-op; notification sends dedupe on a unique trigger row; lifecycle emails dedupe on Redis keys with TTL. Status transitions separated evidence from verdict — per-region rows accumulate evidence, the global status flips only on quorum, so anti-flapping was structural rather than a tuned debounce. Retries at every layer were safe only *because* of that idempotency — I never say one word without the other. Now the flaw, and I'd rather raise it than defend it: the trigger row was inserted *before* the provider call. Crash between insert and send, and the retry hits its own dedupe row — the alert is suppressed forever. That's at-most-once alerting in a paging product; the customer would rather get two pages than zero. The fix is an outbox or send-status on the trigger row with a retry sweep. We chose insert-first fearing duplicates; that was optimizing for the wrong embarrassment."

**Likely follow-up:** "Design the outbox migration — what changes and what new failure mode do you accept?" — Intent row joins the incident transaction, a dispatcher delivers and marks status, and you accept duplicate sends — hence provider-side dedupe keys.

---

## ADR-010 — Integration-first testing

**90 seconds:** "The failures that kill a multi-tenant product — cross-tenant leaks, quota bypasses, missing audit rows — live in the interaction between code, real SQL, and the schema, so mocking the ORM tests nothing real. We tested invariants at the service layer against a real seeded database, with isolation via rollback transactions: each test runs inside a wrapper that rolls everything back, with two constraints documented in the helper itself — no nesting, because the HTTP driver has no savepoints, and every write must use the transaction handle. That second constraint bit us: a test using the default client passed locally while committing rows that made *other* suites flake — the cleanup helper's comments literally record that incident. Each layer owned distinct assertions — services own invariants, route tests own contract shape, Go tests own probe semantics — and we disabled test caching because non-hermetic tests plus cached greens equals lying CI. The confession: our highest-risk code, the alerting pipeline, had no tests at all, because its collaborators — queue, telemetry, wall-clock — weren't injectable. The refactor to fix that was known and just never done."

**Likely follow-up:** "How would you test the concurrent status-flip race you say exists?" — Inject the collaborators, then fire two concurrent regional reports at `/updateStatus` and assert exactly one trigger row — the test exists as a design before the fix does.

---

## Using these

Three rules for delivery. **Volunteer the wart** — every answer above contains one confessed flaw; that's what separates "describes clean architecture" from "describes real architecture." **Name a number or a file-level detail** once per answer (eighty thousand rows/day, deterministic task names, deepEqual-by-hand) — specificity is credibility. **Land on the lesson**, not the feature: "optimizing for the wrong embarrassment," "structural beats disciplined," "state invariants precisely." Those sentences are what interviewers write down.
