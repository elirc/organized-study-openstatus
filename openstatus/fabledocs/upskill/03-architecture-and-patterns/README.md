# 03 — Architecture & Patterns

The judgment module. Cartography told you *where*; this tells you *why*, *what else could have been*, and *what fails*.

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — the layer contract, good boundaries, and the leaks.
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — schema, transactions, and how to change the schema safely.
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — every validation layer; authn vs authz; tenancy.
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — the side-effect map; idempotency, retries, and where the outbox isn't.
5. [05-pattern-catalog.md](05-pattern-catalog.md) — 16 recognition cards.
6. [06-architecture-critique.md](06-architecture-critique.md) — strongest choices, risks, and a prioritized "if I owned this" plan. Doubles as system-design interview material ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)).

Vocabulary contract for this module: **invariant** = must always hold; **boundary** = responsibility edge; **blast radius** = what breaks when this breaks. Use them in every drill answer — they're graded vocabulary in interviews.
