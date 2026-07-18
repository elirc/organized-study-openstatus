# 05 — Security Checklist

A senior engineer's security work isn't pen-testing — it's *preventing* the bugs that lead to incidents. This is a 20-item checklist to run before merging any non-trivial PR.

## The big four

### 1. Authorization

- Does every protected endpoint use `protectedProcedure` (or equivalent)?
- Does every write verb start with `requireScope(ctx, "write")` ([`packages/services/src/auth/require-scope.ts`](../../packages/services/src/auth/require-scope.ts))?
- Does every DB query filter by `workspaceId`?
- For any input ID, does the service verify it belongs to the active workspace before acting on it?
- Are there tests for cross-workspace rejection?

This is the highest-impact category. IDOR (Insecure Direct Object Reference) bugs leak data across tenants.

### 2. Validation

- Is every input validated at the service boundary with `Schema.parse(...)`?
- Are error messages free of secrets, internal IDs, PII?
- Does the wire schema reject extra unknown fields (Zod's default — verify it's not `.passthrough()`)?
- Are file uploads validated for MIME type, size, and content?
- Are URLs validated for SSRF before fetching ([`packages/utils/src/ssrf.test.ts`](../../packages/utils/src/ssrf.test.ts) shows the existing pattern)?

### 3. Auditing

- Does every mutation emit an audit row inside the same transaction?
- Are sensitive fields (credentials, tokens, raw keys) stripped from `before`/`after` snapshots?
- Are action names following `entity.verb` convention?
- Are the action variants added to the discriminated union in [`packages/db/src/schema/audit_logs/validation.ts`](../../packages/db/src/schema/audit_logs/validation.ts)?

### 4. Secrets

- Are secrets read from `process.env` server-side only (no `NEXT_PUBLIC_*`)?
- Are env vars validated at boot (via `@t3-oss/env-nextjs` in `env.mjs`)?
- Are credentials never logged?
- Are credentials never returned in API responses?
- Are credentials never stored in audit log snapshots?

## SSRF (Server-Side Request Forgery)

If your code accepts a URL from user input and then fetches it server-side, you have SSRF risk. An attacker can give you `http://169.254.169.254/...` (AWS metadata), `http://localhost:5432/...` (internal services), `file:///etc/passwd`.

OpenStatus uses a HTTP monitor service that fetches user-provided URLs. The SSRF guard is in [`packages/utils/src/ssrf.test.ts`](../../packages/utils/src/ssrf.test.ts) — read it, understand it, use it.

Rules:

- Reject private IP ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `127.0.0.0/8`).
- Reject loopback / link-local hostnames.
- Allow only `http:` / `https:` schemes.
- DNS-resolve before fetching; check the resolved IPs (a hostname can resolve to a private IP).
- Use a separate egress proxy if you can — defense in depth.

## XSS (Cross-Site Scripting)

React escapes by default — `<div>{userInput}</div>` is safe. The unsafe patterns:

- `dangerouslySetInnerHTML={{ __html: userInput }}` — explicit opt-out. Search for it in any PR.
- `href={userInput}` where userInput is `javascript:alert(1)` — sanitize URLs.
- User-provided markdown rendered as HTML — use a sanitizing renderer (`react-markdown` + `rehype-sanitize`).
- Server-side template injection — rare in Next.js, but possible if you use raw HTML strings.

## CSRF (Cross-Site Request Forgery)

NextAuth handles session CSRF with double-submit tokens. tRPC over POST with `SameSite=Lax` cookies is safe by default. To break it:

- Don't change cookies to `SameSite=None` without `Secure` and a clear need.
- Don't expose mutating GET endpoints.
- Don't rely on Origin headers for security if you can avoid it.

## SQL injection

Drizzle's query builder parameterizes inputs by default. You can't inject through `eq(col, userInput)`. You *can* inject through `sql\`SELECT * FROM x WHERE y = ${userInput}\`` — that's raw interpolation. Use `sql.placeholder()` or `sql\`... ${sql.param(userInput)}\``.

Search any PR for raw `sql\`...${...}\`` and verify the interpolations are safe (either parameterized or hard-coded constants).

## Open redirects

If your code redirects to a user-provided URL (`?redirectTo=...`), an attacker can make your login page redirect to a phishing site. Validate the redirect target against an allowlist of internal paths.

## Session management

- Are session cookies `HttpOnly`, `Secure`, `SameSite=Lax`?
- Is the session token rotated on privilege change?
- Is there a logout flow that invalidates server-side state?
- Is the session timeout reasonable?

OpenStatus uses NextAuth; the defaults are sane. Review provider configurations for `allowDangerousEmailAccountLinking` — currently `true` ([`apps/dashboard/src/lib/auth/providers.ts`](../../apps/dashboard/src/lib/auth/providers.ts)). That's a deliberate UX tradeoff; document it.

