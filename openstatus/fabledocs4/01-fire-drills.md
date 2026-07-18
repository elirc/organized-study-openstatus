# Fire drills

Six incident simulations. Each gives you the page as you'd receive it on call, then staged hints (each hint you open drops your self-grade one level), then resolution criteria. Work with the repo open and write a running incident log: hypothesis → evidence → next check. That log *is* the deliverable — senior on-call is a note-taking discipline, not a knowledge test.

**Grading:** **S** = root cause + fix + prevention with no hints. **A** = root cause with ≤1 hint. **B** = root cause with all hints. Below that, reread the linked docs and rerun the drill in a week.

---

## Drill 1 — "The customer's site was down and nobody got paged"

**The page:** Support escalation. Customer's monitor shows a 40-minute incident window in the dashboard (incident row exists, later auto-resolved). Their PagerDuty has nothing. Their notification channel is configured and passes the test-send button. They are furious and asking for an RCA by EOD.

**Your job:** Explain how an incident row can exist with zero provider deliveries, using only what the code guarantees. Then answer the harder question: *can you prove which of the possible causes it was, from the data that exists today?*

<details><summary>Hint 1</summary>Read the ordering in `apps/workflows/src/checker/alerting.ts:131-176` very carefully. What is written durably, and when, relative to the provider call?</details>
<details><summary>Hint 2</summary>The process runs on Fly. Processes get OOM-killed and restarted. What does the retry see when it re-enters the dedupe check?</details>
<details><summary>Hint 3</summary>Also check the catch-all around the trigger insert (risk R8 in `../fabledocs/upskill/09-reference/risk-register.md`) — there are two distinct suppression paths, and the logs can't distinguish either from "no channels configured."</details>

**Resolution criteria:** You named trigger-before-send at-most-once semantics; you identified that a trigger row with no provider-success record is the fingerprint but that the schema doesn't record send status, so the RCA is necessarily "consistent with, cannot prove"; your prevention section proposes send-status-on-trigger or an outbox (which is Capstone 02). Bonus S+: you drafted the customer-facing RCA paragraph honestly without the word "impossible."

---

## Drill 2 — "Everything is fine, which is the problem"

**The page:** A crontab-style external check you set up yourself (because you read `../fabledocs2/06-observability.md`) reports: zero new Tinybird rows for 25 minutes, all regions. Dashboard charts are flatlining. No Sentry errors. No error-outcome wide events. Workflows `/ping` returns 200.

**Your job:** Build the differential diagnosis: at least four distinct failure points between "cron should tick" and "row lands in Tinybird," ordered by what you'd check first, with the exact evidence source for each (log query, endpoint, console).

