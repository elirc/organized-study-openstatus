# 02 — Validation at Boundaries

A boundary is a place where untrusted data becomes trusted data. The form is a boundary (the user types whatever). The HTTP handler is a boundary (the client sends whatever). The service is a boundary (callers may be wrong about the inputs). The DB is a boundary (rows might violate invariants).

The senior insight: **you validate at every boundary, even if the previous boundary already did.** The duplication is the point.

## Zod, the validation language

Zod is OpenStatus's validation library at every layer that isn't SQL. A Zod schema is both:

- A *runtime validator* — `schema.parse(input)` throws on invalid, returns typed value on valid.
- A *static type generator* — `z.infer<typeof schema>` is the TypeScript type.

```ts
const Input = z.object({
  title: z.string().min(1).max(256),
  status: z.enum(statusReportStatus),
  date: z.coerce.date(),
});
type Input = z.infer<typeof Input>;
```

One declaration, two outcomes. No drift between "what we check" and "what we type."

## The four boundaries OpenStatus validates at

### 1. Form (`@hookform/resolvers/zod`)

[`apps/dashboard/src/components/forms/status-report/form.tsx:62-77`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L62-L77):

```ts
const schema = z.object({
  status: z.enum(statusReportStatus),
  title: z.string(),
  // ...
});
const form = useForm({ resolver: zodResolver(schema) });
```

Purpose: catch user mistakes before they're submitted. Display field-level errors. Cheap UX, no network round-trip.

### 2. tRPC procedure input (`.input(schema)`)

