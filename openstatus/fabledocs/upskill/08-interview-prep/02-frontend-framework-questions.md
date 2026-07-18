# Frontend Framework Question Cards

Twelve cards on Next.js/React/TanStack Query, anchored to real components.

## Q1: Server vs client components — how do you decide?
Round: frontend
Repo anchor: [monitors/(list)](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)) — `page.tsx` (server) + `client.tsx` (client, `"use client"` at [client.tsx:1](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L1)).
Junior: "client components for interactivity."
Mid adds: default server; the boundary is drawn where hooks/state/browser APIs begin; server components keep data-fetching + heavy deps out of the bundle; props crossing the boundary must serialize.
Senior includes: the boundary is an *architecture* decision — this repo pushes the whole interactive table into one client file and keeps params parsing server-side (`search-params.ts` shared), minimizing the serialized surface. Also names the tradeoff: client-fetching via tRPC (chosen here) vs RSC-fetching (fewer round trips, less interactivity).
Follow-ups: what happens if a server component imports `useState` (build error); streaming/Suspense.
Drill: draw the server/client boundary for the monitors page from memory.

## Q2: Where does state live in a well-organized page?
Repo anchor: [client.tsx:43-49](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L43-L49) — three stores: server cache (`useQuery`), URL (`useQueryStates`/nuqs), local (`useState` sorting/filters).
Junior: "useState, maybe Redux."
Mid adds: classify by *owner and lifetime*: server data → query cache; shareable view state → URL (survives refresh, shareable links); ephemeral UI → local state. Global client stores are a last resort.
Senior includes: the sync problem between stores — this page syncs URL→table once on mount with a suppressed lint and a TODO ([client.tsx:80-90](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L80-L90)); explains the fix (derive, don't sync) and why sync-effects are a smell.
Drill: for each piece of state on the page, say what survives refresh.

## Q3: How does TanStack Query change how you write React?
Repo anchor: [client.tsx:44-46](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L44-L46); client setup with `staleTime: 60s` and per-SSR-request client ([trpc/client.tsx:14-37](../../../apps/dashboard/src/lib/trpc/client.tsx#L14-L37)).
Junior: "it fetches and caches."
Mid adds: server state ≠ client state; keys structure the cache; staleTime vs gcTime; mutations + invalidation replace manual refetch; dependent queries via `enabled` ([client.tsx:64-78](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L64-L78)).
Senior includes: why the browser query client must be a singleton but the server one per-request (the comment at client.tsx:26-35 explains the suspense re-render trap); cache invalidation strategy as a *design review* item.
Drill: narrate what happens on mount, at 30s, and at 90s with staleTime 60s.

## Q4: Loading, empty, and error states — why do they get confused?
Repo anchor: `if (!monitors) return null` at [client.tsx:92](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L92) (tickets J4/J15).
Junior: shows a spinner.
Mid adds: `undefined` conflates loading and error; `[]` is a *successful empty* needing its own UX; enumerate all four (loading/error/empty/data).
Senior includes: skeletons matching layout to prevent CLS; error boundaries vs inline errors; retry affordances tied to query retry policy.
Drill: rewrite that line's surrounding logic aloud with all four states.

## Q5: What are stale closures and where do they bite?
Repo anchor: the mount-only effect reading `searchParams` ([client.tsx:82-90](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L82-L90)) — deps suppressed, so later param changes never re-run it.
Junior: recites the setInterval example.
Mid adds: closures capture render-time values; the deps array is the *invalidation list* for the closure; suppressing the linter converts a compiler-checked contract into a comment.
Senior includes: the structural fixes — derive during render, use refs for latest-value escape hatches, or event handlers instead of effects; and *when the suppression is a legitimate, documented compromise* (this one has a TODO naming the better design).
Drill: explain why moving that logic into the nuqs parser defaults removes the effect entirely.

## Q6: How does tRPC + React Query give end-to-end type safety, and where does it stop?
Repo anchor: `createTRPCContext<AppRouter>()` ([trpc/client.tsx:11-12](../../../apps/dashboard/src/lib/trpc/client.tsx#L11-L12)); `AppRouter` from [root.ts](../../../packages/api/src/root.ts#L5-L6).
Junior: "the client knows the server types."
Mid adds: types flow from router *inference*, no codegen; `queryOptions()` bridges into query keys; superjson preserves Dates.
Senior includes: stops at runtime — server deploys ahead of cached clients still break, hence input parsing server-side regardless; and it's first-party-only (versionless contract), which is why the public API is OpenAPI instead ([08/03 Q1](03-api-and-data-modeling-questions.md)).
Drill: explain what happens when the server renames a field while a user has the old bundle open.

## Q7: Optimistic updates — when and how?
Repo anchor: conceptual — check whether monitor delete does optimistic removal (find the mutation in `components/data-table/monitors/`; investigating is the drill).
Junior: "update the UI before the server responds."
Mid adds: onMutate snapshot → rollback on error → invalidate on settle; only for high-confidence, low-conflict mutations.
Senior includes: idempotency + server truth reconciliation; why you *wouldn't* do it for quota-gated creates (server may reject on limits — [create.ts:25-28](../../../packages/services/src/monitor/create.ts#L25-L28)).
Drill: whiteboard the three-callback pattern.

## Q8: A page renders slowly. Walk me through your approach.
Repo anchor: [04-performance-thinking.md](../05-quality-engineering/04-performance-thinking.md) procedure; unmemoized derived arrays ([client.tsx:51-60](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L51-L60)) as the "measure before memo" example.
Junior: "add useMemo/React.memo."
Mid adds: measure first (Profiler commits, network waterfall); classify render-bound vs fetch-bound; fix the dominant term.
Senior includes: the four queries on mount and whether they waterfall ([client.tsx:44-78](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L44-L78)); memoization *costs* (dependency bookkeeping, stale bugs); when the answer is pagination, not memo.
Drill: 90-second aloud answer using this page as the example.

## Q9: How do you handle forms with validation in this stack?
Repo anchor: monitor create form under `apps/dashboard/src/components/forms/` (open it — deliberate homework); shared zod schemas mean client validation mirrors server ([monitor.ts:50-61](../../../packages/api/src/router/monitor.ts#L50-L61)).
Mid answer shape: schema-driven (zod + react-hook-form resolver typical), server errors mapped back to fields via tRPC's zodError formatter ([trpc.ts:109-118](../../../packages/api/src/trpc.ts#L109-L118)).
Senior includes: client validation is UX, server validation is the contract; never trust the mirror to stay in sync — the server parse is authoritative.

## Q10: Accessibility — what do you actually check?
Repo anchor: shadcn/radix components in `packages/ui` give focus management/ARIA for primitives.
Mid answer: semantic elements, keyboard paths, focus visible, labels; leverage a11y-baked component libraries rather than hand-rolling.
Senior includes: a11y of *data tables* (header scope, sort announcements) and status colors (the status page communicates via color — needs text equivalents); a11y as review checklist item, not sprint task.

## Q11: How do multiple Next.js apps share UI safely?
Repo anchor: `packages/ui` consumed by dashboard/status-page/web; theme via `packages/theme-store`.
Mid answer: workspace package, single design system, Tailwind config sharing.
Senior includes: versionless internal sharing means a UI change fans out to three apps — CI must build all consumers (turbo graph does); contrast with published-package versioning tradeoffs.

## Q12: When would you *not* use React Query?
Round: frontend judgment.
Mid answer: truly local state; websocket streams with custom stores; forms.
Senior includes: server components already fetching (double-fetch smell); tiny apps where the cache layer outweighs its value; and knowing what you lose (dedupe, retry, devtools) when you hand-roll.

---

Practice protocol: shuffle, answer aloud with anchors, 2 minutes each. Record Q1, Q3, Q8 — those three carry most frontend loops.
