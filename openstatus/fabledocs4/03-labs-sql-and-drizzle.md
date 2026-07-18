# Labs: SQL & Drizzle

Eight exercises against the real schema (`packages/db/src/schema/`). The earlier docs *describe* the data model; these make you operate it. Run against a local seeded instance (`turso dev` → migrate → seed). For each lab, write the Drizzle version *and* the raw SQL it should produce — the translation in both directions is the skill interviews probe.

**Setup check:** `packages/db` migrated and seeded; you can open a SQL shell against the dev instance and `SELECT count(*) FROM monitor` returns the seed count.

---

## Lab 1 — Tenancy by hand

Write the query behind "list this workspace's live monitors": workspace-scoped, excluding soft-deleted, ordered for the dashboard. Then open `packages/db/src/schema/monitors/monitor.ts:68-72` and explain **why the partial index (`WHERE deleted_at IS NULL` on workspaceId) matches your query exactly** — and what happens to index usage if you write `deletedAt IS NULL OR deletedAt > ?` instead. Verify with `EXPLAIN QUERY PLAN`.

*You've got it when:* you can state the transferable rule — index the filtered working set, not the raw column — and show the query-plan diff that proves it.

## Lab 2 — The IDOR probe

As an attacker with a valid session in workspace B, write the tRPC-equivalent read for a monitor id belonging to workspace A. Trace where it dies: `packages/services/src/monitor/internal.ts:28-47`. Now write the *broken* version of `getMonitorInWorkspace` (drop the workspace filter) and enumerate which tests catch it — is it the cross-tenant service test, the route test, or both? Run the suite and find out.

*You've got it when:* you know exactly which test layer owns this invariant (and why the route layer shouldn't re-test it — ADR-010).

## Lab 3 — The quorum count

Write the SQL `/updateStatus` effectively runs to decide a flip: given a monitorId, count `monitor_status` rows by regional status and compute whether quorum is met. Then write the *interleaving demo*: two sessions in two SQL shells, statements alternated per fire drill 3's trace, showing both reads see pre-flip state. (SQLite's locking may serialize your shells depending on mode — if it does, document what the HTTP driver does differently and why that's exactly risk R7/ADR-005's open question.)

*You've got it when:* you can demonstrate or precisely articulate why demonstrating is hard — both are valid, only one is claimable in an interview, and you know which.

## Lab 4 — The NOT NULL migration, executed

Fabledocs2/3 keep calling this migration "written in my head." Write it for real, on a branch:

1. The violator-finding query (`workspace_id IS NULL`), run against seed data.
2. The backfill decision for violators (delete? assign to a quarantine workspace? — argue it).
3. The Drizzle schema change + `pnpm generate`; **read the generated SQL** and note the SQLite table-rebuild.
4. The zod sibling update in the same change.
5. The deploy-order note: which apps must tolerate what, in which order (four apps share this DB).

*You've got it when:* migration applies cleanly to a seeded DB and the full service test suite passes on the branch.

## Lab 5 — Audit forensics

Using only `audit_log`, answer a support ticket: "everything that happened to monitor X in the last 7 days, by whom." Write the query; then explain what `changed_fields` gives you that `before`/`after` blobs alone don't. Now the gap: pause the monitor via the *free-plan path's* mechanism (manually flip `active` in SQL, simulating `cron/monitor.ts:292-303`) and show the forensic query returns nothing — the ADR-008 asymmetry, demonstrated rather than recited.

*You've got it when:* your query works, and you can write the one-paragraph ticket for system-actor audit rows with the evidence you just produced.

## Lab 6 — Serialized-column tax

`regions` is comma-joined text (`monitor.ts:26`). Write the query "all monitors probing from region `ams`" — feel the `LIKE`/`instr` ugliness and the index impossibility. Then write the counterfactual: the join-table schema that makes it clean, the same query against it, and the migration cost. Finish with a judgment call in writing: at what query frequency does the ADR-005 tradeoff flip from right to wrong?

*You've got it when:* you can argue *both* sides in two sentences each — this is a classic "defend a denormalization" interview seam.

## Lab 7 — Timestamp foot-gun hunt

Schema timestamps are unix **seconds** (`monitor.ts:59-64`); JS `Date.now()` is milliseconds. Write a query that would silently return wrong results if handed a milliseconds value (any `WHERE createdAt > ?` does). Then audit `getScheduledTime` (`apps/workflows/src/cron/monitor.ts:393-407`) — the docs claim it mixes conventions; verify with a concrete trace: pick real input values, compute by hand what it returns, and state whether the email fires at the right time. This is also the setup for the clock-injection lab in 04.

*You've got it when:* your trace has actual numbers in it. "It looks suspicious" is a fail; "with lastAccessedAt = 1752624000, this returns X, which is Y hours off" is a pass either way.

## Lab 8 — Read-your-writes across two planes

No SQL for this one — a design exercise on ADR-004's consequence. A monitor was just created; the dashboard immediately shows its detail page. libSQL has the row; Tinybird has zero pings for it (none run yet — or worse, the first ping was silently lost, risk R3). Write the state machine for the chart component: what are the distinguishable states (no data yet / data lost / genuinely down), which are *indistinguishable today*, and what copy do you show for each? Then check what the real dashboard does.

*You've got it when:* you've internalized that "the UI must tolerate cross-plane disagreement" is a concrete component spec, not a slogan.
