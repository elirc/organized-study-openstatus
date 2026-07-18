# Domain Glossary

Product nouns, their meaning here, and where they live. The confusing near-synonyms are flagged — misusing them is how juniors reveal they haven't read the code.

| Term | Meaning in OpenStatus | Code home |
| --- | --- | --- |
| **Workspace** | The tenant. Owns everything; carries `plan` + `limits` (JSON quotas) | [workspace.ts](../../../packages/db/src/schema/workspaces/workspace.ts#L10-L35) |
| **Monitor** | A configured probe: URL/host, method, regions, periodicity, assertions, timeout, `degradedAfter` | [monitor.ts](../../../packages/db/src/schema/monitors/monitor.ts#L13-L73) |
| **Check / probe** | One execution of a monitor from one region at one timestamp. Result rows live in Tinybird (`ping_response__v8`), not the SQL DB | [checker.go](../../../apps/checker/handlers/checker.go#L50) |
| **Monitor status** (per region) | Row per (monitor, region) holding last known status; upserted on every change | [alerting.ts](../../../apps/workflows/src/checker/alerting.ts#L276-L302) |
| **Monitor status** (aggregate) | `monitor.status` column: `active` / `degraded` / `error` — flipped only when a quorum of regions agree | [checker/index.ts](../../../apps/workflows/src/checker/index.ts#L216-L348) |
| **Incident** | Automatic outage record: opened when a monitor flips to error, resolved (possibly auto) on recovery. **Not** written by humans | [checker/index.ts](../../../apps/workflows/src/checker/index.ts#L301-L327) |
| **Status report** | Human-written communication on a status page ("We are investigating…"), with updates over time | `packages/db/src/schema/status_reports/`, service verbs in `packages/services/src/status-report/` |
| **Maintenance** | Scheduled window; monitors linked via page components are *excluded from checks* during it | [cron/checker.ts](../../../apps/workflows/src/cron/checker.ts#L77-L108) |
| **Page** | A status page: slug, custom domain, access control (`public` / password / SSO domains / IP ranges), locale | [page.ts](../../../packages/db/src/schema/pages/page.ts#L14-L50) |
| **Page component** | A line item displayed on a page, optionally bound to a monitor; groupable | `packages/db/src/schema/page_components/` |
| **Subscriber** | An email/webhook that gets page updates; self-signup uses a verification **token** (a capability, stripped from audit snapshots) | [upsert.ts](../../../packages/services/src/page-subscriber/upsert.ts#L52-L239) |
| **Notification (channel)** | A configured alert destination (Slack, PagerDuty, SMS…) attached to monitors | `packages/db/src/schema/notifications/`, providers in `packages/notifications/*` |
| **Notification trigger** | Dedupe record: one row per (monitor, notification, cronTimestamp) — inserting it twice fails, which is the idempotency mechanism | [alerting.ts](../../../apps/workflows/src/checker/alerting.ts#L257-L274) |
| **Assertion** | A pass/fail rule on a probe response (status code, header, body). Defined in TS ([packages/assertions](../../../packages/assertions)), evaluated in Go ([checker.go](../../../apps/checker/handlers/checker.go#L334-L372)) |
| **Region** | One of 28 probe origins across fly/koyeb/railway; dictionary in `packages/regions`, routing headers per provider at [cron/checker.ts](../../../apps/workflows/src/cron/checker.ts#L325-L335) |
| **Periodicity** | How often a monitor runs (30s…). Maps 1:1 to a **Cloud Tasks queue name** | [cron/checker.ts](../../../apps/workflows/src/cron/checker.ts#L69-L73) |
| **Private location** | Self-hosted checker agent for internal endpoints (8.5MB Docker image per README) | `apps/private-location`, `packages/db/src/schema/private_locations/` |
| **Actor** | Who is performing a service call: `user`, `apiKey`, `mcp`, `slack`, `system`, `webhook`, `subscriber` | [context.ts](../../../packages/services/src/context.ts#L13-L20) |
| **Scope** | API-key permission: `read` / `write` (`*` internal-only) | [require-scope.ts](../../../packages/services/src/auth/require-scope.ts#L25-L48), [auth.ts](../../../apps/server/src/libs/middlewares/auth.ts#L229-L238) |
| **Audit log** | Workspace-visible mutation trail written in-transaction by services | [emit.ts](../../../packages/services/src/audit/emit.ts#L102-L149) |
| **Plan / limits** | Billing tier and its quota object (`monitors`, `max-regions`, `periodicity`, `sms-limit`…) read straight off the workspace row | [post.ts](../../../apps/server/src/routes/v1/monitors/post.ts#L59-L87) |

## Confusables

- **Incident vs status report** — incident = machine-detected outage record; status report = human communication. A page can show both.
- **`monitor.active` vs `monitor.status`** — `active` is *"should we probe it"* (a switch, e.g. flipped off by the free-plan pause workflow at [cron/monitor.ts](../../../apps/workflows/src/cron/monitor.ts#L280-L323)); `status` is *"what did probing find"*. Also distinct from `monitor.deletedAt` (soft delete) — three lifecycle axes on one row.
- **Audit log (product) vs `checkerAudit.publishAuditLog`** — the services audit log is the SQL `audit_log` table; the workflows pipeline publishes operational events through a different Tinybird-backed audit ([checker/index.ts](../../../apps/workflows/src/checker/index.ts#L170-L211)). Same word, two systems.
- **Page vs status page vs dashboard page** — "page" the entity is the public status page; the dashboard's Next.js pages are unrelated.
- **`workspace-slug` cookie vs page slug** — the first selects your active tenant in the dashboard ([trpc.ts](../../../packages/api/src/trpc.ts#L171-L181)); the second is a public URL.

## Drill

Close this file. Write the one-line definition of: incident, status report, notification trigger, actor, periodicity. Then find each term's *schema* file without search — navigate the directory tree from memory. Self-grade: Basic = 3/5 definitions right; Solid = 5/5 plus schema locations; Strong = you can also say which **service verbs** exist for each entity (`ls packages/services/src/<entity>/`).
