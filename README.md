# General Engineering Playbook

**An opinionated engineering playbook with an autonomous AI build system built in.**

The playbook (`01–09`) is a battle-tested reference covering everything a production-grade app needs: architecture, data, auth, security, testing, CI/CD, observability, AI governance, and documentation standards — each rule explained with *why it matters in 2026* and *how it makes AI-assisted development safer*.

The build system (`00-genesis-and-build-cadence.md`) sits on top of the playbook and turns it into action. Tell the AI what you're building, and it converts the playbook into a project-specific phased build plan, writes your cursor rules, scaffolds your repo, and then drives the build autonomously — phase by phase, with verification gates, structured deferrals, and a clean stop at every phase boundary for you to review.

**Last reviewed: May 2026**

---

### Quick start

```bash
# 1. Clone the playbook into your project
git clone https://github.com/[your-org]/engineering-playbook docs/playbook

# 2. Open your project in Cursor and send:
Start Genesis.
```

The AI asks you 15 questions about your project, produces the full build plan and all project artifacts, waits for your approval, then is ready to build. Nothing gets implemented until you say go.

---

## What this actually is (in AI-build-system terms)

**A Tiered Document-Graph Router with episodic memory pruning, driven by a self-grounding loop and backed by deterministic verification gates.** Hand-rolled in markdown and shell — no agent framework, no vector store, no external memory service. The repo is the database. The IDE is the runtime. Cursor rules are the always-on conscience. The phase doc is the working memory. The playbook is the long-term reference.

### The system in one paragraph

The AI runs a one-time **Genesis** intake that converts this playbook into six tiers of project-specific artifacts: cursor rules, an index/router, domain docs, a phased master plan, a parking-lot watchlist, and a pre-shipping handoff register. From then on, an autonomous **Build Loop** reads only the active phase doc plus auto-attached cursor rules, picks one coherent task batch, implements it, verifies against a layered testing ladder (TypeScript → ESLint → Vitest → integration → Playwright), commits, updates a per-phase 20–50 line micro-history block, parks anything deferred, and stops at the next phase boundary. The micro-history is the AI's episodic memory; it's pruned the moment the phase closes. Completed phases never re-enter the active context. The detailed structural layout — tier table, operating modes, failure-mode mitigations, runtime folder tree — is in "Structure and artifacts" below.

### How it compares to mid-2026 AI-build approaches

AI coding in 2026 is genuinely capable at the session level — the tools are not toys. Cursor, Claude Code, Devin, Codex, and OpenCode all produce real production code, and most serious teams now pay for two or three of them simultaneously. Cursor has $2B ARR and 360K paying users. Claude Code sits at 80.9% on SWE-bench Pro. Devin achieves a 67% PR merge rate on well-defined tickets. These are real tools with real strengths.

The problem this system solves is not single-session capability but **multi-week product build coherence** — and that's a distinct problem the ecosystem is still converging on.

| Approach | Where it genuinely works | The multi-week build limitation |
|---|---|---|
| **Raw Cursor / Claude Code agent mode** | Single-session features, refactors, debugging, well-scoped tickets. Best-in-class reasoning per turn. Most professional developers' daily driver. | No persistent plan. Each session starts cold at the repo state. Architectural decisions made in session 3 aren't remembered in session 17. The agent doesn't know what was deliberately deferred vs. not yet reached. Context windows are now large enough to load a codebase — but not to carry a *build's narrative* across weeks. |
| **Memory-bank / AGENTS.md / BRAIN.md systems** (vibe-brain, VibeKit, Devin Wiki) | A real advance over raw agents. Persistent project facts — stack decisions, gotchas, architectural rules — survive session boundaries by writing to checked-in markdown files the agent re-reads each turn. | Flat memory: no phase ordering, no dependency graph, no "done and verified" vs. "done-ish." The agent can act on a decision that's already been superseded, build things out of dependency order, or re-litigate architectural decisions without knowing they were settled. No independent verification layer — the memory is only as accurate as what the AI chose to write. Note rot is a documented production issue even in well-funded implementations. |
| **Fully autonomous agents** (Devin, OpenHands) | Excellent for well-scoped, ticket-style delegation — dependency upgrades, test backfill, framework migrations, PR fixes. 67% merge rate on defined tasks. Fire-and-forget. | Designed for tasks where you already know exactly what you want. Testing shows ~85% failure rate on complex or ambiguous multi-feature work. The problem of *what to build, in what order, with what dependencies* — the problem this system solves — is assumed to already be solved before Devin starts. |
| **Parallel agents without shared state** (Cursor multi-agent, Codex parallel runs) | Multiplied throughput on independent tasks. Running 5–8 parallel agents on a repo is now routine and genuinely fast. | Without a shared, externalized plan they all read-and-write, agents working from different context snapshots diverge on architecture. One agent's Phase 7 work silently breaks another's Phase 5 assumptions. Coordination is the user's problem, and most users solve it by *not* parallelizing, which loses the throughput benefit. |
| **Scaffold-and-iterate without verification gates** | Fast to prototype. Produces immediate visible output. Excellent for exploration, demos, and throwaway spikes. | A 2026 research report on production vibe-coding found: *"isolation rules, role checks, and background processing logic were usually absent unless explicitly specified — generated code optimizes for minimal functionality and lacks awareness of broader system context."* Without independent verification, "done" is the AI's opinion, not a testable fact. Cascading errors compound silently because nothing challenges the agent's own claim of completion. |

**This system is not competing with those tools — it runs on top of them.** You still use Cursor or Claude Code or Codex CLI as the agent executor. What the playbook adds is the layer that gives any of those agents a verified, phase-ordered, externalized plan to anchor against, so multi-week builds don't degrade into the failure modes above.

