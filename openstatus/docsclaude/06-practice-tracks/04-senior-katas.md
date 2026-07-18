# 04 — Senior Katas

These are open-ended exercises with no single right answer. The point is to *argue* — pick a position, defend it, anticipate the counter, refine your position. This is what senior engineers do in design reviews and PR discussions.

For each kata, write a 1-2 page response. Then find a friend (or your future self) to challenge your position.

## Kata 1 — Move notifications server-side

**Current state:** subscriber notifications are triggered from the dashboard's `onSuccess` after a status report mutation succeeds ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L33-L51)). There's a TODO to move it server-side.

**The kata:** propose and defend a server-side design. Compare three options:

1. **In-band synchronous**: service creates report, then sends notification before returning.
2. **In-band async**: service starts notification with Next.js `after()`; user gets a fast response but errors are silent.
3. **Outbox**: service writes a notification intent inside the transaction; a worker drains.

For each: pros, cons, blast radius of failure, debugging story. Pick one. Defend.

## Kata 2 — Split the public status page query

**Current state:** [`packages/api/src/router/statusPage.ts:65-400`](../../packages/api/src/router/statusPage.ts#L65-L400) loads everything in one call — page, workspace, reports, updates, maintenances, components, monitors, incidents, groups. The comment in the file flags performance as a concern.

**The kata:** propose a split. The two natural axes are:

- **By concern**: shell (page metadata), events (open + last), trackers (components + groups + monitor data), feed (reports + maintenances + updates).
- **By interactivity**: embed mode skips the feed; toggled sections lazy-load.

Sketch the API: how many calls, in what order, with what caching? What does the client orchestrate? What stays server-side?

**Tradeoffs to address:** waterfall vs over-fetching, cacheability, code complexity, mobile bandwidth.

## Kata 3 — Branded IDs

**Current state:** workspace IDs, page IDs, monitor IDs are all `number`. Structural typing allows any number to be passed where any ID is expected.

**The kata:** argue for or against introducing branded types (`WorkspaceId`, `PageId`, ...).

- Pros: compiler catches cross-entity ID confusion; IDOR-shaped bugs caught earlier.
- Cons: requires casts everywhere IDs come from "outside" (DB rows, form inputs, URL params); branding ceremony multiplies across the codebase.

Where would branding pay off most? Where would it be net-negative? Is there a partial approach (brand only IDs that travel across workspace boundaries)?

## Kata 4 — Member roles

**Current state:** `requireScope(ctx, "write")` is a no-op for `user` actors ([`packages/services/src/auth/require-scope.ts:25-30`](../../packages/services/src/auth/require-scope.ts#L25-L30)). Member-role enforcement for dashboard users is "a separate project."

**The kata:** design that project. Address:

- Role hierarchy: owner / admin / member / viewer? Capabilities per role?
- Where roles live: DB schema.
- Where enforcement lives: in `requireScope`, or higher up?
- Migration: what role do existing members default to?
- UI: role badge, role-change flow, "you don't have permission" messages.
- Test impact: every write verb's tests gain a "rejects member without write role" case.

What's the smallest first cut that ships value? What can be deferred?

## Kata 5 — Type-safe DB at the boundary

**Current state:** Drizzle row types come from the schema. Service functions return inferred types. UI components consume those types. But: the boundary between "DB row" and "domain object" is fuzzy — the row often *is* the domain object.

**The kata:** argue for or against introducing a clean separation: DB rows are storage shapes; the service maps to domain objects; the UI consumes domain objects.

Pros: looser DB/service coupling; can rename DB columns without UI changes; can compose fields from multiple rows into a single domain object.

Cons: more code; more places to keep in sync; double conversion costs.

When is the separation worth it? (Hint: when the row shape and the consumer shape genuinely differ — see `enrichReportsBatch` which already does this.)

## Kata 6 — Edge vs Lambda boundary

**Current state:** hard-coded list of router names in [`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77) decides where each procedure runs.

**The kata:** propose a less-fragile mechanism. Options:

- **Per-procedure metadata**: `procedure.meta({ runtime: "lambda" }).query(...)`. The client reads metadata to decide.
- **Convention**: any router named `*Router` ending in some suffix runs on lambda.
- **Build-time analysis**: detect which procedures import Node-only modules; route them automatically.

Each has costs (type-system gymnastics, build-time complexity, runtime overhead). Pick one. Build a strawman API.

## Kata 7 — Audit log queryability

**Current state:** audit log is a single table with `action`, `entity_type`, `entity_id`, `before`, `after`, `metadata`, `actor`, `workspace_id`.

**The kata:** design a search UX for it. "Show me everything user X did this week." "Show me every change to monitor 42." "Show me every delete in the last 30 days."

Address:
- Indexes needed for these queries.
- UI: filters, search, time range, export.
- Compliance: tamper-evident logs? Append-only enforced?
- Retention: how long do we keep audit rows? Archive vs delete?

## Kata 8 — A failed PR review

**The kata:** find an old PR (in this repo or one of your own). Apply the [security checklist](../05-quality-engineering/05-security-checklist.md). Write a complete PR review as if it were submitted today. Note:

- What you'd ask for changes on.
- What you'd block-merge on.
- What you'd accept as a follow-up.
- What you'd write as commentary even if you didn't block.

Then read the actual review (if available). Compare your review with the actual one. What did you miss? What did they miss?

## Kata 9 — A 30-minute architecture interview

**The kata:** pretend you're being interviewed. The question: "design a system to monitor 1 million HTTP endpoints, alerting on downtime in < 1 minute, with a public status page."

Constraints:
- Budget: small team, AWS or similar.
- No build-time-static — checks happen continuously.
- Multi-region for accuracy.
- Subscribers (web push, email, Slack, PagerDuty).
- Multi-tenant.

Outline:
- Data model: monitors, checks, incidents.
- Check workers: how distributed? How scheduled? How aggregated?
- Storage: hot path (recent) vs cold path (history).
- Alerting: dedup, threshold, escalation.
- Public status: how derived, how cached.
- Subscribers: how dispatched at scale.

You have 30 minutes. Sketch on paper. Then read OpenStatus's actual architecture and diff.

## Kata 10 — Rewrite advocacy

**The kata:** pretend a junior on your team says "this codebase is a mess; we should rewrite it in Remix + Convex." Write the response.

Two versions:
1. **Why they're wrong.** What does the existing architecture get right? What invariants are encoded that a rewrite would have to rediscover? What's the cost of a rewrite vs incremental improvement?
2. **Why they might be partially right.** What real friction motivates the desire? What incremental improvements address that friction? When would a rewrite actually be the right call?

The mark of a senior is being able to write both versions credibly.

## Self-prompt for AI help

```
I'm working on Senior Kata N from docs/06-practice-tracks/04-senior-katas.md.
Here's my current take: [paste].
Don't tell me which is "right." Argue back against my position with three concrete challenges.
```
