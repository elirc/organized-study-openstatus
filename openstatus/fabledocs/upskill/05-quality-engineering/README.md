# 05 — Quality Engineering

How this repo keeps (and sometimes fails to keep) itself correct, fast, secure, and observable — and the transferable discipline behind each.

1. [01-testing-strategy.md](01-testing-strategy.md) — the layers that exist here, what belongs where, flake prevention.
2. [02-writing-tests-here.md](02-writing-tests-here.md) — six copy-adaptable recipes using the repo's real helpers.
3. [03-systematic-debugging.md](03-systematic-debugging.md) — the method + five repo-specific scenarios.
4. [04-performance-thinking.md](04-performance-thinking.md) — measure-first; likely hotspots with anchors.
5. [05-security-checklist.md](05-security-checklist.md) — the full map, ending in a pre-merge checklist.
6. [06-observability-and-operations.md](06-observability-and-operations.md) — "how would I know this broke?" per major flow.

The through-line: **quality is invariants + evidence.** Tests pin invariants; observability produces evidence; debugging is the search procedure between them.
