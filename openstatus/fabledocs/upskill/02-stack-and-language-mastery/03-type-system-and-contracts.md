# Type System & Contracts

## Inference first, annotation at boundaries

Mental model: let TS infer inside function bodies; annotate exported signatures and boundary shapes. The repo's style matches: service verbs annotate `args` and `Promise<Monitor>` ([create.ts:16-19](../../../packages/services/src/monitor/create.ts#L16-L19)) while internals infer.

## Discriminated unions as domain modeling

The `Actor` union ([context.ts:13-20](../../../packages/services/src/context.ts#L13-L20)) is the repo's best type: seven caller kinds, each with exactly its own fields. Every consumer is an exhaustive `switch` ([context.ts:60-97](../../../packages/services/src/context.ts#L60-L97)) — extension is compiler-driven. Compare: `notifType: "alert" | "recovery" | "degraded"` ([alerting.ts:30](../../../apps/workflows/src/checker/alerting.ts#L30)) drives a switch whose branches are near-duplicates — a union *without* the payload variance that justified it (ticket J13 collapses it).

Judgment rule: union when variants carry different data or different invariants; parameterize when they differ only by a string.

## Deriving types from values

When a library doesn't export a type, derive it: `DrizzleTx = Parameters<Parameters<DrizzleClient["transaction"]>[0]>[0]` ([context.ts:7-11](../../../packages/services/src/context.ts#L7-L11)). Same family: `z.infer<typeof selectMonitorSchema>` used as `TaskInput["row"]` ([cron/checker.ts:43-48](../../../apps/workflows/src/cron/checker.ts#L43-L48)). The schema/value is the source of truth; types are projections. Cost: upgrades break loudly — that's a feature, note it aloud in interviews.

## Runtime contracts: zod at trust boundaries

Static types stop at the process edge. The repo parses at every boundary crossing:
- HTTP in: OpenAPI request validation ([post.ts:21-30](../../../apps/server/src/routes/v1/monitors/post.ts#L21-L30)), tRPC `.input()` ([monitor.ts:80](../../../packages/api/src/router/monitor.ts#L80)).
- Queue in: [checker/index.ts:96-99](../../../apps/workflows/src/checker/index.ts#L96-L99).
- **DB out** (the one juniors skip): `selectMonitorSchema.parse(row)` ([create.ts:61](../../../packages/services/src/monitor/create.ts#L61)) — because columns like `regions` hold serialized text that the schema transforms into typed arrays, and because a DB migration can drift from the code's assumptions.
- Analytics out: Tinybird pipe `data` schemas ([client.ts:74-97](../../../packages/tinybird/src/client.ts#L74-L97)).
- Response out: `.output(...)` doubling as a field-stripping security control ([statusPage.ts:76](../../../packages/api/src/router/statusPage.ts#L76)).

Duplicated schemas are sometimes correct: the tRPC input schema is kept separate from the service input schema *to preserve the wire contract exactly* during the services migration — the comment says so ([monitor.ts:39](../../../packages/api/src/router/monitor.ts#L39)). Decoupling wire contract from domain contract is a senior-level distinction.

## Cast discipline

The repo's policy ([CLAUDE.md](../../../CLAUDE.md), "Type Cast Discipline") is quotable in interviews: unavoidable casts get **a named helper + a comment stating the runtime guarantee**; a scattered `as never` is a missing helper. Live example of a commented safe cast: audit row snapshot casts at [emit.ts:142-144](../../../packages/services/src/audit/emit.ts#L142-L144) ride on the zod parse three lines earlier.

## Pitfall checklist

- [ ] `interface` merging surprises vs `type` (repo uses `type` for domain shapes)
- [ ] Optional field (`db?: DB`) defaults hidden in helpers — know that `withTransaction` falls back to the global client ([context.ts:44](../../../packages/services/src/context.ts#L44)); in tests, forgetting `db: tx` silently writes to the real DB
- [ ] Enum-ish DB columns typed by `{ enum: [...] }` ([monitor.ts:17-23](../../../packages/db/src/schema/monitors/monitor.ts#L17-L23)) — SQLite doesn't enforce them; the type is a promise the DB doesn't keep (zod on read closes the gap)
- [ ] `String(...)` coercions at id boundaries (`entityId: String(parsed.entityId)`, [emit.ts:141](../../../packages/services/src/audit/emit.ts#L141)) — number/string id ambiguity is a classic bug source

## Drills

1. Write the type of `withTransaction` from memory, generics included; check against [context.ts:40-47](../../../packages/services/src/context.ts#L40-L47).
2. Find the transform in `selectMonitorSchema` that turns `regions` text into an array (open `packages/db/src/schema/monitors/validation.ts` — deliberately unanchored; go verify).
3. Take the `PingData` Go struct ([checker.go:26-45](../../../apps/checker/handlers/checker.go#L26-L45)) and write the zod schema you'd use if this crossed into TS. Compare field-by-field with what Tinybird's ingestion expects.

## Interview angle

- Discriminated unions & exhaustiveness → [08/01 Q2](../08-interview-prep/01-js-ts-node-deep-dive.md).
- `unknown` vs `any`, cast policy → Q3.
- zod/parse-don't-validate → Q4.
- Generics from values → Q14.
