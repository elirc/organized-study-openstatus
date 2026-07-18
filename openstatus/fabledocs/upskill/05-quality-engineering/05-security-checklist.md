# Security Checklist

Map first (what exists, with anchors), then the pre-merge checklist. Label discipline: items marked *investigate* are questions, not findings.

## Authorization & tenancy
- Workspace scoping by construction: [internal.ts:28-47](../../../packages/services/src/monitor/internal.ts#L28-L47); join-row ownership checks [internal.ts:63-84](../../../packages/services/src/monitor/internal.ts#L63-L84). IDOR defense = these + the `'rejects read-only actor'`/cross-tenant test conventions.
- Scopes on API keys, enforced at service ([require-scope.ts](../../../packages/services/src/auth/require-scope.ts#L25-L48)) and v1-route level ([v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147)).
- Known gap (documented): no member-role RBAC for `user` actors ([require-scope.ts:19](../../../packages/services/src/auth/require-scope.ts#L19)).

## Input validation
- zod at wire (tRPC/OpenAPI/queue), domain, and DB-read boundaries — full map in [03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).
- Go side binds JSON to structs ([checker.go:69-75](../../../apps/checker/handlers/checker.go#L69-L75)); assertion JSON unmarshal errors are surfaced, but note ignored marshal error at L98–104.

## SSRF — the product *is* an SSRF engine
Monitors fetch arbitrary user-supplied URLs from checker infrastructure. That's the business. The question is blast-radius: can a monitor target internal endpoints (GCP metadata `169.254.169.254`, private RFC1918 ranges, the workflows service itself)? `isAuthorizedDomain` exists in the cron file ([cron/checker.ts:50-52](../../../apps/workflows/src/cron/checker.ts#L50-L52)) but its enforcement point isn't visible there (*investigate*: what filters monitor URLs at create time — check `CreateMonitorInput` URL validation in `packages/services/src/monitor/schemas.ts` — and what the Go checker refuses to fetch). Interview-grade topic: egress filtering, metadata-endpoint blocking, per-provider network policy.

## XSS
- React escapes by default; risk concentrates in `dangerouslySetInnerHTML` (grep it), user-controlled markdown on status reports (*investigate* renderer), and email templates fed user content (`packages/emails`).

## CSRF
- Dashboard: NextAuth cookie-based sessions — Auth.js CSRF protection on auth routes; tRPC mutations ride same-site cookies (*investigate* SameSite config in `apps/dashboard/src/lib/auth`).
- REST v1: header-key auth (`x-openstatus-key`) — immune to CSRF by construction (no ambient credential). Being able to *say why* is an interview point.

## Injection
- Drizzle parameterizes throughout. The raw `sql` fragments interpolate **column refs and bound params**, not user strings ([proxy.ts:34](../../../apps/status-page/src/proxy.ts#L34), [statusPage.ts:81](../../../packages/api/src/router/statusPage.ts#L81)) — tagged-template binding, safe. Know the difference between `sql\`lower(${page.slug}) = ${value}\`` (bound) and string concat (injection).

## Open redirects
- Status-page login redirects sanitize the target: `sanitizeRedirectParam` ([proxy.ts:10, 77](../../../apps/status-page/src/proxy.ts#L77)) — the exact defense interviewers ask about ("how do you handle ?redirect=").

## Secrets handling
- API keys stored hashed with prefix lookup ([auth.ts:159-183](../../../apps/server/src/libs/middlewares/auth.ts#L159-L183)); subscriber tokens stripped from audit snapshots ([upsert.ts:154-158](../../../packages/services/src/page-subscriber/upsert.ts#L154-L158)); CLAUDE.md mandates stripping credentials from audit snapshots generally.
- Weak spot: one static `CRON_SECRET` across the pipeline (risk R4); page passwords stored in a plain text column ([page.ts:34-38](../../../packages/db/src/schema/pages/page.ts#L34-L38) — "no specific restriction on password" comment; *possible risk*: stored unhashed — verify before claiming, then argue whether page passwords are secrets or access codes).

## Rate limiting & abuse
- Public subscriber signup: plan-gated ([upsert.ts:69-71](../../../packages/services/src/page-subscriber/upsert.ts#L69-L71)); request-rate limiting at this surface *not observed* (*investigate* `apps/dashboard/src/lib/rate-limit` usage and any middleware on status-page tRPC). Email-send abuse is the concrete risk (verification spam).
- SMS quota enforced by count ([alerting.ts:88-121](../../../apps/workflows/src/checker/alerting.ts#L88-L121)).

## Webhooks & third-party
- Outbound notification webhooks (`packages/notifications/webhook`) — *investigate*: URL validation and SSRF posture same as monitors.
- Inbound Slack: signature verification expected in `apps/server/src/routes/slack/` (*verify* before relying).

## Dependency & supply chain
- `pnpm.onlyBuiltDependencies` allowlist ([package.json:34-36](../../../package.json#L34-L36)) limits postinstall scripts — a real, quotable supply-chain control. Lockfile committed; renovate/dependabot not checked.

## Pre-merge security checklist (use on every PR)

- [ ] Every new query filters by `workspaceId` (or justifies why not)
- [ ] New mutation: `requireScope` first line + audit row + both standard negative tests
- [ ] New input crosses a trust boundary exactly once per boundary — zod parse present
- [ ] No secrets/capabilities in logs, audit snapshots, or error messages (generic messages for unknown errors — [service-adapter.ts:79-86](../../../packages/api/src/service-adapter.ts#L79-L86))
- [ ] New public endpoint: rate-limit story + enumeration answer + idempotency
- [ ] Redirect/URL parameters sanitized (open-redirect, SSRF)
- [ ] Public API status codes unchanged (contract), or versioned
- [ ] New env var: added to turbo env allowlist if build-affecting; never `NEXT_PUBLIC_` unless truly public

Drill: run the checklist against fake-PR Kata 5 (rate limiting) and Kata 3 (Discord in tx) — each violates at least two lines. Which?
