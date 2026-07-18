# Labs: pipeline & Go

The two least-practiced muscles in fabledocs 1-3: making the untested workflows pipeline testable (the named precondition for both ADR-010's gap and the capstone), and reading/modifying the Go checker. These are ordered — Lab A unblocks the capstone's Milestone 5.

---

## Lab A — Make workflows testable (the clock + collaborator refactor)

**Why this exists:** `apps/workflows` — cron fan-out, `/updateStatus`, alerting — has zero tests, and every doc in this series names the same blocker: its collaborators (wall-clock, Cloud Tasks client, notification providers, Tinybird) are reached directly, not injected. This lab is the refactor fabledocs2 Part 4 said "I will approve the hour it opens."

**Step 1 — Seam inventory (1 hr).** Read `apps/workflows/src/cron/monitor.ts` and `src/checker/alerting.ts` and list every direct reach to an un-mockable collaborator: `Date.now()`/`new Date()` calls, the tasks client, provider sends, Redis, email. Output: a table of file:line → collaborator → proposed seam.

**Step 2 — The clock (2 hrs).** Smallest possible change: thread a `now: () => number` (decide the unit! Lab 03/7 showed you why) through `getScheduledTime` (`cron/monitor.ts:393-407`), defaulting to the real clock so no call site changes behavior. Then write the first test file the workflows app has ever had: table-driven cases for `getScheduledTime` — including the exact ms/seconds trace from Lab 03/7, now pinned forever. If the docs' suspicion was right, this test *documents the bug* first (assert current behavior, mark with a TODO) — fixing it is a separate, reviewable commit.

**Step 3 — The provider seam (2-3 hrs).** Alerting's send path takes its provider client(s) as a parameter (or via a context object mirroring `ServiceContext`), defaulting to the real ones. Write the first `/updateStatus`-adjacent test: a transition that should trigger, with a recording fake provider, asserting exactly one send and one trigger row.

**Step 4 — The concurrency test (the R1 evidence, 2-3 hrs).** Fire two concurrent regional reports for the same monitor at the handler (Promise.all against the Hono app, real DB) and assert **exactly one** trigger row. On `main`-equivalent code this test is *expected to be flaky-red* — that's not failure, that's the evidence-gathering the architecture critique called for before fixing R1. Record the flake rate over 50 runs; that number is a sentence in your interview story ("I didn't claim the race, I measured it at N%").

**Done when:** workflows has a test directory with ≥3 green tests + 1 documented expected-flaky concurrency test, and nothing about production behavior changed (pure seam-introduction, defaults preserved). This PR shape — *refactor for testability with zero behavior change, then tests, then fixes as separate commits* — is itself the lesson.

---

## Lab B — Read the Go checker like you own it

**Why this exists:** the curriculum covers the TS side deeply; the Go side you've mostly taken on faith. You don't need to be a Go engineer; you need to never be afraid of this directory.

**Step 1 — The probe trace (1 hr).** Read `apps/checker/handlers/checker.go:47-120` and write the numbered lifecycle of one HTTP probe: payload decode → auth check (L52) → client construction with per-monitor timeout (L76-79) → request → assertion evaluation (L334-372) → Tinybird write (L255) → status-transition report (`checker/update.go:29-93`). Ten numbered lines, no more.

**Step 2 — The assertion engine (1-2 hrs).** TS defines assertion *schemas* (`packages/assertions`); Go *evaluates* them (`checker.go:334-372`). Map each TS assertion type to its Go evaluation branch. Then answer the drift question: if TS ships a new assertion kind and Go hasn't, what exactly happens at runtime — error, or silent pass? Read the Go default branch and find out; this is the ADR-003 data-only-boundary cost, made concrete.

**Step 3 — Write one Go test (1-2 hrs).** Pick an assertion case with no existing coverage in `apps/checker/**/*_test.go` (inventory first) and add a table-driven test for it, matching the existing test style. Run `go test ./...`. The goal isn't Go mastery — it's proving to yourself the barrier is an afternoon, not a career.

**Step 4 — The retry/backoff audit (1 hr).** Read the `backoff` usage around probes (`checker.go:276`). Answer precisely: what's retried, how many times, and what does a "down" verdict mean in retry terms — first failure or exhausted retries? Now connect it upward: how does this interact with the quorum layer (ADR-009)? Write the two-sentence answer to "how do you avoid false-positive alerts?" that cites both layers.

**Done when:** you have the trace, the mapping table with the drift answer, one green new Go test, and the two-layer false-positive answer written out.

---

## Lab C — Contract drift, caught in CI (stretch)

ADR-001/003 both flag the same unpaid debt: nothing validates the TS→Go JSON payload contract. Close it: generate a JSON Schema from the TS payload builder (`apps/workflows/src/cron/checker.ts:249-320` — a zod schema of the payload probably already half-exists; if not, write one), commit a matching fixture payload, and add a Go test that unmarshals the fixture into the Go structs and asserts no unknown/missing required fields (Go's `json.Decoder.DisallowUnknownFields` is the 5-line trick). Wire both into CI mentally (or actually — a `test.yml` step).

**Done when:** deliberately adding a field on the TS side without the Go side fails a test on one side or the other. You've converted a convention into a contract — which, if you're keeping score, is the recurring theme of every improvement this pack has had you make.
