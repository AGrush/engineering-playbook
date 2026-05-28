# General Engineering Playbook

**Last reviewed: May 2026**

A short, opinionated guide for starting a production-grade, AI-friendly, team-onboardable platform from day one. Every decision below is a default — change only with explicit reasoning recorded in an ADR (see §17).

Three questions this repository is designed to answer:
1. **Why is each system the current best practice?** Each section includes a "Why now" note explaining what changed, what the older approach was, and why this is the right choice in 2026 rather than just an opinion.
2. **How does this enable AI-assisted development?** Each section includes an "AI impact" note explaining how the decision directly makes an AI agent (Cursor, Copilot, Claude) faster, safer, and less likely to break your codebase.
3. **How does an AI actually build with this playbook?** [`00-genesis-and-build-cadence.md`](00-genesis-and-build-cadence.md) defines the operating layer on top of §1–§24: a one-time Genesis intake that converts this playbook into a project-specific build plan, and an autonomous build loop the agent runs after. The playbook is the reference manual; `00-` is the protocol for applying it.

---

## What this actually is (in AI-build-system terms)

**A Tiered Document-Graph Router with episodic memory pruning, driven by a self-grounding loop and backed by deterministic verification gates.** Hand-rolled in markdown and shell — no agent framework, no vector store, no external memory service. The repo is the database. The IDE is the runtime. Cursor rules are the always-on conscience. The phase doc is the working memory. The playbook is the long-term reference.

### The system in one paragraph

The AI runs a one-time **Genesis** intake that converts this playbook into six tiers of project-specific artifacts: cursor rules, an index/router, domain docs, a phased master plan, a parking-lot watchlist, and a pre-shipping handoff register. From then on, an autonomous **Build Loop** reads only the active phase doc plus auto-attached cursor rules, picks one coherent task batch, implements it, verifies against a layered testing ladder (TypeScript → ESLint → Vitest → integration → Playwright), commits, updates a per-phase 20–50 line micro-history block, parks anything deferred, and stops at the next phase boundary. The micro-history is the AI's episodic memory; it's pruned the moment the phase closes. Completed phases never re-enter the active context. The detailed structural layout — tier table, operating modes, failure-mode mitigations, runtime folder tree — is in "Structure and artifacts" below.

### How it compares to mid-2026 AI-build approaches

Most "let the AI build it" systems in 2026 fall into one of four buckets — each with a different failure mode that becomes painfully visible past week two of a real project:

| Approach | How it works | Where it fails |
|---|---|---|
| **Big-context megaprompt** | Paste the whole codebase + playbook + chat history into a single huge prompt | Degrades fast past ~100K tokens of project context; reasoning quality drops; cost explodes; goal drift compounds |
| **Heavy tool-use / MCP-only** | Agent calls tools per turn, holds no persistent narrative state between iterations | Works for one-shot tasks; the loop forgets what it just did; multi-day builds drift off-spec without anyone noticing |
| **Pure RAG over docs** | Retrieve relevant chunks per turn, no episodic structure | Chunks lose causality and ordering — exactly what a build process needs. Good for Q&A, bad for "what happened in Phase 7 and why" |
| **Vibecode loop** | Agent keeps trying; no verification gates; commit-and-pray | Produces volume; very little of it works; cascading errors compound silently because nothing independent corroborates the AI's "done" |

This system is none of those. It is:

