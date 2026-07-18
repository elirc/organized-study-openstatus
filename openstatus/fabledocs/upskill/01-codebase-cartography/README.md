# 01 — Codebase Cartography

Before you can judge a codebase you have to be able to *navigate* it without a guide. This module builds the map layer by layer:

1. [01-system-map.md](01-system-map.md) — what the monorepo contains, who owns what, what's public vs internal.
2. [02-file-reading-order.md](02-file-reading-order.md) — 30 files in the order that builds understanding fastest, with junior/mid/senior paths.
3. [03-domain-glossary.md](03-domain-glossary.md) — the product nouns (monitor, incident, status report, component…) and where each lives in code.
4. [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) — package manager, build graph, runtimes (Node/Bun/Edge/Go), env vars.
5. [05-key-flows.md](05-key-flows.md) — seven end-to-end traces. This is the module's centerpiece; the rest of the curriculum rotates through these flows.

Transferable skill: **cartography is a repeatable procedure**, not talent. Inventory → manifests → entry points → schema → one flow end-to-end. Practice it here, then time yourself doing the same on any unfamiliar repo (a common take-home/onboarding interview task is literally "how would you get up to speed on our codebase?" — your answer is this module's table of contents).
