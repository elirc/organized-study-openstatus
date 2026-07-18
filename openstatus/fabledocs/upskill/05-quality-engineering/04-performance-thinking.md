# Performance Thinking

## Rule zero: measure first

This repo hands you the measurement layer: every request already logs `duration_ms` in a wide event, and slow requests (>2s) are *always* kept by tail sampling ([server index.ts:90-100](../../../apps/server/src/index.ts#L90-L100), [workflows index.ts:42-62](../../../apps/workflows/src/index.ts#L42-L62)). Before any optimization, the question is "what does Axiom say the p99 is, and on which route?" Saying that sentence in an interview beats naming ten optimizations.

## The performance domains here, with likely hotspots

### Server/DB
- **Remaining N+1s:** the notification loop runs per-notification workspace + SMS-count queries ([alerting.ts:70-121](../../../apps/workflows/src/checker/alerting.ts#L70-L121)); the fixed contrast is right above it — the batched status fetch labeled "N+1 fix" ([cron/checker.ts:127-151](../../../apps/workflows/src/cron/checker.ts#L127-L151)). Pattern for finding them: any `await db.` inside a `for` loop.
- **Heavy relational fetch on the hottest public read:** `statusPage.get` loads reports + updates + maintenances + components + monitors + incidents in one nested query, with a TODO admitting it ([statusPage.ts:50, 80-113](../../../packages/api/src/router/statusPage.ts#L50)). Mitigations that exist: Tinybird pipe caching (5-min public cache, [client.ts:11-13](../../../packages/tinybird/src/client.ts#L11-L13)).
- **Middleware DB lookup per status-page request** ([proxy.ts:30-36](../../../apps/status-page/src/proxy.ts#L30-L36)) — a candidate for short-TTL caching keyed by host; measure hit rate first.
- **Indexes:** the partial index matches the dominant monitor query ([monitor.ts:68-72](../../../packages/db/src/schema/monitors/monitor.ts#L68-L72)). Audit-log and incident query paths — verify indexes exist before assuming (drill below).

### Network/client
- Dashboard fires monitors + tags + two metrics queries on mount ([client.tsx:45-78](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L45-L78)) — the "HMM: why do we need two queries?" comment (L63) is an open invitation; batching by type client-side vs one pipe parameterized by type is a nice measured-PR (ticket M2).
- `staleTime: 60s` ([client.tsx:14-23](../../../apps/dashboard/src/lib/trpc/client.tsx#L14-L23)) prevents refetch storms — know why it's there.

### Pipeline/throughput
- Fan-out arithmetic: tasks per tick = active monitors × regions (×2 for 30s) — [cron/checker.ts:153-183](../../../apps/workflows/src/cron/checker.ts#L153-L183). Concurrency 100 caps enqueue parallelism. This is the axis that scales with growth; the 10× question in [08/04 Step 6](../08-interview-prep/04-system-design-from-this-repo.md).
- Go checker sends one Tinybird event per probe ([checker.go:255](../../../apps/checker/handlers/checker.go#L255)) — batching is the classic next step at volume.

### Bundle/render
- Unmemoized derived arrays each render ([client.tsx:51-60](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L51-L60)) — harmless at n=50, measurable at n=5000; profile before touching.

## How to find each class (transferable)

| Smell | Detector |
| --- | --- |
| N+1 | `await db.` inside loops (grep); query logs with repeated shapes |
| Serial async | waterfall in devtools/network; independent awaits in sequence |
| Expensive renders | React Profiler commit counts, not intuition |
| Missing index | `EXPLAIN QUERY PLAN` in sqlite/drizzle studio on the real query |
| Unbounded query | endpoints returning arrays with no LIMIT — check `listMonitors` and audit list for pagination (*verify*) |
| Oversized bundle | `next build` output, `@next/bundle-analyzer` |

## Drill

Pick the status-page read path. Write the measurement plan (which three numbers you'd collect, from where) *before* proposing any change; then propose the smallest change and its predicted effect. Basic: names caching. Solid: measurement plan first, one bounded change. Strong: predicts the number, states the invalidation story (when a report is published, how fast must the page update?), and notes the 5-min public cache already bounds staleness.

## Interview angle

"How would you speed up a slow page?" — the graded answer is a *procedure* (measure → localize to domain → cheapest fix → verify), illustrated with the statusPage.get example and its existing TODO. Cross-links: [08/02 Q8](../08-interview-prep/02-frontend-framework-questions.md), [08/03 Q4](../08-interview-prep/03-api-and-data-modeling-questions.md).