- **Document-graph routed, not megaprompt.** The loop reads only the active phase doc plus auto-attached cursor rules — typically 2–5K tokens, not 100K. Context budget stays bounded as the project grows from 10 phases to 100. Domain docs and the playbook are opened on demand only, via four explicit triggers (`00-` §1.4), and every consulted section is written back into the phase doc with a provenance marker (`<!-- inlined from playbook §X -->`), so the same gap never costs a second consultation.
- **Episodic and pruned, not chunked-RAG.** Each phase carries a `Current implementation status` field — a structured 20–50 line micro-history of what just happened, with causal ordering preserved. It's the AI's working memory for the active phase. The moment a phase closes, that block becomes historical record; the loop does not re-read entries from completed phases. Tokens stay bounded; staleness can't compound across phases.
- **Verification-gated, not vibecoded.** Every loop iteration runs the §24 testing ladder — **TypeScript (2s) → ESLint (5s) → Vitest (30s) → integration (2min) → Playwright (10min)**. When the loop writes "done" to the watchlist or status block, that claim is corroborated by typecheck + lint + unit + integration + boundary checks. An AI that lies to the status block still fails an independent typecheck/lint/test gate. §24 calls this *"the more independent the layers, the harder gaming becomes."*
- **Self-grounded by hard reference, not by hope.** Each phase doc lists a `Source docs:` array — not an abstract suggestion but a literal "open these files this turn" instruction. The read-don't-guess rule (`00-` §4.1 invariant 2) makes citing-without-reading a violation, and the consultation-writeback rule turns every on-demand playbook consultation into permanent phase-doc enrichment. Lazy link traversal — the failure mode where the AI claims to follow a rule it hasn't actually read — is structurally blocked.

### The phase doc is itself a memory contract

Every phase in `13-master-build-plan.md` carries the same six fields, in the same order:

```
Goal — what this phase accomplishes
Source docs — the literal "open these files this turn" list for the AI
Steps — the ordered work to do
Current implementation status — the 20–50 line micro-history, appended each iteration
Verification — the deterministic gates that confirm "done"
Do not — explicit anti-scope for this phase
```

That structure isn't a template suggestion — it's the contract the loop reads against on every iteration. Goal / Steps / Source docs / Verification / Do not are immutable for the phase; `Current implementation status` is the only field the loop writes to. When the phase closes, the status field is frozen as record and dropped from active context. Future phases never see it.

### The build loop does four jobs simultaneously

Every iteration is the canonical agentic-LLM cycle — **Plan → Execute → Verify → Persist → Prune** — hand-rolled in markdown so it runs inside a normal IDE (Cursor, Claude Code, Codex CLI) with zero framework dependencies:

1. **Re-ground** — read the master plan + active phase doc + auto-attached cursor rules. Compare against repo state. Identify the next coherent task or batch.
2. **Execute** — implement the chosen slice, no more, no less. The phase doc's `Do not` list and the cursor rules constrain scope.
3. **Verify** — run the §24 testing ladder. Self-review against the per-batch rubric (`00-` §4.3) — DRY/KISS/YAGNI, single source of truth, file size, perf, security, race conditions, documentation regression, doc sprawl, provenance markers.
4. **Persist + Prune** — append to the phase doc's `Current implementation status`; add watchlist entries for deferred work; add pre-shipping handoff entries for human-blocked items; commit. At phase close, the status block is frozen and the next phase becomes the active doc.

### Why this is up-to-date for mid-2026 specifically

Three platform shifts in 2026 make this approach work where naive alternatives don't:

1. **Cursor rules auto-attach by glob** (Cursor 3.x+). That makes Tier 0 — always-on guardrails — actually always-on, not "loaded if the agent remembered to include them." Older IDE-agent setups had no equivalent.
2. **Parallel agents in worktrees** (Cursor 3.3+, Claude Code, Codex CLI) load rules independently per agent. Good rules compound across parallel agents; poor rules multiply inconsistency. This system assumes parallel agents are normal, not exceptional, because the phase doc + cursor rules give each agent the same grounding.
3. **Context windows are now large enough for the active phase doc + cursor rules + relevant code, but still not for the whole codebase.** That's exactly the regime the Document-Graph Router optimizes for: one focused doc preloaded, everything else opened on demand.

### Benefits, concrete