## Rate limiting

Without rate limiting, brute force, scraping, and denial-of-service are trivial. Layers:

- Edge proxy (Vercel, Cloudflare).
- Middleware (per-IP, per-cookie).
- Per-user / per-workspace (quota at the service layer).

OpenStatus's per-workspace quotas are encoded in plan limits ([`packages/services/src/auth/`](../../packages/services/src/auth/) and elsewhere). Per-IP rate limiting at the middleware tier is *not* in the OSS repo; verify your deployment adds it.

## Webhook security

If you accept webhooks, verify signatures. Stripe, GitHub, and others sign their payloads with a shared secret. The handler should:

- Verify the signature *before* parsing the payload.
- Reject on invalid signature.
- Be idempotent (the same event might be delivered twice).

OpenStatus's Stripe webhook handling lives in `packages/api/src/router/stripe/`; review it for signature verification.

## Email security

If your app sends email:

- SPF, DKIM, DMARC configured.
- Links in transactional emails are signed or time-limited.
- Don't include passwords or secrets.
- Verify the recipient was authorized (don't email the wrong workspace).

If your app receives email (replies, signup flows), watch for header injection, attachment-based exploits.

## File uploads

OpenStatus has `blobRouter` (for file uploads). Checklist:

- Allowed MIME types enforced.
- Max file size enforced.
- Filenames sanitized (no `../`).
- Stored on a separate origin (so a malicious upload can't run JS on your domain).
- Virus scanning if relevant.

## Dependencies

- `pnpm audit` regularly. Update high/critical fixes promptly.
- Lock files committed.
- Don't run `npm install` on production servers — use frozen lockfile installs.
- Be wary of new transitive dependencies in PRs — supply chain attacks are a real risk.

## Environment / deployment

- Production env vars set via Vercel/whichever provider, not committed to the repo.
- `.env*` in `.gitignore`. Verify on every PR.
- Branch protection on `main` (no direct push, requires PR + reviews).
- CI runs tests before deploy.
- Rollback path documented.

## OWASP top 10 mapped

Quick reference, mapping OWASP Top 10 (2021) to this stack:

1. **Broken Access Control** → IDOR / cross-workspace checks. Covered.
2. **Cryptographic Failures** → don't roll your own; use NextAuth, libsql with TLS.
3. **Injection** → Drizzle parameterizes, React escapes. Watch raw `sql\`\``.
4. **Insecure Design** → service layer, audit log, fail-closed.
5. **Security Misconfiguration** → env validation, secure cookies.
6. **Vulnerable Components** → `pnpm audit`, lockfile.
7. **Authentication Failures** → NextAuth, session timeouts.
8. **Software & Data Integrity** → signed webhooks, supply chain.
9. **Logging & Monitoring** → audit log + production logs + alerting.
10. **SSRF** → covered above.

## Pre-merge security checklist (the actual checklist)

Before you click Merge:

- [ ] Every protected endpoint uses `protectedProcedure`.
- [ ] Every write service starts with `requireScope(ctx, "write")`.
- [ ] Every DB query filters by `workspaceId`.
- [ ] Every input ID is verified to belong to the active workspace.
- [ ] Every mutation emits an audit row inside the transaction.
- [ ] Audit snapshots don't include secrets.
- [ ] Error messages don't leak PII or internal details.
- [ ] No new `dangerouslySetInnerHTML`.
- [ ] No raw `sql\`\`` with unparameterized interpolation.
- [ ] No new dependencies without an audit pass.
- [ ] No `NEXT_PUBLIC_` env vars with secret content.
- [ ] User-provided URLs go through SSRF protection if fetched server-side.
- [ ] Webhook handlers verify signatures.
- [ ] File uploads enforce type and size.
- [ ] New cookies are `HttpOnly`, `Secure`, `SameSite=Lax`.
- [ ] Tests cover the cross-workspace rejection case.
- [ ] Tests cover the read-only-actor rejection case (for writes).
- [ ] No `console.log` of secrets / tokens.
- [ ] No commented-out auth checks "for testing."
- [ ] Threat model updated if attack surface changed.

If you can run this checklist in your head in 60 seconds for any PR, you're senior-shaped on security.

## Drills

- Read [`packages/utils/src/ssrf.test.ts`](../../packages/utils/src/ssrf.test.ts). For each test, identify the attack it prevents.
- Find every `dangerouslySetInnerHTML` in the codebase. Justify each one.
- Find every raw `sql\`\`` usage. Verify the interpolations are safe.
- Run `pnpm audit`. Triage the findings: which need immediate action?
- Walk through a hypothetical IDOR attack: a malicious user, what they'd try, what stops them at each layer.
- Pick one PR (recent, real). Apply the checklist above. How many items pass? Document the gaps.