[`packages/api/src/router/statusReport.ts:24-32`](../../packages/api/src/router/statusReport.ts#L24-L32):

```ts
const createStatusReportTRPCInput = z.object({
  title: z.string(),
  status: CreateStatusReportInput.shape.status,
  pageId: z.number(),
  pageComponents: z.array(z.number()),
  date: z.coerce.date(),
  message: z.string(),
  notifySubscribers: z.boolean().nullish(),
});
```

Purpose: validate the HTTP wire shape. Catches malformed payloads from non-dashboard callers (curl, malicious clients, broken middleware). On failure, tRPC throws `BAD_REQUEST` with the structured Zod error, and the form sees field-level errors via `isTRPCClientError` + `error.data.zodError`.

### 3. Service input (`Schema.parse(...)`)

[`packages/services/src/status-report/create.ts:30`](../../packages/services/src/status-report/create.ts#L30):

```ts
export async function createStatusReport(args) {
  const { ctx } = args;
  requireScope(ctx, "write");
  const input = CreateStatusReportInput.parse(args.input);
  // ...
}
```

Purpose: the service must be safe to call from anywhere — not just from a tRPC router that did its own validation. Background workers, scripts, tests can all call the service. The service can't assume its caller validated.

### 4. DB schema constraints

```ts
status: text("status", { enum: statusReportStatus }).notNull(),
title: text("title", { length: 256 }).notNull(),
```

Purpose: the last line of defense. If something slipped through the upper layers, the DB rejects the insert. Constraints (NOT NULL, CHECK, FK, UNIQUE) are guarantees enforced even if all the application code is wrong.

## "Why validate at every layer?"

Because each layer has different threats and different cost profiles:

- **Form**: instant feedback, no network. Catches typos, missing fields.
- **tRPC**: catches non-form callers. Cheap (CPU-bound).
- **Service**: catches non-tRPC callers (workers, scripts). Cheap (CPU-bound).
- **DB**: catches application bugs. Slightly more expensive (DB round-trip) but absolute.

If you delete tRPC validation "because the form already does it," you've broken the service for any non-form caller. If you delete service validation "because tRPC does it," you've broken it for workers. The duplication isn't redundant — it's *layered defense*.

## One source of truth for shared schemas

The same enum is reused everywhere via import. From [`packages/db/src/schema/status_reports/status_reports.ts:8-13`](../../packages/db/src/schema/status_reports/status_reports.ts#L8-L13):

```ts
export const statusReportStatus = ["investigating", "identified", "monitoring", "resolved"] as const;
```

Imported by:
- The DB schema (Drizzle enum constraint).
- The service schema (`z.enum(statusReportStatus)`).
- The form (same import).
- The router (via `CreateStatusReportInput.shape.status`, which is the service schema).

One change updates all five. There's no drift because no one re-types the values.

**Senior aside:** *If a string literal value appears in three files, hardcoded each time, that's a refactor opportunity. Hoist it to a constant in the schema and re-import. If the team rejects that "for clarity," push back — clarity dies the moment two of those copies disagree.*

## Wire shape vs domain shape (recap)

The wire shape (`pageComponents`) can differ from the domain shape (`pageComponentIds`). Zod schemas at each layer encode the contract for that layer. The router adapts between them.

Don't try to share the wire schema with the service schema; they encode different intents. Reuse *primitives* (the enum), not *aggregate shapes*.

## Custom error classes for the service layer

OpenStatus defines a hierarchy of service errors:

- `ServiceError` (base)
- `NotFoundError` — resource missing or not visible to this actor.
- `ForbiddenError` — actor lacks permission.
- `ConflictError` — state conflict (duplicate slug, already-resolved report).
- `ValidationError` — input fails a domain rule (not a Zod schema rule).
- `LimitExceededError` — quota exhausted.
- `PreconditionFailedError` — invariant violated (e.g., trying to add an update to a deleted report).
- `InternalError` — wrapped unknowns.

The adapter ([`packages/api/src/service-adapter.ts:32-87`](../../packages/api/src/service-adapter.ts#L32-L87)) maps each to a tRPC code:

```ts
case "NOT_FOUND":  throw new TRPCError({ code: "NOT_FOUND", message: err.message });
case "FORBIDDEN":  throw new TRPCError({ code: "FORBIDDEN", message: err.message });
case "LIMIT_EXCEEDED": throw new TRPCError({ code: "TOO_MANY_REQUESTS", message: err.message });
// ...
```

Why custom errors instead of `TRPCError` directly?

- **Transport independence.** A Hono or Connect adapter can map the same errors to its codes. The service doesn't know about tRPC.
- **Test ergonomics.** Tests assert `expect(err).toBeInstanceOf(NotFoundError)`, not `expect(err.code).toBe("NOT_FOUND")`.
- **Centralized mapping.** One file decides "Forbidden → 403"; no router can disagree.

## Error message hygiene

```ts
throw new NotFoundError("page", input.pageId);   // ✅ structured, no PII
throw new Error(`User ${user.email} tried to ...`);   // ❌ leaks PII to clients
```

The service produces structured errors. The adapter maps to tRPC codes with the *message text* (which the client sees). Don't put secrets, internal IDs, or PII in messages — they end up in browser consoles, logs, and screenshots.

The adapter has a final safety net: unknown errors become a generic `"Internal server error"` ([`packages/api/src/service-adapter.ts:78-87`](../../packages/api/src/service-adapter.ts#L78-L87)):

```ts
throw new TRPCError({
  code: "INTERNAL_SERVER_ERROR",
  message: "Internal server error",     // generic to client
  cause: err,                            // original for Sentry/logs
});
```

Generic to the user, full detail to the logger. That's the right tradeoff.

## Validation vs sanitization

Validation says "is this allowed?" Sanitization says "if this is dirty, clean it up."

OpenStatus validates; it rarely sanitizes. That's the right default — silently changing user input is surprising. Exceptions:

- `z.string().trim()` — collapse leading/trailing whitespace.
- `z.coerce.date()` — accept string or number, return Date.
- `slug.toLowerCase()` in the public statusPage query — case-insensitive slug lookup.

Each of those is a deliberate normalization, not a blind cleanup.

## When validation should be looser

Some inputs are user content where we *want* leniency:

- A status report message — markdown, links, mentions. Validate length, not shape.
- A monitor name — any non-empty string the user picks.
- A page slug — must match a regex, lowercase, unique. *Tight* validation.

Match the strictness to the field's role. Slugs go in URLs and DB indexes — strict. Message bodies go in UI as rendered markdown — lenient (but still escaped for XSS).

## Pitfall checklist

1. **Validating only at the form.** Backend trusts the client. The client lies.
2. **Validating only at the router.** Background worker calls the service with garbage. Crash.
3. **Validating only at the service.** Form errors become server-side `BAD_REQUEST`s with no field guidance.
4. **Wire schema imported by the service.** Coupling reversed. Service shouldn't know what the transport accepts.
5. **`TRPCError` thrown from a service.** Breaks transport independence.
6. **Error message leaking PII or secrets.** `err.message` in `cause` is fine; in user-facing `message` it's not.
7. **`z.any()` in a schema.** You opted out of validation; might as well use a string.
8. **Schemas duplicated as TypeScript interfaces *and* Zod schemas.** Pick one as the source; derive the other with `z.infer`.
9. **`.parse()` in a hot loop.** Zod isn't free. Validate once at the boundary, then trust.
10. **Sanitizing instead of rejecting.** A slug with spaces shouldn't auto-fix to dashes; it should error so the user picks something they understand.

## Drills

- Open one router procedure and list every Zod schema involved (form, router, service, DB constraint). Confirm they all reference the same enum source.
- Find one place where service input is validated. Try passing in a malformed object from a test. Verify the error surfaces correctly.
- Add a new `ServiceError` subclass for a real scenario in the repo (e.g., `MonitorPausedError`). Add the adapter mapping. Use it.
- Look for one place where validation is *missing* at a layer — push back if you find one.
- Take a `TRPCError` thrown from inside a procedure and refactor it to throw a `ServiceError` instead. Pass the test.