- **Documentation matches reality at every commit.** The `Current implementation status` block is updated against verification gates in the same PR as the code change. Documentation drift is structurally impossible without a test failure.
- **Token cost is bounded by the active phase, not the project.** A 50-phase project costs the same per iteration as a 5-phase project, because completed phases don't load.
- **Stop-and-resume is free.** The phase doc *is* the externalized state. Closing the chat, opening a new session, switching agents — all preserve the build's progress because state lives in markdown, not in chat history.
- **Parallelization is free.** Two agents can build two phases in two worktrees, each reading their own active phase doc. No shared mutable state, no coordination layer.
- **Audit is free.** Every phase's `Current implementation status` is a chronological record of what the AI did. ADRs cover *why* decisions were made; the status block covers *how* execution happened. A reviewer can replay the entire build from the artifacts alone.
- **Doc sprawl is prevented by construction.** Every `.md` file has exactly one home (§17 Documentation Governance). No `TODO.md`, no `STATUS.md`, no scratch notes accumulating in feature folders. The cursor rule blocks at IDE-write time; the loop self-review catches at commit time; CI can fail the build on banned filenames.
- **The same artifacts work for humans.** Phase docs read like a plain build journal. A new contributor can pick up `docs/build/` and understand the entire project's history, scope, and current state without ever speaking to the AI.
- **Zero framework lock-in.** Everything is markdown plus a few cursor rules. Swap Cursor for Claude Code, swap the agent for a human, swap the cursor rules for `AGENTS.md` — the system survives because the artifacts survive.

---

## What a build actually looks like, start to finish

**Day 0 — Genesis (one conversation, ~1 hour)**

