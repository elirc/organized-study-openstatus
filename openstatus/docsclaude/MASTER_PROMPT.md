# MASTER PROMPT — Build an Upskilling Curriculum from an Open Source Codebase

Copy everything below the horizontal rule into Codex, Claude Code, or any agentic coding assistant after cloning an open source repository. The agent will analyze the codebase and produce a multi-module learning curriculum that takes a junior software engineer to mid/senior level through code-based review, analysis, and contributions.

The prompt is language-and-framework agnostic. It adapts to TypeScript monorepos, Python services, Go microservices, Rust workspaces — anything substantial enough to host real architecture.

---

## ROLE

You are a senior staff engineer and curriculum designer. You've shipped production code in this stack and you've mentored juniors into mid- and senior-level engineers. You write technical material the way you teach: grounded in real code, honest about tradeoffs, dense with concrete examples, light on ceremony. You assume your reader is capable but new — never condescend, never hide complexity, never wave at hard parts.

## MISSION

You are about to produce a complete learning curriculum, stored under `docsclaude/` (create the folder if needed), that uses the cloned repository as a laboratory for upskilling a junior software engineer to mid/senior level. The curriculum must:

1. **Teach generalizable language, framework, and runtime knowledge** — not just "where things live in this repo."
2. **Use this repo's real code as the worked example** for every concept, with file-and-line anchors.
3. **Move learners through reading, modifying, building, designing, reviewing, and explaining** — five modes of practice, not just reading.
4. **Make the architecture's invariants explicit** — multi-tenant isolation, transaction boundaries, audit logging, validation layering, error mapping, scope/auth enforcement, whatever this codebase actually does.
5. **Include honest critique** of the codebase and of your own material at the end.

This is not a README. This is not API docs. This is a 6–10 week curriculum a junior could follow with discipline and emerge mid/senior-shaped in this stack.

## NON-NEGOTIABLES

- **Every claim about the codebase must cite a real file path and line range.** If you can't cite it, don't claim it. Format: `[path/to/file.ext:LINE-LINE](path/to/file.ext#LLINE-LLINE)` rendered as Markdown links.
- **Open every file you cite before writing about it.** Do not paraphrase from memory or fabricate plausible-sounding line numbers. If a citation is approximate, mark it as such.
- **Density beats length.** Cut filler. Every paragraph should teach something a senior engineer would care about.
- **Teach the *why*, not just the *what*.** What the code does is obvious from reading it. Why it's written that way — what trap it avoids, what invariant it enforces — is the senior insight.
- **Acknowledge what's *bad* in the codebase.** Real codebases have TODOs, fragilities, and shortcuts. A curriculum that pretends otherwise is teaching juniors to be junior.
- **No emojis. No marketing voice. No motivational filler.**

---

## PROCESS

Do these phases in order. Do not skip Phase 1 to start writing files; you will write the wrong things.

### Phase 1 — Reconnaissance (gather facts before designing)

Before writing a single curriculum file, build a structured understanding of the repository. Produce a private internal map (notes you'll reference; you don't need to ship them). Cover:

1. **Repository shape.**
   - Monorepo or single package? What's the workspace tool (pnpm, yarn workspaces, lerna, turborepo, nx, bazel, cargo workspaces, go modules)?
   - List the apps/services and the shared packages/libraries. Note what each one *deploys* vs what each one *exports*.
   - Identify the package manager, runtime, and build tool. Note any unusual choices (Bun, Deno, esbuild, vite, Turborepo remote cache, etc.).

2. **Stack identification.** For each major surface (frontend, backend, DB, infra, etc.) note:
   - The language and version.
   - The major frameworks and libraries (React/Vue/Solid; Express/Hono/Fastify/Axum/Gin; Postgres/SQLite/MongoDB; etc.).
   - The validation/serialization library if any (Zod, Pydantic, serde, Joi).
   - The ORM/query builder if any (Drizzle, Prisma, SQLAlchemy, sqlx, gorm, ent).
   - The auth approach (NextAuth, Lucia, Auth0, custom JWT, session cookies).
   - The test framework and runner.

3. **Architectural conventions.** Read enough representative files to identify:
   - The layering pattern (e.g., router → service → DB; controller → handler → repository; clean architecture; hexagonal). Name it accurately even if the team uses different words.
   - Where business logic lives. Where validation lives. Where errors are mapped at the boundary.
   - How multi-tenancy or per-resource authorization is enforced (if applicable).
   - How transactions are composed.
   - How side effects (emails, webhooks, queues) are dispatched. In-band? Async? Outbox?
   - How tests achieve isolation (per-test transaction rollback? fixtures? mocks?).
   - The repo's discriminated unions, branded types, or other invariant-encoding mechanisms.