<details><summary>Hint 1</summary>Walk the chain: Fly cron trigger → `apps/workflows/src/cron/checker.ts` fan-out → Cloud Tasks queue → Go checker per region → Tinybird ingest. Absence at the end doesn't localize the break.</details>
<details><summary>Hint 2</summary>Which links have *positive* evidence when healthy (fan-out summary log with success/failed counts, `cron/checker.ts:225-242`) and which are silent-when-dead (the cron trigger itself, the TB ingest — `checker.go:255-257` logs failures only in the checker's own logs)?</details>
<details><summary>Hint 3</summary>Cloud Tasks has its own console: queue depth rising = checker not consuming; queue empty + no fan-out log = cron never fired; fan-out log present + queue draining + no TB rows = checker-side or TB-side.</details>

**Resolution criteria:** Correct localization order (cheapest/most-discriminating check first), correct evidence source per link, and — the senior mark — you noted that this drill's premise (an external absence-detector) doesn't exist in the repo and wrote the ticket for it: what heartbeats, what watches, what threshold.

---

## Drill 3 — "The monitor is flapping but the regions disagree"

**The page:** A customer reports getting a down-then-recovered notification pair twice within three minutes. Dashboard shows the monitor's global status flipped error→active→error→active. Per-region history: two regions consistently failing, the rest healthy, no changes in that window.

**Your job:** Reconcile "quorum-based flipping is structurally anti-flapping" (ADR-009) with observed flapping. Where exactly can the read-check-write in `/updateStatus` interleave to produce this, and what would you log/query to confirm?

<details><summary>Hint 1</summary>Two regional reports arrive concurrently. Each runs upsert → read all `monitor_status` rows → count → conditional update (`apps/workflows/src/checker/index.ts:122-316`). No transaction spans those statements.</details>
<details><summary>Hint 2</summary>Interleaving A-reads/B-reads/A-writes/B-writes: both can act on pre-flip state. Now add a third report from a recovering region in the same window.</details>
<details><summary>Hint 3</summary>Confirmation evidence: wide events for `/updateStatus` carry the affected regions and timestamps (`checker/index.ts:153-162`) — overlapping request windows for the same monitorId are your smoking gun.</details>

**Resolution criteria:** A concrete interleaving trace (two columns, statement by statement) producing the double flip; the confirming log query; and both fix candidates with their costs (per-monitor serialization via queue key vs transaction around flip+incident+trigger — and why the HTTP driver makes the second one need verification first, ADR-005).

---

## Drill 4 — "The quota tests fail, but only on Thursdays"

**The page:** CI is red on `packages/services` quota tests, intermittently, and a teammate swears their PR "doesn't touch anything near quotas." Reruns sometimes pass. Local runs pass for everyone individually.

**Your job:** Find the class of bug before finding the instance. Then find the instance: what specifically do you audit in the teammate's diff?

<details><summary>Hint 1</summary>What isolation do these tests assume, and what single mistake silently breaks it? Read the constraints documented inside `packages/services/test/helpers.ts:37-50`.</details>
<details><summary>Hint 2</summary>A test that writes with the default `db` client instead of the `tx` handle commits real rows. Quota tests *count rows*. Whose rows are they counting now?</details>
<details><summary>Hint 3</summary>Why "only on Thursdays": test *ordering* and accumulation — the failure appears in whichever suite runs after the leak, not the suite containing it. `cleanQuotaGatedTables` (helpers.ts:52-88) exists because of exactly this; its comments tell the story.</details>

**Resolution criteria:** You named ambient coupling / rollback-bypass as the class; your diff audit greps the new tests for `defaultDb`/`db:` usage inside `withTestTransaction` bodies; prevention includes the review-checklist line and (S grade) a proposal for making it structural — e.g. a lint rule or a helper that shadows `db` inside the wrapper.

---

## Drill 5 — "A password-protected status page is leaking"

**The page:** A security researcher emails: they can read a password-protected status page's monitor data without the password. They won't share the method yet. You have two hours to find it yourself before the disclosure call.

**Your job:** Audit the gate. Where is access control for status pages enforced, what does it protect, and what *doesn't* it protect?

<details><summary>Hint 1</summary>The gate is `composePageAction` in `apps/status-page/src/proxy.ts:64-86` — middleware. Middleware protects *routes*. What else serves data for a page? (Risk R6.)</details>
<details><summary>Hint 2</summary>Enumerate every read path that returns page/monitor data: the page HTML route, but also any tRPC/REST endpoints the page's client components call, RSS/atom feeds, badge/status endpoints, subscriber flows. Which of them re-check the password gate?</details>
<details><summary>Hint 3</summary>Also test the fail-open path: what happens in the proxy when the page row fails schema validation? (It fails open, silently — `../fabledocs2/05-error-handling.md`.) Could an attacker *cause* that?</details>

**Resolution criteria:** A written inventory of every data-serving path for a status page with gated/ungated verdicts (verify in code — this is one of the docs' deliberately unresolved questions; your finding may be "gated after all," which is a valid drill outcome if evidenced); a defense-in-depth proposal moving the check into the data layer; and a disclosure-call plan with severity honestly assessed.

---

## Drill 6 — "Ghost probes"

**The page:** Billing anomaly: a workspace deleted all its monitors last month but Cloud Tasks volume attributed to it hasn't dropped. Finance wants to know if we've been probing deleted monitors — and if customers' *deleted* endpoints are still receiving our traffic.

**Your job:** Resolve the docs' standing bug-suspicion with evidence: does the cron selection query exclude soft-deleted monitors?

<details><summary>Hint 1</summary>The selection query is `apps/workflows/src/cron/checker.ts:98-108`. Read the actual WHERE clauses. What does it filter on — and is `deletedAt` among them?</details>
<details><summary>Hint 2</summary>Now check the delete verb: does `packages/services/src/monitor/remove.ts` (or equivalent) set `active=false` as well as `deletedAt`? If yes, the scheduler is *accidentally* safe; if no, ghosts.</details>
<details><summary>Hint 3</summary>"Accidentally safe" is a finding too: the invariant holds only because two independent code sites cooperate. What single-line change makes it structural, and what test pins it?</details>

**Resolution criteria:** A definitive answer with file:line evidence for both the query and the delete verb; the legal/illegal table for the 8 (active × status × deleted) states; and either the bug ticket (with the fix and its test) or the hardening ticket (make the query filter `deletedAt` regardless, plus a test asserting soft-deleted monitors never enqueue). Any answer without having read both sites is a fail regardless of correctness.