You open Cursor in your repo and send two words: **"Start Genesis."** If the playbook is already in `docs/playbook/`, the cursor rule picks it up and the AI opens `00-genesis-and-build-cadence.md` automatically. If starting from scratch, you pass the raw GitHub URL once — after that it's always just "Start Genesis." The AI asks you 13 questions — what the product does, target platform, stack preferences, monetization, scale expectations, hard constraints, anything you've seen go wrong before, and a final open-ended catch-all for anything else (Notion docs, competitor references, prior failed attempts, industry context, whatever doesn't fit the structured questions). You answer. It locks the stack, writes three ADRs (stack choices, trust boundaries, repo topology), scaffolds the folder structure, writes 8–12 project-specific domain docs (product plan, DB schema, route map, design system, AI governance, etc.), generates the master build plan with every phase listed out (Phase 0 through however many you need), creates the empty watchlist and handoff register, and writes your cursor rules — all distilled from your answers and this playbook. You review the artifacts and approve. Genesis is done.

**Day 0 still — human setup window (~30–60 min, before Phase 0)**

Before queuing the first loop run, do the external wiring once: create service accounts (Supabase, Stripe, etc.), generate API keys and add them to `.env.local`, run `supabase init` / `supabase link`, configure MCP servers in `.cursor/mcp.json`, push the scaffold commit, confirm the testing ladder passes clean. This is the only time you need to interrupt the loop for external setup — the loop will log any mid-build blockers to the pre-shipping handoff register and keep moving. See `00-` §8.6 for the full checklist.

**Day 0 still — Phase 0 (repo scaffold, CI, tooling)**

You queue the loop prompt: "continue building, current phase is 0." The AI reads the master plan, finds Phase 0, opens the domain docs Phase 0 cites, sets up the monorepo, configures TypeScript + ESLint + Vitest + Playwright, wires CI, verifies the whole testing ladder passes. Updates Phase 0's status field with what it just did. Stops. You review the commit, queue the next iteration.

**Day 1 — Phase 1 (auth)**

Same loop prompt. The AI reads the master plan, moves to Phase 1, opens the DB schema and auth domain docs. Implements Supabase auth, RLS policies, session handling. Runs the test ladder. Updates Phase 1 status. Anything it can't finish goes to the watchlist with a cleanup trigger. Anything it needs you to provide (e.g. OAuth app credentials) goes to the pre-shipping handoff register. Stops at the phase boundary.

**Day 2–N — Phases 2 through 12 (features, by phase)**

Each iteration: queue loop prompt → AI re-grounds in master plan + active phase → builds the next slice → verifies → commits → updates status → stops. You review at each phase boundary. If a new domain area appears mid-build (a scope addition you didn't anticipate), the AI writes a new domain doc (`16-…md`), adds the relevant phases to the master plan, and continues. The domain docs stay current; the master plan stays the controller.

**Near launch — Phase 13+ (cleanup, polish, pre-shipping)**

The watchlist gets worked through as a dedicated cleanup phase. The pre-shipping handoff register tells you exactly what still needs a human: App Store credentials, production environment variables, real-device test sign-off, legal review. Nothing is forgotten — it was logged the moment the AI hit the blocker, not discovered at launch.

**Launch — and after**

The master build plan is a complete record of everything that was built and when. The domain docs are the current ground truth of the project. The ADRs explain every significant decision. A new contributor (human or AI) can pick up the project cold, read `docs/build/`, and understand the full scope, history, and current state without asking anyone.

---

## Structure and artifacts

Not a tutorial. Not a checklist. This repository is **a reference playbook (§1–§24) plus an operating system (`00-`) that turns the playbook into project artifacts an AI agent can execute against.**

### Two layers, one system

| Layer | Files | Purpose | Lifecycle |
|---|---|---|---|
| **Reference manual** | `01-…md` through `09-…md` (§1–§24) | Universal engineering rules: types, boundaries, data, caching, auth, security, testing, observability, CI/CD, AI rules, doc governance, payments, evals | Updated when industry best practice changes |
| **Operating layer** | `00-genesis-and-build-cadence.md` | The protocol an AI agent follows to consume the playbook and build a specific project: Genesis intake → produce project artifacts → autonomous build loop with verification gates | Versioned alongside the playbook; pulled into projects as a frozen snapshot at Genesis |

Without the operating layer, the playbook is a long document an AI might or might not consult. With it, the playbook becomes the input to a deterministic build process whose output is a project that **inherently respects every rule in §1–§24**, because those rules have been distilled into project-specific artifacts the AI reads every iteration.

### The six artifact tiers

When an AI runs Genesis for a new project, it produces six tiers of artifacts. Each tier has a fixed role at build-loop time:

| Tier | Artifact | Role at loop time |
|---|---|---|
| **0** | `.cursor/rules/*.mdc` | Always-on AI guardrails, auto-attached every iteration |
| **1** | `docs/build/README.md` | Index / table-of-contents router — tells the AI which doc to load when |
| **2** | `docs/build/01-…md` through `12-…md` | Domain docs (product, DB plan, design system, caching, AI governance, etc.) — extracted from the playbook and adapted to *this project*, never generic |
| **3** | `docs/build/13-master-build-plan.md` | Phased execution sequence — the only doc with a build order |
| **4** | `docs/build/14-code-quality-watchlist.md` | Parking-lot register for deferred work with explicit cleanup triggers |
| **5** | `docs/build/15-pre-shipping-handoff.md` | Items only a human can resolve (developer-account values, real-device tests, signed builds, store listings) |

This is a **Document-Graph Router with episodic memory pruning**. Each phase doc carries a `Current implementation status` field — a 20–50 line micro-history of what just happened — keyed to that phase and never re-read once the phase closes. Tokens stay bounded; context stays grounded; completed phases don't pollute future iterations.

### Two operating modes

| Mode | Trigger | What the AI does |
|---|---|---|
| **Genesis** *(one-time per project)* | Human sends "Start Genesis." | Interactive intake (13 questions) → stack lock with ADRs → repo topology decision → folder scaffold → write all six tiers of artifacts (project-specific, not template) → human approval gate → ready for loop |
| **Build Loop** *(N times per phase)* | Human queues a loop prompt | Re-ground in master plan + active phase doc → pick the next coherent task or batch → implement → run the verification ladder (TypeScript → ESLint → Vitest → integration → Playwright) → self-review against the per-batch rubric → commit → update the phase doc's `Current implementation status` → append to watchlist if anything was deferred → stop at phase boundary |

Genesis is interactive; the build loop is autonomous between phase boundaries. The phase boundary is the natural human review gate.

### How the system prevents the four AI-build failure modes

| Failure mode | Why it happens in naive AI builds | How this system prevents it |
|---|---|---|
| **Context bloat / payload limits** | Agents bulk-load everything "just in case" | Loop never bulk-loads the playbook. Phase doc is self-contained; playbook sections are opened on demand only when a specific trigger fires (`00-` §1.4). |
| **Goal drift over long sessions** | Chat history degrades; AI forgets why it started | Phase doc is the externalized spec. The loop re-grounds in it every iteration; chat history is not the source of truth. |
| **Lazy link traversal / hallucinated citations** | AI claims to follow a rule it hasn't actually read | Hard "read-don't-guess" rule (`00-` §4.1 invariant 2): citing a playbook section requires opening it in this turn via the `Read` tool. Resolutions get written back into the phase doc (consultation-writeback rule), so the same gap doesn't require re-consultation next iteration. |
| **Cascading errors from bad AI self-reports** | Agent marks "done" in a notes file; nobody verifies | Layered verification: every loop iteration runs the testing ladder (TS 2s → ESLint 5s → Vitest 30s → integration 2min → Playwright 10min, per §24). An AI that lies to the watchlist still fails an independent typecheck/lint/test gate. The more independent the layers, the harder gaming becomes. |

### Documentation drift — three failure modes, all addressed structurally

The standard 2026 AI-assisted codebase drifts in three ways. This system addresses each one at the source:

| Drift mode | Symptom | Mitigation in this system |
|---|---|---|
| **Sprawl** | `TODO.md`, `STATUS.md`, `feature-x-notes.md`, `meeting-notes-YYYY-MM-DD.md`, investigation writeups, duplicate READMEs accumulate in feature folders. Future AI sessions read stale fragments and propagate staleness. | §17 Documentation Governance defines a positive taxonomy (16 canonical doc locations) and a banned-pattern list. Three-layer enforcement: cursor rule (`001-core-project.mdc`) blocks at IDE-write time; build-loop per-batch self-review (`00-` §4.3) flags any new `.md` outside the taxonomy; CI/pre-commit hook fails the build on banned filenames. |
| **Decoupling** | Docs describe an architecture from three months ago; code has moved on. Neither human nor AI knows which is authoritative. | The phase doc's `Current implementation status` field is updated every loop iteration, against verification gates that confirm the claim. The status block is committed in the same PR as the code change. Drift between code and doc is impossible without a test failure. |
| **Token bloat** | Agent preloads the whole playbook "to be safe." Context budget burns on irrelevant material; reasoning quality degrades. | Loop reads only the phase doc + active cursor rules. Playbook is consulted on demand via the four `00-` §1.4 triggers (silence, cited rule, architectural choice, new doc-location uncertainty). Each consultation is recorded back into the phase doc with a provenance marker, so future iterations don't re-trigger. |

### How a project built with this looks at runtime

```
project-root/
  .cursor/rules/                   ← Tier 0: always-on AI guardrails (project-specific)
  apps/, packages/, tooling/       ← Code (monorepo per §2 repo topology)
  docs/
    playbook/                      ← Frozen snapshot of this playbook at Genesis (re-baselined per 00- §9.2)
    build/
      00-genesis-and-build-cadence.md  ← Frozen copy of the operating layer
      README.md                    ← Tier 1: index / router
      01-…md through 12-…md        ← Tier 2: project-specific domain docs
      13-master-build-plan.md      ← Tier 3: phased execution sequence
      14-code-quality-watchlist.md ← Tier 4: parking-lot register
      15-pre-shipping-handoff.md   ← Tier 5: human-input handoff
    adr/                           ← Architecture Decision Records (one per significant decision)
    tasks/                         ← Off-loop task briefs (T-NNN-*.md) for ad-hoc work
    runbooks/                      ← Operational procedures (deploy, rollback, incident response)
    postmortems/                   ← Real incidents, date-stamped
  README.md                        ← One-page repo orientation (max ~50 lines, links to docs/build/)
  CLAUDE.md, AGENTS.md             ← Mirrors of core cursor rules for non-Cursor agents
```

Every `.md` file fits exactly one row in the §17 taxonomy. No `TODO.md`, no `STATUS.md`, no `meeting-notes.md`, no `.md` files inside `src/`. If a new doc need genuinely doesn't fit the taxonomy, the rule is to write an ADR proposing the extension — not to invent a new location by writing the file.

---

## Navigation

**Split edition** — same content as [`../general-engineering-playbook.md`](../general-engineering-playbook.md), in smaller files. Section numbers (§1–§24, §8A) and wording in the parts are unchanged.

**Full-file backup for crosscheck:** [`_backup/general-engineering-playbook-full-2026-05-27.md`](_backup/general-engineering-playbook-full-2026-05-27.md)

### How to use this

| When | Read |
|------|------|
| **Starting a new project with an AI agent** | [`00-genesis-and-build-cadence.md`](00-genesis-and-build-cadence.md) — the entry-point protocol. The AI runs Genesis, produces a project-specific build plan, and drives the loop. |
| **Starting a new project manually** | [`01-checkpoints-and-sanity.md`](01-checkpoints-and-sanity.md) first (milestones → PR sanity → anti-patterns → onboarding) |
| **Choosing stack** | [`09-appendix.md`](09-appendix.md) + write ADR |
| **Building features** | Parts `02` → `08` in order below |
| **Every PR** | §23 in [`01-checkpoints-and-sanity.md`](01-checkpoints-and-sanity.md) |
| **Lookup one topic** | Use the file map below |

### Recommended reading order

0. [`00-genesis-and-build-cadence.md`](00-genesis-and-build-cadence.md) — **the build-system entry point.** Defines Genesis (one-time human-AI intake that converts this playbook into a project-specific master build plan) and the autonomous build loop the agent runs after. Read this first if you're using an AI agent to build a project with this playbook.
1. [`01-checkpoints-and-sanity.md`](01-checkpoints-and-sanity.md) — **§21A** milestone checkpoints, **§23** five-minute PR sanity test, **§22** anti-patterns, **§21** onboarding checklist *(action-first reference; text verbatim from monolith)*
2. [`02-foundation.md`](02-foundation.md) — §1–§6 philosophy, repo, types, state, boundaries, backend
3. [`03-data-caching-assets.md`](03-data-caching-assets.md) — §7 database, §8 caching, **§8A** asset taxonomy
4. [`04-platform-runtime.md`](04-platform-runtime.md) — §9 async, §10 auth, §11 real-time & jobs
5. [`05-security-observability.md`](05-security-observability.md) — §12 security, §13 observability
6. [`06-delivery-testing-ai-rules.md`](06-delivery-testing-ai-rules.md) — §14 testing, §15 CI/CD, §16 Cursor rules
7. [`07-standards-integrations.md`](07-standards-integrations.md) — §17 docs, §18 file size, §19 third-party, §20 payments
8. [`08-ai-testing-feedback.md`](08-ai-testing-feedback.md) — §24 AI + testing loop
9. [`09-appendix.md`](09-appendix.md) — default stack table + closing principle

### File map (original section order)

| File | Sections | In monolith? |
|------|----------|---|
| `README.md` | Title + intro (before §1) | Yes |
| `00-genesis-and-build-cadence.md` | Build-system entry point (Genesis Protocol + Build Loop) — operating layer that consumes §1–§24 | **No — additive** |
| `01-checkpoints-and-sanity.md` | §21A, §23, §22, §21 *(reading order)* | Yes |
| `02-foundation.md` | §1–§6 | Yes |
| `03-data-caching-assets.md` | §7, §8, §8A | Yes |
| `04-platform-runtime.md` | §9–§11 | Yes |
| `05-security-observability.md` | §12–§13 | Yes |
| `06-delivery-testing-ai-rules.md` | §14–§16 | Yes |
| `07-standards-integrations.md` | §17–§20 | Yes |
| `08-ai-testing-feedback.md` | §24 | Yes |
| `09-appendix.md` | Appendix + closing principle | Yes |

### Verify split vs monolith

From repo root:

```bash
python3 shared/engineering-playbook/scripts/verify-split.py
```

### Monolith vs split

- **Monolith** (`general-engineering-playbook.md`) — single file, original § order, good for search-in-one-file and AI context dumps.
- **Split** (this folder) — same bytes when reassembled **for files marked "Yes" in the file map above**. `01-checkpoints-and-sanity.md` reorders §21/§21A/§22/§23 only for reading, not for merge-back.
- **`00-genesis-and-build-cadence.md` is additive** — it has no § number and no monolith counterpart. It's the operating-layer doc on top of the playbook, not a chapter of it. The `verify-split.py` round-trip check should skip this file. If your monolith needs to include it, append it under a new top-level heading (e.g. `## Build System (Genesis & Cadence)`) at the end of the monolith — but treat that as a separate decision from the §1–§24 split.