Concretely, the four properties no individual tool provides today but this system does:

1. **Plan anchoring across sessions.** `13-master-build-plan.md` is the single source of truth every agent loads at iteration start — not conversation history, not the agent's internal memory, not the IDE's cached context. A markdown file in the repo that survives session boundaries, agent swaps, IDE restarts, and parallel worktrees equally.
2. **Episodic memory with natural pruning.** Each phase's `Current implementation status` carries a structured micro-history of what was done, with causal ordering preserved. When the phase closes it freezes; future phases never re-read it. This solves the "memory grows stale as the project grows" problem that flat AGENTS.md files run into — token cost stays bounded by the active phase, not the total project size.
3. **Independent verification.** Every loop iteration runs the §24 testing ladder: TypeScript → ESLint → Vitest → integration → Playwright. "Done" means passing the gate, not the agent's opinion of done. An agent that mis-reports completion in the status block still fails an independent typecheck/lint/test layer — the same property §24 describes as *"the more independent the layers, the harder gaming becomes."*
4. **Explicit constraint specification via Genesis.** The 2026 research finding above — "absent unless explicitly specified" — is addressed structurally. Genesis produces explicit constraints before any code runs: phase-level `Do not:` items, cursor rules as always-on guardrails, cross-cutting concerns with their own phases that precede the features that consume them. The constraints aren't written after the bugs appear; they're the first output of the system.

This system is:

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

You open Cursor in your repo and send two words: **"Start Genesis."** If the playbook is already in `docs/playbook/`, the cursor rule picks it up and the AI opens `00-genesis-and-build-cadence.md` automatically. If starting from scratch, you pass the raw GitHub URL once — after that it's always just "Start Genesis." The AI asks you 15 questions — what the product does, target platform, stack preferences, monetization, scale expectations, hard constraints, a full screen and feature inventory (every page and its key interactions), your design system approach (component library, brand tokens, fidelity expectation), anything you've seen go wrong before, and a final open-ended catch-all for anything else (Notion docs, Figma links, competitor references, prior failed attempts, industry context). You answer. It locks the stack, writes three ADRs (stack choices, trust boundaries, repo topology), scaffolds the folder structure, writes 8–12 project-specific domain docs (product plan, DB schema, route map, design system, AI governance, etc.), generates the master build plan with every phase listed out (Phase 0 through however many you need), reviews the plan twice for dependency ordering and self-sufficiency, creates the empty watchlist and handoff register, and writes your cursor rules — all distilled from your answers and this playbook. You review the artifacts and approve. **Genesis then prints the exact loop prompt to copy-paste into Cursor, with a checklist of the human setup steps to do first.** You don't need to find or write the prompt — it's handed to you ready to go.

**Day 0 still — human setup window (~30–60 min, before Phase 0)**

Before queuing the first loop run, do the external wiring once: create service accounts (Supabase, Stripe, etc.), generate API keys and add them to `.env.local`, run `supabase init` / `supabase link`, configure MCP servers in `.cursor/mcp.json`, push the scaffold commit, confirm the testing ladder passes clean. This is the only time you need to interrupt the loop for external setup — the loop will log any mid-build blockers to the pre-shipping handoff register and keep moving. See `00-` §8.6 for the full checklist.

**Day 0 still — Phase 0 (Pre-Build Lock)**

You copy the loop prompt Genesis gave you and paste it into Cursor. Phase 0 is the confirmation step — no code yet. The AI re-reads the master plan, confirms every domain doc exists and is reviewed, confirms the MVP boundaries match your intent, confirms the human setup window is complete. If anything is off, it stops and surfaces it before any code lands. This is the last cheap moment to fix Genesis output.

**Day 1 — Phase 1 (repo, tooling, and design system foundation)**

Same loop prompt. Phase 1 is where actual code starts. It always covers two things together: the full monorepo and tooling setup (TypeScript strict, ESLint, Zod, env validation, test runner, CI wiring), and the complete design system foundation — Tailwind CSS v4, shadcn/ui copied into `packages/ui/`, design tokens wired in `packages/design-tokens/`, fonts via `next/font`, Lucide for icons. Both must land in Phase 1. Every subsequent feature phase builds on the design system — there's no "add styling later."

**Day 2 — Phase 2 (§21A Checkpoint 1 verification)**

The AI walks the Day-1 foundation checklist against the codebase, confirming cursor rules, tooling, and the design system are all in place before feature work begins.

**Day 3–N — Feature phases (auth, DB, screens, by phase)**

Same loop prompt. Each phase is one coherent capability: auth (Supabase auth, RLS, session handling), then database schema, then vertical feature slices (each screen or flow is its own phase). Every feature is built using `packages/ui/` primitives and design token values from day one — the UI is always usable and evaluable, never a naked skeleton. Anything the AI can't finish goes to the watchlist. Anything it needs you to provide goes to the pre-shipping handoff register.

**Near launch — polish phase + cleanup + pre-shipping**

If the project needs pixel-perfect polish (set during Genesis Q13), a dedicated visual polish phase runs after all features are stable: micro-interactions, hover/focus states, responsive edge cases, WCAG contrast, dark mode. Then the watchlist cleanup phase. Then the pre-shipping handoff register tells you exactly what a human still needs to provide: App Store credentials, production env vars, real-device test sign-off, legal review. Nothing is forgotten — it was logged the moment the AI hit the blocker.

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
| **Genesis** *(one-time per project)* | Human sends "Start Genesis." | Interactive intake (15 questions) → stack lock with ADRs → repo topology decision → folder scaffold → write all six tiers of artifacts (project-specific, not template) → human approval gate → ready for loop |
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