4. **The "primary feature thread."** Pick **one** end-to-end feature that touches the most layers — UI → API → service → DB → side effects → public read model (or this codebase's equivalent flow). It will become your *primary* worked example so a learner can build a deep mental model.
   - Choose wisely. The right thread is rich enough to anchor most lessons but small enough to memorize.
   - Then identify **3–5 secondary threads** (different entities, different flows) so examples don't become repetitive. Codex-style docs that loop the same flow over and over are a known failure mode — avoid it.

5. **Known fragilities, TODOs, and risks.** Grep for `TODO`, `FIXME`, `HACK`, `XXX`, `HMM`, and `eslint-disable` / `biome-ignore`. Read the surrounding code. Note which TODOs are load-bearing and which are cosmetic. Make a list — you'll use them in the critique sections and as teaching opportunities for "this is what real code looks like."

6. **Test landscape.** What test types exist? What's covered, what isn't? Are there integration tests against a real database? End-to-end tests? How fast is the suite?

7. **Documentation that already exists.** Read the root `README.md`, any `CONTRIBUTING.md`, `CLAUDE.md`/`AGENTS.md`, architecture docs, ADRs, RFCs. Note what's already taught so you don't repeat it — and so your material doesn't contradict it without acknowledgment.

8. **What the codebase is *for*.** Read the marketing site or product README. Understand the domain. A monitoring SaaS is different from a docs platform is different from a video player. Domain shapes architecture.

Output of Phase 1 (internal, not shipped): a few pages of structured notes. Stack inventory, primary feature thread file list, secondary threads, fragility list, test inventory, doc inventory, domain summary. **Do not proceed without these.**

### Phase 2 — Curriculum design (adapt the template to what you found)

The default folder structure (specified below) is a starting point. Adapt to the stack:

- For a TypeScript/React/Node stack: the default works as-is.
- For a Python/Django backend: replace `02-frontend-deep-dive` with the relevant equivalent (or omit if there's no frontend in scope) and reshape `03-server-mastery` around Django/Flask/FastAPI specifics.
- For a Rust/Go service: focus heavily on the language module (memory model, ownership, concurrency primitives) and the architecture module.
- For a polyglot repo: cover the dominant stack in depth and reference the others in a smaller "language tour" file.

Decide:
- **The week-by-week pacing.** Map modules to weeks based on the depth needed.
- **The primary feature thread.** Lock it in. You'll cite it dozens of times across modules.
- **The secondary threads** that diversify examples in each module.
- **Which "career skills" to include.** If the project has an active OSS contribution culture (active issue tracker, contributor docs), add an OSS-contribution module.

Write the curriculum's top-level `README.md` outlining the journey and a `00-fast-track.md` for time-constrained learners. These set tone for everything else.

### Phase 3 — Write the curriculum

Write modules in dependency order: language → framework → server → architecture → quality → practice → career → critique. Within each module, write the README first, then each numbered file, then exercises.

Per-file specifications are in the **MODULE SPECIFICATIONS** section below. The **UNIVERSAL QUALITY BAR** applies to every file.

### Phase 4 — Self-critique

After all modules are written, produce `docs/CRITIQUE-curriculum.md` (or `docsclaude/CRITIQUE.md` if there's no peer docs to compare against). Honestly assess:
- Where your material is strong.
- Where it's weak or under-served.
- Inaccuracies in your own citations (re-verify a sample of file:line references).
- Bias in your perspective.
- What a different curriculum author would emphasize differently.

This file isn't a victory lap. It's an example of senior-shaped honesty about your own work, and a model the learner can imitate when assessing PRs and design docs.

---

## OUTPUT STRUCTURE

Create exactly this folder structure under `docsclaude/`. Adapt module 02's name to the dominant frontend stack (or remove if not applicable). Adapt 03's emphasis to the dominant backend stack.

```
docsclaude/
├── README.md                              # curriculum overview, week-by-week journey, module index
├── 00-fast-track.md                       # weekend version for time-constrained learners
├── 01-language-and-runtime-core/
│   ├── README.md
│   ├── 01-mental-model.md                 # language-level mental model (closures/ownership/etc.)
│   ├── 02-concurrency-and-io.md           # async model for this language
│   ├── 03-modules-and-tooling.md          # package system, build tool, monorepo wiring
│   ├── 04-type-system-essentials.md       # if statically typed; else "runtime contract patterns"
│   ├── 05-type-system-power-tools.md      # advanced; or extra topic if untyped
│   └── exercises.md
├── 02-{frontend|alt}-deep-dive/
│   ├── README.md
│   ├── 01-how-it-renders.md               # framework's render/update model
│   ├── 02-state-primitives.md
│   ├── 03-state-architecture.md
│   ├── 04-forms-and-validation.md         # or domain-specific equivalent
│   ├── 05-server-vs-client.md             # RSC, SSR, hydration if applicable
│   ├── 06-data-fetching-and-cache.md
│   └── exercises.md
├── 03-server-mastery/
│   ├── README.md
│   ├── 01-runtimes-explained.md           # Node vs Edge vs Deno; or async vs sync; or processes vs threads
│   ├── 02-http-handlers.md                # request lifecycle, middleware, context
│   ├── 03-databases-and-data-layer.md     # queries, joins, transactions, N+1
│   ├── 04-rpc-or-api-internals.md         # how the wire layer actually works in this codebase
│   ├── 05-auth-and-multi-tenancy.md       # sessions, scopes, tenant isolation
│   └── exercises.md
├── 04-architecture-and-patterns/
│   ├── README.md
│   ├── 01-layering.md
│   ├── 02-validation-and-boundaries.md
│   ├── 03-transactions-and-audit.md       # adapt if no transactional layer; else "consistency model"
│   ├── 04-derived-data-and-caching.md
│   ├── 05-patterns-catalog.md             # named patterns with file anchors
│   └── exercises.md
├── 05-quality-engineering/
│   ├── README.md
│   ├── 01-testing-strategy.md
│   ├── 02-writing-tests-here.md           # concrete recipes for this repo's test stack
│   ├── 03-systematic-debugging.md
│   ├── 04-performance-thinking.md
│   ├── 05-security-checklist.md
│   └── exercises.md
├── 06-practice-tracks/
│   ├── README.md
│   ├── 01-reading-katas.md                # 10 short, dense reading exercises with rubrics
│   ├── 02-modify-tickets.md               # 10 small but real tickets, Easy→Expert
│   ├── 03-build-projects.md               # 5 progressively deeper builds
│   ├── 04-senior-katas.md                 # open-ended design/refactor exercises
│   └── 05-oss-contribution-guide.md       # ONLY if the repo accepts external contributions
├── 07-career-skills/
│   ├── README.md
│   ├── 01-code-review-mindset.md
│   ├── 02-writing-prs-and-rfcs.md
│   └── 03-interview-prep.md               # framed around this codebase
└── CRITIQUE.md                            # honest self-assessment of the curriculum
```

If the repo has no frontend, omit module 02 and renumber. If it's a CLI tool or library, replace 02 with a "consumer experience" module. If it's pure infra (Terraform modules, Kubernetes operators), reshape modules 02–03 around declarative-config patterns and operator design.

---

## UNIVERSAL QUALITY BAR

Apply these rules to **every** file you write.

### Anchor every claim

Format file references as Markdown links the IDE can open:
```
[packages/services/src/status-report/create.ts:24-99](../packages/services/src/status-report/create.ts#L24-L99)
```
- Always include the line range, not just the file.
- Prefer narrow ranges (~5–30 lines) over wide ones (>100). Narrow ranges teach precision.
- Re-verify line numbers when you reference older code; the agent's first impression is often slightly off.

### Distinguish illustrative code from quoted code

- **Quoted code** is copy-pasted from the repo with a citation. Use this for any substantive claim.
- **Illustrative code** is hand-written by you to demonstrate a pattern more cleanly than the production code allows. Label it explicitly:
  ```
  ```ts
  // Illustrative — not from the repo
  ...
  ```
  ```
- Never let illustrative code be mistaken for real code. Mislabeled examples are how learners end up with confidently wrong models.

### Teach the why, not just the what

Each substantial code block should be followed by 1–3 sentences of *why this shape* — what invariant it enforces, what trap it avoids, what alternative would have been worse. Without the *why*, learners memorize patterns without understanding when to break them.

### Use "Senior aside" callouts

When you have an observation that a senior would have and a junior wouldn't, mark it explicitly. Format:

> **Senior aside:** *One or two italicized sentences naming the pattern, the tradeoff, or the failure mode being avoided.*

These callouts are how you transmit *taste*. Use them sparingly — 1–3 per file — so they retain weight. Overuse turns them into noise.

### End every file with a "Pitfall checklist"

A bulleted list of 8–12 specific bug patterns or anti-patterns related to the file's topic. Each item should be concrete enough that a reader can recognize it in code review. "Be careful with state" is not a pitfall; "mutating React state in place defeats the equality check, so `setItems(items.push(x))` is a no-op" is.

### End every file with "Drills"

3–7 specific things the reader can do *in this codebase* to test their understanding. Drills should be doable in 5–30 minutes each and should reference specific files. They are not quizzes — they're exercises that build the reflex.

### Honest about gaps

Every file should include at least one moment of "the codebase doesn't do this perfectly, here's the trade-off." Reading a curriculum that pretends the code is flawless teaches juniors to be uncritical readers.

### Difficulty laddering

User stories, modify tickets, build projects, and senior katas should be labeled Easy / Medium / Hard / Expert. Provide a clear gradient so a learner can self-select.

### Self-grading rubrics

For Socratic checkpoints and reading katas, include rubrics: what a strong answer cites, what a weak answer misses. This teaches the meta-skill of recognizing senior-quality answers.

### Source-of-truth links

When a pattern, library, or concept is well-documented externally (React docs, MDN, the framework's own docs, a canonical blog post), reference it once at the top of the file. Don't recreate the official docs.

### Sentence length and density

Aim for one idea per sentence. Use semicolons or em-dashes for closely related ideas. Avoid run-on sentences with multiple "and"s. Long, complex sentences are how juniors learn to write impressive-sounding but vague text — model the opposite.

### Length targets per file

- README files: 50–150 lines.
- Concept files in modules 01–05: 200–500 lines. Dense, not padded.
- Exercise files: 50–250 lines.
- Practice track files: 200–500 lines depending on the number of tickets/projects.
- Critique: 200–600 lines.

If a file is approaching 800 lines, split it. If it's under 100 lines and isn't a README or exercises file, you're under-teaching.

---

## MODULE SPECIFICATIONS

For each module, here is what *must* be covered, why it matters, and how to adapt to the stack.

### Top-level `README.md`

- A 1–3 paragraph framing: who this is for, what's different about this material, the journey arc.
- A week-by-week table mapping modules to weeks.
- A module index with one-line hooks per module.
- A short note on conventions (file references format, "Senior aside" meaning, illustrative vs quoted code).
- A closing paragraph on the senior mindset (one paragraph; not a manifesto).

### `00-fast-track.md`

- The weekend (or single-day) version for someone who needs to get oriented fast.
- 4–6 timed blocks, each with a goal and concrete steps.
- One block must be "trace one feature end-to-end" using the primary feature thread.
- One block must be "make one real change" pointing at the easiest modify ticket.
- One block must be "read critically" pointing at the critique file.

### Module 01 — Language and Runtime Core

The single most important module for upskilling. A junior with shaky language fundamentals stays junior. Cover:

**01-mental-model.md** — value vs reference / ownership / scope / closures / equality / coercion. The model needed to predict program behavior without running it.

**02-concurrency-and-io.md** — for JS/TS: event loop, microtasks, promises, async/await, race conditions, real concurrency patterns in this repo. For Python: GIL, asyncio, the event loop, threading vs multiprocessing. For Go: goroutines, channels, select, context. For Rust: tokio, futures, send/sync. Always include "the most common bug pattern in this concurrency model."

**03-modules-and-tooling.md** — how the import system works, how the package manager resolves dependencies, how the build tool turns source into output. Reading `package.json` / `Cargo.toml` / `go.mod` / `pyproject.toml` *like a senior would*: which fields are load-bearing, what's the dependency graph, what's the lockfile guaranteeing.

**04-type-system-essentials.md** — for statically typed languages: structural vs nominal, narrowing, generics, inference, union/sum types. For dynamic languages: replace with "runtime contracts" — Pydantic, Zod, dataclasses, validation libraries, duck typing pitfalls.

**05-type-system-power-tools.md** — advanced TS (conditional/mapped types, `infer`, template literals, `satisfies`, branded types) or the equivalent advanced feature in this language (Python's `Protocol`/`TypedDict`/`Generic`, Rust's trait objects + lifetimes, Go's generics). The features that make library code feel like "magic" — and the demystification.

**exercises.md** — predict-then-run drills, OpenStatus-style ("predict the output before running"). At least one exercise per concept. Include a self-grading rubric: 0 (couldn't attempt), 1 (off on details), 2 (right with help), 3 (right with explanation).

### Module 02 — Frontend / Primary Stack Layer Deep Dive

Adapt the file names if the dominant frontend is Vue / Solid / Svelte. If there's no frontend, replace with the next-most-important layer (e.g., a CLI library's UX, a Kubernetes operator's reconcile loop).

**01-how-it-renders.md** — the framework's actual render/update model. For React: render vs commit, reconciliation, keys, why hooks have rules. For others: the equivalent mental model. Five-sentence summary of the model up front. Worked example from the primary feature thread.

**02-state-primitives.md** — the framework's state hooks/signals/reactives. What they actually do, what their constraints are, when to reach for each.

**03-state-architecture.md** — local / lifted / URL / server / derived state. Picking the right home. URL-state library if the repo uses one (nuqs, react-router params, similar).

**04-forms-and-validation.md** — the form library (react-hook-form, formik, vee-validate, native form actions). Controlled vs uncontrolled. Async submission. Dirty tracking. Real form from the repo, walked through.

**05-server-vs-client.md** — if the stack has a server/client split (RSC, SSR, hydration, server actions), explain it from first principles. If not, replace with the equivalent boundary in this stack.

**06-data-fetching-and-cache.md** — the data-fetching library (React Query, SWR, Apollo, RTK Query, urql) and the cache mental model. Real mutations and invalidations from the repo.

**exercises.md** — "predict the re-render" drills, network-panel exercises, profiler exercises, refactor exercises.

### Module 03 — Server / Backend Mastery

**01-runtimes-explained.md** — Node vs Edge vs Browser, or async vs sync Python, or single-threaded vs multi-process, or whatever the operative split is. What works where and why. Cite real runtime decisions in the repo.

**02-http-handlers.md** — the request lifecycle from socket to handler. Middleware (framework-level *and* application-level). Context creation. Error boundaries. Real handler code from the repo.

**03-databases-and-data-layer.md** — the query builder or ORM. Reading schemas. Indexes. Transactions. The N+1 trap and how this repo avoids it (cite the batched-enrichment pattern or equivalent). EXPLAIN / query plans if applicable.

**04-rpc-or-api-internals.md** — tRPC, REST, GraphQL, gRPC, JSON-RPC, MCP — whatever this repo uses. Not "how to call it" but "how it works under the hood." Type inference paths, codegen, transformer libraries (superjson), batching.

**05-auth-and-multi-tenancy.md** — sessions, cookies, JWTs, scopes, tenant isolation, IDOR prevention. Cite the real authorization helpers (`requireScope`, role checks, ACL functions). Walk through a cross-tenant attack and where it's stopped.

**exercises.md** — runtime identification, N+1 hunts, auth attack traces, performance counting (round-trips), middleware writing.

### Module 04 — Architecture and Patterns

The taste module. This is where you turn "I can read the code" into "I can argue about whether the code is right."

**01-layering.md** — the layer diagram. Who talks to whom. What belongs in each layer. The exact service contract pattern in this repo. The cost of mixing layers.

**02-validation-and-boundaries.md** — every boundary where untrusted becomes trusted. The validation library at each layer. Why duplication is the point. Error class hierarchy and the error-mapping adapter.

**03-transactions-and-audit.md** — if applicable, atomicity, audit log, fail-closed design. If not applicable (e.g., a read-only service), adapt to "consistency model" or "idempotency design."

**04-derived-data-and-caching.md** — derive vs store. Read models. Cache invalidation strategies. Real cache keys and invalidations.

**05-patterns-catalog.md** — named patterns from the repo, each with: where it is, why it exists, when to use it elsewhere. At least 10 patterns. Each gets ~1 paragraph. End with "how to recognize a pattern in unfamiliar code" — the meta-skill.

**exercises.md** — design exercises, layering questions, "find the missing X" hunts. Open-ended; provide a rubric for self-evaluation.

### Module 05 — Quality Engineering

**01-testing-strategy.md** — the pyramid the team actually builds. What's tested where. Why some things aren't tested. Coverage vs quality.

**02-writing-tests-here.md** — concrete recipes. The test runner's syntax. The transaction-rollback helper or equivalent. Audit/spy assertions. Time/randomness handling. *Copy-pasteable* templates the learner can modify.

**03-systematic-debugging.md** — the general method (reproduce → narrow → hypothesize → test → fix root cause). Browser tools, server tools, DB query log, profiler. Real debugging scenarios from this codebase ("symptom X: where do you look?").

**04-performance-thinking.md** — measure first. The three domains (bundle, render, backend). Tools per domain. Common bug shapes. The 80/20 of perf work (cite real performance wins or near-wins from the repo).

**05-security-checklist.md** — OWASP Top 10 mapped to this stack. SSRF, IDOR, XSS, CSRF, SQLi as they would appear here. Pre-merge security checklist (15–20 items, all concrete). Cite the real security tests in the repo.

**exercises.md** — test-writing, bug-hunting, performance-counting, security-checklist application.

### Module 06 — Practice Tracks

This module is *all* exercises. Heaviest investment of the curriculum.

**01-reading-katas.md** — 10 short reading exercises. Each: cite a file/range, ask 1–3 questions, provide a rubric. Include a "self-prompt for AI help" template that turns AI into a question-asker, not a vending machine. (Codex's `docs/junior-feature-ticket-book/README.md` has a good template — steal the pattern.)

**02-modify-tickets.md** — 10 real tickets, Easy → Expert. Each ticket: acceptance criteria, anchor files, plan outline, what could go wrong, suggested checks. The tickets must touch *different* parts of the codebase — don't have 10 tickets all on the same feature.

**03-build-projects.md** — 5 projects, each multi-day to multi-week, each integrating multiple modules' worth of skills. Each project: design checklist, files affected, stretch goals, what makes it "senior-shaped" work. Encourage RFC-style planning before coding.

**04-senior-katas.md** — 8–12 open-ended design/refactor exercises. No single right answer. Each: present the current state, pose the question, list the tradeoffs to address. The reader argues both sides.

**05-oss-contribution-guide.md** — *only* if the repo accepts external contributions. Cover: how to find a good first issue, how to comment on an issue before claiming it, the repo's CONTRIBUTING conventions, branching, commit messages, PR descriptions, review etiquette, what to do if your PR sits for weeks. This is rare and valuable.

### Module 07 — Career Skills

Short module, high leverage.

**01-code-review-mindset.md** — what seniors look for. The layered checklist (works → correct → stays-correct → good-citizen → kind). How to give review that teaches. How to receive review with grace. Cite a real PR or commit from the repo and walk through reviewing it.

**02-writing-prs-and-rfcs.md** — PR descriptions (what/why/test), RFC template (problem/goals/proposal/alternatives/migration), decision logs, commit messages. *Show* a good one — write a sample PR description for one of the modify tickets.

**03-interview-prep.md** — 40–60 senior interview questions grounded in this codebase. Language, framework, server, architecture, quality, communication. Plus "tell me about a time" prompts framed as "I studied this codebase, what struck me was…" so the reader can talk about open source work in interviews credibly.

### `CRITIQUE.md`

The final file. Honest self-assessment. Cover:

- **What the curriculum did well.** Specific files, specific patterns.
- **What it missed or under-served.** Be specific. "I didn't cover X" with reasons.
- **Inaccuracies you suspect.** Re-verify a sample of citations; flag any you find were imprecise.
- **Where the codebase changed your plans.** "I expected to cover Y, but the codebase doesn't do Y, so I covered Z instead."
- **Bias you brought.** Are you over-rotated on the patterns you personally prefer? Are there equally valid architectures you under-respect?
- **What a different curriculum author would emphasize differently.** A persona contrast: "a backend-leaning senior would have done more on X."
- **Concrete improvements for v2.** What you'd add if you had another pass.

This file is also your test: a learner who can write a similar critique of *their own* work is senior-shaped.

---

## STYLE GUIDE

### Voice
- Direct. Senior peer, not professor.
- Confident but acknowledging uncertainty when present ("I think," "this looks like," "verify before relying on").
- Never condescending. Never apologetic.
- No marketing voice. No motivational filler ("you got this!").
- No emojis unless the codebase's CONTRIBUTING explicitly uses them.

### Formatting
- Markdown only. No HTML except where Markdown can't express it.
- Code blocks fenced with language tag.
- File references as Markdown links: `[path:lines](path#Lline-Lline)`.
- Tables for comparisons (2+ axes), bullets for lists (1 axis), prose for explanations.
- One blank line between paragraphs. No double blank lines.
- Headings: ATX (`# H1`, `## H2`), title case.

### Tone calibration
- A junior should feel respected, not intimidated.
- A mid should feel like there's new depth to learn.
- A senior should feel that someone thoughtful wrote this — not a content mill.

### What to cut
- "Let's dive in." → cut.
- "In this section, we will explore…" → cut.
- "It's important to note that…" → cut; just state the note.
- Paragraph-long preambles → cut to one sentence.
- Bullet lists where each item is a complete redundant sentence → consolidate.

### What to keep
- Specific file paths.
- Concrete code examples.
- "Senior aside" callouts.
- Pitfall checklists.
- Honest acknowledgment of codebase limits.
- One short metaphor per file *max* — and only if it adds clarity.

---

## ANTI-PATTERNS TO AVOID

These are the failure modes I've seen in similar efforts. If you find yourself doing one of them, stop and refactor.

1. **The same feature thread in every file.** Rotate examples across the primary and secondary threads. A learner who reads "the status report flow" 200 times will not be senior-shaped in the *codebase*, just in that one flow.

2. **Codebase-specific knowledge dominating over generalizable skill.** A learner finishing the curriculum should be better at the *language and stack*, not just this repo. Every module should teach concepts that transfer.

3. **Pattern names with one example each.** Real pattern teaching needs the name + problem + shape + multiple examples + failure modes. Otherwise it's a glossary, not pedagogy.

4. **Symptom-indexed debugging guides without teaching the method.** A runbook lists symptoms; a method teaches *how to debug* anything. Teach the method first, then use this repo's symptoms as practice.

5. **"Magic" left unexplained.** If the framework or library feels magical, that's a teaching opportunity. Decompose the magic into the actual mechanism (proxies, codegen, type inference paths, decorators). Don't leave learners believing in magic.

6. **Padded prose.** "It is important to consider that the implications of this choice may have significant downstream effects on the maintainability of the codebase" → "This choice affects maintainability later." Cut every word that doesn't carry its weight.

7. **No exercises, or exercises with answer keys.** Self-grading rubrics and AI-coach prompts, not answer keys. The struggle is the point.

8. **Cargo-culted "best practices."** Every "best practice" should come with the context where it applies. "Always do X" without "because Y, except when Z" is dogma, not engineering.

9. **Citing nonexistent files or wrong line ranges.** Re-verify before shipping. A single wrong citation poisons a learner's trust in everything else.

10. **Hiding the codebase's flaws.** Real codebases have TODOs, fragilities, and outdated patterns. Teaching juniors to see and name them is part of becoming senior. Don't gloss.

11. **Over-formalizing.** Bullet lists nested four levels deep, tables for everything, structured templates that bury the actual idea. Prose with one well-placed table beats a wall of bullets.

12. **Excessive cross-linking.** Every file linking to every other file becomes navigation soup. Link only when a reader benefits from following the link *now*.

13. **One file per pattern.** A "Patterns Handbook" with 30 files of 50 lines each is fragmentation. Combine related patterns into one substantive file.

14. **Re-explaining the official docs.** If React's docs explain hooks excellently, link to them and add the *codebase-specific* perspective. Don't recreate the canon poorly.

15. **Length theater.** A 2000-line file looks impressive but isn't. Density beats length. Aim for the length the content needs.

---

## OUTPUT PROTOCOL

Operate in this sequence. Use the agent's task / todo tool to track progress visibly.

1. **Phase 1 reconnaissance.** Read aggressively. Take notes. Do not write any curriculum files yet.

2. **Confirm with the user** (if the agent has a UI for confirmation): list the primary feature thread, the secondary threads, the stack inventory, and any module-shape adjustments. Wait for confirmation if possible; otherwise proceed and document the choice.

3. **Top-level files first.** `docsclaude/README.md` and `docsclaude/00-fast-track.md` set tone for everything else.

4. **Modules in dependency order.** 01 → 02 → 03 → 04 → 05 → 06 → 07. Within a module: README → numbered files → exercises.

5. **Verify citations as you go.** When you cite a file:line, you must have opened it in the current session (or the prior phase). Re-open if necessary. Imprecise citations are worse than no citation.

6. **Self-critique last.** `CRITIQUE.md` after everything else is done. Re-verify a random sample of 10–20 citations from your other files. Document any you got wrong.

7. **Final readthrough.** Open every README and the fast-track file once more. Check tone, consistency, and that the modules cohere into a journey.

8. **Report.** A short summary message to the user: what you produced, what you adapted from the default template, what you couldn't cover or chose to defer.

---

## CALIBRATION FOR ADAPTATION

The default template assumes a substantial TypeScript/React/Node codebase. For other stacks, here's how to adapt module emphasis:

| If the codebase is… | Increase emphasis on… | Decrease emphasis on… |
| --- | --- | --- |
| Pure Python backend (Django/FastAPI) | async/sync split, type hints + Pydantic, ORM (SQLAlchemy/Django ORM) | RSC, hydration, React Query |
| Go microservices | goroutines/channels, context propagation, error wrapping, generics | type-system power tools |
| Rust workspace | ownership/borrowing, lifetimes, traits + generics, error handling with `Result` | runtime concurrency models |
| Polyglot monorepo | the dominant stack at length; a "stack tour" file for the rest | secondary stacks in depth |
| CLI library | UX, ergonomics, argument parsing, exit codes, plugin systems | server/auth/multi-tenancy |
| Kubernetes operator | reconcile loops, controllers, CRDs, eventual consistency | request/response framing |
| Browser library | bundle size, tree shaking, API design, DOM model, browser quirks | server runtime concerns |
| Mobile app | platform conventions (iOS/Android), state management, navigation | server runtime concerns |

Use judgment. The point isn't to mechanically apply the table — it's to put depth where the codebase puts complexity.

---

## FINAL CHECKLIST BEFORE DECLARING DONE

Tick each one. If any is "no," fix it.

- [ ] Every module README exists and is 50–150 lines.
- [ ] `docsclaude/README.md` includes a week-by-week table and a module index.
- [ ] `00-fast-track.md` is a real weekend plan, not a summary.
- [ ] Every numbered file has: code-anchored examples, "senior aside" callouts (1–3), a pitfall checklist (8–12 items), drills (3–7).
- [ ] At least 80% of code examples cite real files with line ranges.
- [ ] At least one illustrative example per file is labeled as such.
- [ ] Every module's exercise file exists and includes self-grading rubrics where relevant.
- [ ] The practice tracks include 10 reading katas, 10 modify tickets (Easy→Expert), 5 build projects, 8–12 senior katas.
- [ ] Every modify ticket is touching a different area of the codebase. No duplication.
- [ ] Career skills module covers code review, PRs, RFCs, and interview prep.
- [ ] `CRITIQUE.md` exists and is honest. It re-verifies a sample of citations and reports any errors found.
- [ ] No emojis, no marketing voice, no padding.
- [ ] No nonexistent files cited.
- [ ] The curriculum's progression makes sense: a learner could start at fast-track, work through modules, and arrive at "able to argue about architecture" by the end.
- [ ] The codebase's actual TODOs and fragilities are surfaced as teaching opportunities, not glossed.
- [ ] Adapted to this codebase's actual stack — not pasted boilerplate.

When all boxes are ticked, ship the curriculum and report to the user.

---

## EXAMPLES OF THE TARGET QUALITY

If you need to calibrate the style, follow these as reference (these are real files I've seen produce the right quality):

- For *pattern catalogs with file anchors*: see `docs/pattern-recognition-handbook/` in the OpenStatus repo (the Codex-produced effort). Strengths: every pattern named, every example anchored.
- For *language depth grounded in real code*: see `docsclaude/01-javascript-typescript-core/` in the OpenStatus repo (the Claude-produced effort). Strengths: concept → real example → pitfall → drill.
- For *honest self-critique*: see `docs/CRITIQUE-codex-vs-claude.md`. Strengths: names strengths and weaknesses of both efforts without false balance.

You don't have to copy structure from these — they're the result of one author's choices, not the platonic ideal. But the *quality bar* they set is what you're aiming for.

---

## CLOSING NOTE TO THE AGENT

You are not summarizing a codebase. You are teaching a person to become a senior engineer using this codebase as a lab. Every page should make a junior more capable when they close it. Every page should make a senior nod with recognition.

If you produce a 30,000-line curriculum that doesn't move the needle on a learner's actual judgment, you've failed. If you produce a 10,000-line curriculum that turns a focused junior into a credible mid-level in 8 weeks, you've succeeded.

Begin with Phase 1. Take your time. Don't write a single file until you understand the codebase the way a senior engineer who's worked on it for six months would.
