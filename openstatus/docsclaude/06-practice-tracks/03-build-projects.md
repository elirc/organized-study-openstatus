# 03 — Build Projects

Five progressively deeper builds. Each is a complete feature, designed to integrate everything from the prior modules. Don't skip the planning step; senior work starts with thinking, not typing.

## Project 1 (1-2 days) — Incident postmortems

**Idea:** Add a "postmortem" feature attached to a resolved status report. Postmortems have a title, a markdown body, an author, and a published flag. Public status pages can show them under each resolved incident.

**Design checklist (do before coding):**
- [ ] DB schema: new table, indexes, FK to `status_report`, cascade?
- [ ] Service verbs: `createPostmortem`, `updatePostmortem`, `publishPostmortem`, `getPostmortem`, `listPostmortems`.
- [ ] Each verb: scope, ownership, transaction, audit.
- [ ] Router: 5 procedures, wire shapes, error mapping.
- [ ] UI: dashboard form, public render with sanitized markdown.
- [ ] Tests: each verb has happy + scope + cross-workspace.
- [ ] Security: markdown rendering (no XSS), workspace isolation.

**Stretch:** allow comments on postmortems (only from workspace members).

## Project 2 (3-5 days) — Scheduled maintenance reminders

**Idea:** When a maintenance window is scheduled, automatically email subscribers 24h before, 1h before, and at start. Use an outbox + cron worker.

**Design checklist:**
- [ ] Cron schedule: how often does the worker check?
- [ ] Idempotency: how do you prevent double-sends?
- [ ] Timezone handling: subscribers in different TZs?
- [ ] Cancellation: if the maintenance is canceled, don't send the future reminders.
- [ ] Templates: how is the email body shaped?
- [ ] Tests: a scheduled maintenance with a faked clock fires reminders correctly.

**Files:** new outbox table, new service for reminders, modifications to maintenance create/update/delete to enqueue/cancel reminders, a workflow handler.

**Stretch:** SMS reminders for plans that support it.

## Project 3 (1 week) — Bulk import of monitors via CSV

**Idea:** Dashboard users can upload a CSV of monitors (name, URL, method, interval, headers, expected status code). The system parses, validates, and creates them in one operation. Errors per row are surfaced.

**Design checklist:**
- [ ] CSV parsing: which library? Streaming or in-memory?
- [ ] File upload: blob storage or in-memory?
- [ ] Validation: per-row Zod parse with full error collection.
- [ ] Transaction: all-or-nothing, or partial success?
- [ ] Limits: max rows per import?
- [ ] UI: progress, errors per row, retry just-the-failed-rows.
- [ ] Tests: 100-row CSV with mixed valid/invalid rows.

**Files:** new tRPC procedure (lambda, because CSV parsing libraries are Node), new service, new UI component (dropzone + result table).

**Stretch:** export the same format. Round-trippable.

## Project 4 (1-2 weeks) — Real-time monitor status via WebSockets

**Idea:** Dashboard users see monitor status updates in real time without polling. Use WebSockets (tRPC subscriptions, or a third-party).

**Design checklist:**
- [ ] Transport: tRPC subscriptions vs native WebSocket vs Server-Sent Events.
- [ ] Server: where does the WebSocket server live? (Edge can't hold connections; Lambda can't either; you need a separate Node service.)
- [ ] Authorization: subscribers can only see their own workspace's events.
- [ ] Scaling: how do events from the checker reach the WebSocket server? (Pub/sub.)
- [ ] Backpressure: what if a client subscribes to 10,000 monitors?
- [ ] Reconnection: client logic to recover from disconnect.
- [ ] Tests: integration test with a real WebSocket.

**Files:** a new app (`apps/realtime` or similar), the checker writes events to a pub/sub, the new app reads + forwards to subscribed clients.

**Stretch:** real-time on the public status page too.

## Project 5 (2-4 weeks) — Multi-region failover monitor checks

**Idea:** A check that "fails" only when it fails from 2+ regions, not 1. Reduces false positives from flaky regional networks. Configurable per monitor: "alert if N of M regions fail."

**Design checklist:**
- [ ] Data model: regional check results (already in `apps/checker`?). How are they aggregated?
- [ ] Aggregation: do you alert per-region or roll up?
- [ ] Per-monitor config: `min_failed_regions: 2`.
- [ ] Migration: existing monitors default to current behavior (1 region = fail).
- [ ] UI: a clear explanation of the new config.
- [ ] Tests: synthetic check results across regions, assert alert fires only at the right threshold.

This is a real product feature most monitoring tools have. Building it forces you to read the checker code (Go), the alerting code, the schema, and the dashboard.

**Files:** modifications across `apps/checker` (Go), `packages/db/src/schema`, the alerting service, the dashboard monitor form.

**Stretch:** "fail only if all regions in a continent fail" — geography-aware aggregation.

## Senior-shaped patterns to apply

Across every project:

- Write a 1-page design doc *before* coding. Get a friend to red-team it.
- Sketch the layers explicitly before opening a file.
- Decide which side effects are in-band vs outbox before writing.
- Decide which validation is at which layer.
- Write the first failing test before writing the first line of production code (for the verb you're most uncertain about).
- Plan your migration / rollback before merging the migration.
- Profile before optimizing.
- Apply the security checklist before merging.

## Self-prompt for AI help

```
I'm building Project N from docs/06-practice-tracks/03-build-projects.md.
I'm at the [planning | implementation | review] stage.
My current design / progress: [paste].
Don't write the code for me. Ask me 5-10 questions that surface what I haven't decided yet.
```
