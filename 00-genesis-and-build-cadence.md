<!-- Entry point for AI-driven project kickoff using this playbook. Read this first. -->

# Genesis & Build Cadence

**The operating system for AI-driven builds using this playbook.**

This document is the single entry point for starting a new project with the engineering playbook. It defines two things:

1. **Genesis** — a one-time, interactive intake the AI runs with the human to convert the playbook + the project requirements into a self-contained build plan.
2. **Build Cadence** — the autonomous loop the AI runs to execute that plan, with verification gates and structured deferral so context never drifts and progress never stalls.

The user-facing trigger is two words: **"Start Genesis."** That tells the AI to open this file and run the protocol below.

The rest of the playbook (`01-checkpoints-and-sanity.md` through `09-appendix.md`) defines *what good code looks like*. This file defines *how to build it with an AI agent without losing context or violating architecture*.

---

## For the Human

You only need to read this section before kicking off. Hand the rest to the AI.

### What this system does

It turns a conversation about your project into an exhaustive, phased build plan that an AI agent can execute autonomously while:

- Staying aligned with every rule in this playbook (without re-reading it every iteration).
- Never losing context across long sessions (state lives in markdown files, not chat history).
- Parking deferred work in a structured register instead of forgetting it.
- Logging items only a human can resolve (e.g. Apple Team ID) into a separate handoff doc.
- Committing to git at every meaningful step so progress is traceable and reversible.

### What you'll need before you start

- A repo (empty is fine) with `docs/` and `.cursor/` writable.
- A copy of this playbook at `docs/playbook/` in the new repo (snapshot, frozen at adoption time).
- A real conversation's worth of context about the product you're building: vision, scope, MVP boundaries, constraints, target platforms, monetization timing, scale expectations, any existing assets (Figma, brand, content), and anything you explicitly *don't* want built.

### How to start

**If the playbook is already in your repo** (`docs/playbook/` exists):

Open Cursor in the repo and send:

```
Start Genesis.
```

That's it. The cursor rule picks up the instruction, the AI opens this file, and the protocol begins.

**If you're starting from scratch** (no repo yet, or playbook not yet copied in):

Option A — reference the raw GitHub URL directly:

```
Read https://raw.githubusercontent.com/[your-org]/engineering-playbook/main/00-genesis-and-build-cadence.md and start Genesis.
```

Option B — clone the playbook first, then:

1. Copy this playbook folder into your new repo at `docs/playbook/`.
2. Open the repo in Cursor.
3. Send: `Start Genesis.`

**What the AI will do when it receives "Start Genesis":**

1. Open this file (or confirm it's already loaded).
2. Begin Step 1 of the Genesis Protocol — ask you the 15 intake questions.
3. Work through all Genesis steps, producing artifacts as it goes.
4. Present a summary of everything produced and wait for your approval.
5. Only after approval: suggest the first loop prompt from §8.

The AI will not start building until you explicitly approve the Genesis output. That's the human gate.

**After Genesis — running the loop:**

Queue one of the loop prompts from §8 in Cursor on repeat. The right variant depends on phase state (§8.5 decision table). Between phases, you review and re-queue. The loop is designed to be queued identically every iteration — it re-grounds itself from the docs each time, so the same prompt works throughout the build.

**Full step-by-step for new users:**

1. Copy this playbook into your new repo at `docs/playbook/`.
2. Open the repo in Cursor.
3. Send `Start Genesis.` — answer all 15 intake questions thoroughly.
4. Review the produced artifacts. Edit anything that doesn't match your intent.
5. Do the human setup window (§8.6) — external service accounts, API keys, MCP config, `.env`.
6. Send the Phase 0 loop prompt from §8. Review and repeat per phase.
7. At each phase boundary, review the commit, check the watchlist, approve before re-queuing.
8. Near launch, work through the pre-shipping handoff register (`15-…md`).

### What the AI will produce in Genesis

Genesis produces two categories of artifacts: **fixed** (always produced regardless of project) and **variable** (determined by the conversation).

**Fixed artifacts — always produced:**

```
docs/
  build/
    README.md                    ← index + "which doc to use" routing table
    13-master-build-plan.md      ← the phased execution sequence (central product of Genesis)
    14-code-quality-watchlist.md ← deferral register, starts empty
    15-pre-shipping-handoff.md   ← agent-blocked items, starts empty
  adr/
    001-stack.md
    002-trust-boundaries.md
    003-repo-topology.md
  playbook/                      ← the snapshot you copied in
.cursor/
  rules/
    001-core-project.mdc
    011-build-loop.mdc           ← enforces the loop invariants in this doc
CLAUDE.md
.cursorignore
```

**Variable artifacts — derived from your project:**

The numbered domain docs under `docs/build/` (e.g. `01-product-and-architecture-plan.md`, `05-database-schema-plan.md`, `07-design-system.md`) and the additional cursor rules (`002-typescript-standards.mdc` through `010-assets-cdn.mdc`) are **not a fixed template**. They're the AI's answer to the question: *"given what this project needs, which playbook sections are relevant, and what project-specific detail does each require?"*

A mobile-only app gets different docs than a web-only SaaS. A solo developer's internal tool gets tighter, fewer docs than a team product. A project with no payments doesn't get a payment-patterns doc. The conversation in Step 1 of Genesis drives the selection — the playbook is the source of content for each doc, but the project determines what exists at all.

The AI will tell you which docs it's creating and why, before writing them. If the set looks wrong, say so before approving. The master build plan (`13-…`) is always the last thing produced, because it depends on all the domain docs being finalized first.

### What you'll do day-to-day

- **Queue the loop prompt in Cursor** (large / medium / small variants in §8 of the AI Agent section — pick based on phase size).
- **Review PRs** as the loop ships them.
- **Approve checkpoint phases** (the ones tagged `(§21A Checkpoint N)` in the master plan).
- **Resolve pre-shipping items** when the loop logs them (Apple Team ID, store listing copy, real-device tests, etc.).
- **Re-baseline if the playbook updates** — see §9 of the AI Agent section.

### What you should not do

- Don't skip Genesis "to get started faster." Every shortcut here costs a week later.
- Don't paste the whole playbook into the loop prompt. The AI shouldn't re-read it every iteration (see §1.4 of the AI Agent section).
- Don't let the loop run past its stop gate without review. Each phase is a natural pause point.

---

## For the AI Agent

You are the agent operating this system. Everything below is your protocol. Read it top-to-bottom on first invocation; treat §3–§9 as reference material you return to during operation.

### 1. System Overview

#### 1.1 What you are building

Every project built with this playbook produces five tiers of artifacts. You create the first four during Genesis; you maintain them during the build loop.

| Tier | Artifact | Created in | Role at loop time |
|---|---|---|---|
| 0 | `.cursor/rules/*.mdc` | Genesis | Always-on conscience (auto-attached every iteration) |
| 1 | `docs/build/README.md` | Genesis | Index / table-of-contents router |
| 2 | `docs/build/01-…` through `12-…` | Genesis | Domain reference (consulted when phase doc cites them) |
| 3 | `docs/build/13-master-build-plan.md` + per-phase content | Genesis (skeleton), loop (status updates) | Primary spec; loaded every iteration |
| 4 | `docs/build/14-code-quality-watchlist.md` | Genesis (empty), loop (entries) | Parking lot for deferred work |
| 5 | `docs/build/15-pre-shipping-handoff.md` | Genesis (empty), loop (entries) | Agent-blocked items |

**Doc categories outside the build cadence** — these exist in projects but are governed by playbook §17 Documentation Governance, not by Genesis/loop mechanics:

- ADRs (`docs/adr/NNN-*.md`) capture stack and architectural decisions made during Genesis and afterwards.
- Off-loop task briefs (`docs/tasks/T-NNN-*.md`) for ad-hoc work that doesn't fit the master build plan (urgent fixes, spikes).
- Runbooks (`docs/runbooks/*.md`) for operational procedures (deploy, rollback, incident response).
- Postmortems (`docs/postmortems/YYYY-MM-DD-*.md`) for real incidents.
- Repo and package READMEs for orientation.

Every other `.md` file is a sprawl risk. The build loop's self-review rubric (§4.3) and the `001-core-project.mdc` cursor rule (Genesis Step 7) both enforce the taxonomy.

#### 1.2 Two operating modes

| Mode | Trigger | You do |
|---|---|---|
| **Genesis** | Human invokes "start a new project with this playbook" | Interactive intake → produce all artifacts → wait for approval |
| **Build Loop** | Human queues a loop prompt (§8) | Re-ground in master plan + active phase doc → pick next task → implement → verify → commit → update phase doc status → optionally update watchlist → stop at phase boundary |

#### 1.3 The core architectural principle

**Phase docs are self-contained at loop time.** All playbook content relevant to a phase is extracted, adapted to the project, and inlined into the phase doc during Genesis. The loop reads the phase doc, not the playbook.

The playbook remains available as a reference manual you can open on demand (§1.4) — but it is never preloaded into the loop's context window, and the loop never re-reads it in bulk.

#### 1.4 Playbook consultation rules — read carefully

| Behavior | Status |
|---|---|
| Phase doc preloaded into loop context | **Required** every iteration |
| Cursor rules auto-attached | **Required** every iteration |
| Bulk-loading the whole playbook | **Forbidden** — causes token bloat and degraded reasoning |
| Opening a specific playbook section on demand | **Allowed and encouraged** when one of the triggers below fires |
| Citing a playbook rule without opening the section in this turn | **Forbidden** — that's hallucination, not reference |
| Resolving a gap by opening the playbook and not writing the resolution back into the phase doc | **Forbidden** — same gap must not require re-consultation next iteration |

**Four triggers for opening a specific playbook section on demand:**

1. The phase doc is silent on a question that arose mid-task.
2. A cursor rule cites a section and you need the full reasoning before applying it.
3. You're about to make a non-trivial architectural choice the phase doc didn't anticipate.
4. You are about to create a new `.md` file and the destination is not obvious — open playbook §17 Documentation Governance to route it through the taxonomy. If no row fits, write an ADR proposing the extension instead of inventing a new doc location.

**When you do open a section on demand:**

1. Use the `Read` tool to open the specific file (e.g. `docs/playbook/05-security-observability.md`).
2. Apply the rule to the current work.
3. Write the resolution back into the phase doc as a new step or `Do not` item, with a provenance marker: `<!-- inlined from playbook §X during Phase N mid-loop consultation YYYY-MM-DD -->`.
4. Commit the phase doc update alongside the code change.

This is the consultation-writeback rule. The phase doc accumulates targeted, recorded just-in-time additions from the playbook — but only the parts that actually came up. Token cost stays bounded; the playbook stays a reference.

---

### 2. Genesis Protocol

When the human invokes the kickoff prompt, run this protocol top-to-bottom. Do not begin step N+1 until step N is complete and recorded.

#### Step 0 — Opening statement (send this before asking any questions)

Before asking anything, send the following message verbatim so the human knows what they're in:

```
Genesis started.

I'm going to ask you 15 questions about your project. Your answers drive everything — the more detail you give now, the less the build loop has to guess or ask later. Questions 12 and 13 ask about your screens/features and design approach in detail, so take your time on those. There are no wrong answers; "I don't know yet" and "skip / decide later" are valid for anything you're unsure of.

After the questions I'll produce:
  - A phased master build plan (your project broken into numbered phases)
  - Project-specific domain docs (DB schema, product plan, design system, AI governance, etc.)
  - Cursor rules tailored to your stack and constraints
  - Architecture Decision Records for key choices
  - Empty watchlist and pre-shipping handoff registers

Nothing gets built until you review and approve all of that. You can edit anything before giving the go-ahead.

Ready when you are — let's start with question 1.
```

Then begin Step 1 immediately without waiting for a reply.

#### Step 1 — Project Context Capture (interactive)

Ask the human these questions in order. Do not assume answers. Do not proceed until every question has a clear answer or an explicit "skip / decide later."

1. **Product vision** — one paragraph: what does this product do, for whom, and what is its single differentiator?
2. **MVP scope** — what is in / what is explicitly out / what is future-phase?
3. **Target platforms** — web only / mobile only / web + mobile / desktop / other?
4. **Deployment expectations** — serverless (Vercel/CF Workers) / long-running (Fly.io/Render) / self-hosted / hybrid?
5. **Backend choice** — BaaS (Supabase / Firebase) / custom server / hybrid?
6. **Monetization timing** — never / day-1 / post-MVP / post-PMF? (Determines whether playbook §20 content needs to be inlined now or deferred.)
7. **Scale expectations** — alpha (10s of users) / launch (1K) / scale (10K-100K) / hyperscale (100K+)? (Determines which §21A checkpoints become embedded phases.)
8. **AI features in product** — none / one feature (e.g. chat) / AI-native? (Determines whether playbook §14A LLM evals content gets inlined.)
9. **Hard constraints** — compliance (GDPR / HIPAA / PCI / SOC2), existing infrastructure to integrate with, language/framework constraints, anything non-negotiable.
10. **Team profile** — solo dev / small team / hired contractors / multiple AI agents in parallel? (Determines tightness of cursor rules and ADR formality.)
11. **Existing inputs** — Figma files, brand assets, content sources, API contracts, prior code, design references. If Figma files exist, ask the human to share links or exported assets — these become source material for the design system domain doc and screen map.
12. **Screen and feature inventory** — list every screen/page the product needs, even roughly. For each screen: what are the primary actions a user takes on it? What data does it display? What are the key interactive elements (forms, modals, tables, carousels, maps, etc.)? This doesn't need to be pixel-precise — a bullet list is enough. This drives the screen map domain doc and ensures no feature is missing from the build plan. If the human has a Figma file or written spec, that counts — they can paste a list or describe it conversationally.
    - *Why this matters:* Without a screen inventory, the AI guesses which features exist and the build plan will have gaps. Gaps discovered mid-build become unplanned phases that disrupt the loop.
13. **Design system and styling approach** — three sub-questions:
    - *Component library:* Start from a library (shadcn/ui, Radix + custom, MUI, Mantine, Ant Design) or build components from scratch? The playbook recommends shadcn/ui (headless, composable, Tailwind-based, 2026 standard for React) unless there's a reason not to. Are there any existing brand constraints that override this?
    - *Design fidelity expectation:* MVP/functional (structured and usable, not pixel-perfect), polished product (pixel-perfect, micro-interactions, motion), or somewhere in between? This determines whether a dedicated visual polish phase is warranted and how detailed the design system domain doc needs to be.
    - *Brand tokens:* Are brand colors, typography, and spacing already defined (Figma variables, a style guide, a brand kit)? Or does the AI propose a palette based on the product vision? This becomes the `packages/design-tokens/` setup in Phase 1.
14. **Anti-scope** — anything the human has seen done badly before and explicitly wants prevented (these become `Do not` items and anti-pattern cursor rules).
15. **Everything else** — any other context that doesn't fit the above: competitor references, industry-specific domain knowledge, previous failed attempts and why they failed, links to Notion docs / Google Docs / specs / wireframes / prior codebases, anything the human thinks the AI should know before planning. There is no wrong answer here. If the human says "nothing else", move on. If they dump a wall of text, extract the relevant signals into the product plan and flag anything that changes answers to questions 1–14.

Record the answers in `docs/build/01-product-and-architecture-plan.md` as you go. Do not summarize — capture verbatim where possible.

#### Step 2 — Stack Lock (playbook §9 Appendix)

Open `docs/playbook/09-appendix.md`. For each row in the default stack table, decide:

- **Adopt** — use the default. Note in ADR-001.
- **Substitute** — replace with reasoning. Note both choice and reasoning in ADR-001.
- **Defer** — not needed for this project. Note in ADR-001.

Produce `docs/adr/001-stack.md` with the full table marked Adopt / Substitute / Defer per row.

#### Step 3 — Trust Boundary Enumeration (playbook §1)

Open `docs/playbook/02-foundation.md` and read §1 (Core Philosophy → Trust boundaries).

Enumerate the four boundaries for this specific product:

- **HTTP edge** — what's the network surface? (Web routes, mobile API, webhooks, public read endpoints.)
- **DB edge** — what reads/writes the database? (BaaS client, ORM, raw SQL, RPC.)
- **AI edge** — only if this product uses LLMs. Where do prompts get assembled? Where do tools execute?
- **Third-party edge** — every external API the product calls (payments, email, SMS, storage, analytics, error capture).

Produce `docs/adr/002-trust-boundaries.md` listing the four boundaries with the specific entry points for each.

#### Step 4 — Repo Topology (playbook §2)

Open `docs/playbook/02-foundation.md` and read §2.

Decide:
- Modular monolith vs. microservices vs. distributed monolith (default: modular monolith — playbook §2).
- Monorepo tool (pnpm + Turborepo unless ADR-001 substituted).
- Which `apps/` exist (web / mobile / workers / admin).
- Which `packages/` exist (validation / database / ui / ai / observability / design-tokens / assets / config — only those that apply).
- Which `tooling/` configs are shared.

Produce `docs/adr/003-repo-topology.md`.

Then scaffold the folder structure on disk. The required structure:

```
apps/
packages/
tooling/
docs/
  build/         (Genesis populates with 00-, 13-, 14-, 15-, plus domain docs)
  adr/           (Genesis populates with 001-, 002-, 003- minimum)
  playbook/      (snapshot already here)
.cursor/
  rules/
```

Conditionally scaffold these folders (empty) if Step 1 indicated the project will use them, per playbook §17 Documentation Governance:

```
docs/
  tasks/         (only if off-loop work briefs are expected — usually yes)
  runbooks/      (only if the project ships ops procedures — usually yes for anything with deploys)
  postmortems/   (only if the project will have real incidents — usually yes post-launch)
```

If a folder is created, drop a one-line `README.md` inside it explaining what kind of doc belongs there and what the filename convention is (per §17 taxonomy). This makes the policy visible from inside the folder, not just in the playbook.

#### Step 5 — Domain Doc Enumeration

Based on the project context captured in Step 1, decide which domain docs this project needs. There is no fixed list — the set is determined by the project.

**How to decide what to create:**

For each of the questions below, if the answer is yes, create the corresponding doc by opening the cited playbook sections, extracting what applies to *this* project, and combining it with the project-specific information from Step 1.

| Question | If yes, create | Open these playbook sections |
|---|---|---|
| Does the project need a product vision, architecture, MVP scope, and technical roadmap documented? | `01-product-and-architecture-plan.md` | `02-foundation.md` §1–§2, `09-appendix.md` |
| Does the team need to understand *why* the chosen stack was chosen (for onboarding and AI alignment)? | `02-stack-and-build-system-review.md` | `02-foundation.md` §2, `09-appendix.md` |
| Will an AI agent be building or modifying this codebase? | `03-ai-development-governance.md` | `02-foundation.md` §1 §5, `06-delivery-testing-ai-rules.md` §16 §18, `07-standards-integrations.md` §17 §22 |
| Does the project have CI/CD, branching strategy, environments, or a deployment workflow to document? | `04-development-workflow.md` | `06-delivery-testing-ai-rules.md` §15 §16 |
| Does the project have a database with tables, schema, RLS, migrations, or seed data to plan? | `05-database-schema-plan.md` | `03-data-caching-assets.md` §7 |
| Does the project have user-facing routes or screens that need to be mapped, including empty/error/loading states? | `06-screen-map.md` | `07-standards-integrations.md` §17 (routes as single source of truth) |
| Does the project have a UI with design tokens, component strategy, or styling constraints? | `07-design-system.md` | `07-standards-integrations.md` §17 (design tokens) |
| Does the project have media assets (images, icons, brand materials) with CDN, storage, or naming rules? | `08-asset-system.md` | `03-data-caching-assets.md` §8A |
| Does the project have client-side data fetching, caching, or offline behavior to plan? | `09-caching-strategy.md` | `03-data-caching-assets.md` §8 §11 |
| Will this project launch to real users? | `10-mvp-readiness-checklist.md` | `01-checkpoints-and-sanity.md` §21A Checkpoints 1–2 distilled for this project |
| Does the project need structured logging, error monitoring, analytics, or observability? | `11-observability-system.md` | `05-security-observability.md` §13, `06-delivery-testing-ai-rules.md` §14A (if AI), `07-standards-integrations.md` §19 resilience |
| Does the project use LLMs, have AI-powered features, or ingest data from external sources? | `12-ai-and-ingestion-strategy.md` | `06-delivery-testing-ai-rules.md` §14A, `05-security-observability.md` §12 LLM Product Security |
| Does the project need domain-specific planning not covered above? | Create additional numbered docs as needed | Open the relevant playbook sections |

There may also be project-specific domains that have no playbook analog (e.g. `13-crawler-and-ingestion-provider-strategy.md` in a project that scrapes external data). Create those too — they'll source entirely from the Step 1 conversation and domain knowledge, not from the playbook.

**Before writing any doc, tell the human which docs you're planning to create and why.** They may remove some, rename some, or add others based on their knowledge of the project. This is the last decision point before Genesis becomes expensive to redo.

**For each doc you create:**

1. Open the cited playbook sections with the `Read` tool. Do not summarize from memory.
2. Extract the rules, anti-patterns, decision trees, and templates that apply to *this* project. Omit playbook content that doesn't apply.
3. Combine with the project specifics from Step 1 — schema details, product decisions, MVP boundaries, team constraints.
4. Write the result as a project-specific, self-contained document. A developer or agent reading this doc should not need to open the playbook to understand what to do.
5. Add provenance markers wherever playbook content was inlined: `<!-- inlined from playbook §X -->`.

The numbered suffix on each doc (01-, 02-, etc.) reflects the recommended reading order for an agent starting fresh, not a fixed schema. Number them so the most foundational docs come first.

The master build plan (`13-master-build-plan.md`) is always the last doc produced in this step, because it depends on all the domain docs being finalized first — its phases will cite them as Source docs.

#### Step 6 — Master Build Plan (`13-master-build-plan.md`)

This is the single most important Genesis output. It's the phased execution sequence the loop will follow.

Produce phases using the template in §3 of this protocol. Phase enumeration heuristic:

Note: by the time the loop reaches Phase 0, cursor rules, ADRs, domain docs, the watchlist, the pre-shipping handoff, and `CLAUDE.md` already exist on disk — Genesis Steps 5–10 produced them. The master build plan starts after Genesis, not as part of it.

- **Phase 0** — Pre-Build Lock (confirm Genesis outputs are in place, MVP boundaries clear, all domain docs reviewed by the human).
- **Phase 1** — Repository, Tooling, and Design System Foundation. Two concerns always go in Phase 1 together:
  1. *Tooling:* monorepo, TS strict, ESLint, Zod, env validation, test runner wiring (§21A Checkpoint 1 items).
  2. *Design system:* `packages/design-tokens/` with brand colors, typography scale, spacing scale. Component library installed and configured (shadcn/ui by default, or the library chosen in Genesis question 13). Base layout components (shell, nav, page wrapper). This must be done in Phase 1, not later — all feature phases build on top of it. A design system retrofitted after 15 screens is a multi-week project.
- **Phase 2** — `(§21A Checkpoint 1)` Day-1 Foundation Verification. Walk the §21A Checkpoint 1 list against the codebase; this is where Genesis's cursor rules and Phase 1's tooling + design system get validated together.
- **Phase 3 onwards** — Vertical slices of product work. Each phase is one coherent capability (e.g. "Web admin MVP", "Mobile app shell", "Event detail screen", "URL ingestion to draft", etc.). **Styling approach for feature phases:** build each feature using real components from the design system — correct layout, correct token values, correct semantic HTML, usable and evaluable from day one. Do not use placeholder/unstyled markup with the intent to style it later. Do not do pixel-perfect polish mid-feature-build — that comes in the polish phase below.
- **Phase N — Visual Polish Pass** *(only if design fidelity expectation from Genesis question 13 is "polished product" or "somewhere in between")*.* Walk every screen: micro-interactions, hover/focus/active states, focus rings (WCAG), responsive edge cases, dark mode if applicable, motion/animation, empty states, loading skeletons. This phase comes after all features are built and verified, so the AI has a stable target. It's fast because the structure is already correct — only visual details remain.
- **Phase N+1 — `(§21A Checkpoint 2)` Pre-launch Readiness.** Walk Checkpoint 2 against codebase.
- **Phase N+2 — Code Quality Watchlist Cleanup.** Run through `14-…` and resolve triggered items.
- **Phase N+3 — Public Launch Preparation.** Real-device verification, seed data, smoke tests.
- **Phase N+4 — `(§21A Checkpoint 3)` First Paying User Readiness** *(only if monetization is in MVP scope; else defer to post-launch)*.
- **Phase N+5 — `(§21A Checkpoint 4)` Scale and Reliability** *(post-launch)*.

Each phase doc field must be populated such that the loop, reading only that phase doc + cursor rules, has enough information to complete the phase without opening the playbook.

If you find yourself wanting to write "see playbook §X" instead of inlining the content, **inline it instead.** The exception: if the same content appears in many phases (e.g. file-size limits, anti-patterns), put it in cursor rules and reference the rule, not the playbook.

#### Step 6a — Build Plan Review Pass 1: Dependency and Ordering Audit

Do not proceed to Step 7 until this pass is complete and all issues are fixed.

Re-read the entire master build plan as if you are the loop, starting cold at Phase 0. For every phase, ask:

1. **Dependency satisfied?** Does every tool, package, schema, auth model, env var, API contract, or design token this phase references actually exist by the time this phase runs? If Phase 7 uses a DB table that Phase 3 was supposed to create — does Phase 3's `Steps:` actually create it?
2. **Phase ordering correct?** Could any two adjacent phases be swapped without breaking anything? If yes, re-order so dependencies always precede dependents.
3. **Scope leak?** Does any phase do work that belongs to another phase? Each phase should be one coherent capability, not a grab-bag.
4. **Gap detection — Step 1 coverage?** Go back to the Step 1 answers. Is every product requirement, hard constraint, anti-scope item, and input (Figma, brand, API contract, prior code) reflected somewhere in the phases? List any orphaned requirements and add phases or steps to cover them.
5. **Gap detection — playbook coverage?** Every significant playbook rule that applies to this project should appear either as a phase step, a verification item, or a cursor rule. Scan the domain docs produced in Step 5. Is every rule they contain accounted for in the plan?
6. **§21A Checkpoint alignment?** The checkpoints in `01-checkpoints-and-sanity.md` are milestone gates. Confirm each applicable checkpoint is embedded as a dedicated verification phase, not buried inside a feature phase.

Fix all issues found before moving on. If a fix changes a phase significantly, re-read that phase's neighbours to ensure the fix didn't break their ordering or dependencies.

#### Step 6b — Build Plan Review Pass 2: Self-Sufficiency and Consistency Audit

This pass checks that the plan is internally consistent and that the loop can execute it without external help.

For every phase, ask:

1. **Self-sufficient?** Could a fresh AI agent, reading only this phase doc and the cursor rules, complete the phase without opening the playbook, asking clarifying questions, or improvising architecture? If not, what is missing? Inline it.
2. **Source docs accurate?** Does the `Source docs:` list for each phase cite the domain docs that phase actually needs — not too many (bloat), not too few (gaps)?
3. **Verification concrete?** Is every `Verification:` item a specific, runnable check (a test name, a CLI command, a manual step with expected output)? "Verify auth works" is not concrete. "All tests in `auth.test.ts` pass; unauthenticated request to `/api/events` returns 401" is.
4. **Do not items specific?** Are the `Do not:` items specific enough to prevent a real mistake? "Do not add features" is useless. "Do not create user-facing UI in this phase — backend only" is enforceable.
5. **No playbook citations in phase docs?** Phase docs must not contain "see playbook §X" — the content must be inlined. Check every phase.
6. **Current implementation status blank?** Every phase's `Current implementation status:` must be empty at Genesis. It is written by the loop, not Genesis.

After both passes, if more than three phases required significant changes, do a final quick re-read of the full plan in order to confirm the changes haven't introduced new ordering or dependency issues.

Only when both passes produce no issues: proceed to Step 7.

#### Step 7 — Cursor Rule Distillation

Two rules are always produced regardless of project:

| Rule file | Content | Activation |
|---|---|---|
| `001-core-project.mdc` | Project scope, approved stack, module boundaries, absolute prohibitions, anti-scope from Step 1, **and the doc-governance bullet** (see below). Distilled from playbook §1, §17 Documentation Governance, and Genesis intake. | `alwaysApply: true` |
| `011-build-loop.mdc` | The build loop invariants verbatim. See template below. | `alwaysApply: true` |

**The doc-governance bullet that must appear in `001-core-project.mdc`** (paste verbatim, then optionally tighten with project-specific banned filenames the human flagged in Step 1's anti-scope):

```markdown
## Documentation Governance (see playbook §17)

- BEFORE creating any new `.md` file, route it through the taxonomy in `docs/playbook/07-standards-integrations.md` §17 Documentation Governance.
- NEVER create `TODO.md`, `STATUS.md`, `BACKLOG.md`, `NOTES.md`, `SCRATCHPAD.md`, `WIP.md`, `PROGRESS.md`, `IDEAS.md`, or meeting-notes files anywhere in the repo. Use the watchlist (`docs/build/14-code-quality-watchlist.md`), a new phase in the master build plan, or an ADR — whichever the §17 routing table directs.
- NEVER create `.md` files inside `apps/*/src/`, `packages/*/src/`, or any source folder. Use TSDoc/JSDoc adjacent to the code.
- NEVER create investigation writeups (`<feature>-investigation.md`, `debug-notes.md`, etc.) — the investigation is throwaway; its outputs become an ADR, a runbook, a postmortem, or an update to the relevant domain doc.
- NEVER duplicate documentation: the same concept must not be documented in two places. Pick one canonical location; the other becomes a one-line link.
- WHEN the documentation need doesn't fit the §17 taxonomy, stop and propose an ADR to extend the taxonomy. Do not invent a new doc category by writing the file.
- WHEN adding a new package or app folder, create one `README.md` inside it (max ~30 lines, one paragraph + link to the relevant `docs/build/0X-…md`). Do not create nested or competing READMEs.
```

The remaining rule files are determined by the project, parallel to the domain doc decision in Step 5. For each concern the project has, ask: *"Would an AI agent working in these files make the same mistake repeatedly without a rule?"* If yes, create the rule.

**Decision table — create a rule file if the project has that concern:**

| Project concern | Likely rule file | Distill from playbook | Glob |
|---|---|---|---|
| TypeScript strict mode, Zod, no `any`, file size, import standards | `002-typescript-standards.mdc` | §3, §17 | `**/*.{ts,tsx}` |
| Web framework conventions (Server Components, state, mutations, route ownership) | `003-frontend.mdc` | §4, §8, §6 | `apps/web/**/*.{ts,tsx}` or equivalent |
| Mobile-specific conventions (RN, Expo, bundle size, startup time) | `004-mobile.mdc` | §2, §8 | `apps/mobile/**/*.{ts,tsx}` |
| Database access patterns, RLS, migrations, no raw client calls from UI | `005-database.mdc` | §7 | `packages/database/**/*`, migrations glob |
| Backend layering, auth at boundaries, resilience patterns, env validation | `006-backend.mdc` | §6, §12, §19 | API route glob |
| Testing pyramid, what to write first, mock boundaries, no `expect(true)` | `007-testing-quality.mdc` | §14, §14A, §24 | `**/*.{test,spec}.{ts,tsx}` |
| Design tokens, no hardcoded colors/spacing, component primitives | `008-styling-design.mdc` | §17 design tokens, project design system | `**/*.{tsx,css}` |
| Cache layers, query keys, stale times, observability log levels | `009-observability-caching.mdc` | §8, §13 | Observability and hook files |
| CDN asset taxonomy, no inline URLs, bucket model, staging bypass | `010-assets-cdn.mdc` | §8A — copy the rule body from playbook §16 verbatim | `alwaysApply: true` (per playbook §16) |
| Anything the project's anti-scope or recurring AI mistakes require | Additional numbered rules | Relevant playbook sections + project specifics | As appropriate |

Do not create a rule file for a concern the project doesn't have. A mobile rule for a web-only project is dead weight that dilutes the always-on rules. Rules are expensive when they're wrong; they're free when they're right.

**How to write each rule file:**

For each rule file you've decided to create:

1. **Open the cited playbook sections** with the `Read` tool. Read the actual §N content — don't summarize from memory.
2. **Extract enforceable rules only.** The playbook mixes rationale prose ("Why now: …") with actual rules. Rationale goes into domain docs (Step 5); only the actionable rules go into the cursor rule. Strip the reasoning — compress it to a one-sentence "do" or "do not" bullet.
3. **Layer in project specifics from Step 1.** Every rule file should have project-specific content — the actual package paths for this repo, the specific prohibited patterns that came up in the anti-scope conversation, the concrete glob that matches this project's folder structure. A generic rule is less effective than a project-tuned one.
4. **Reference the project's domain docs, not the playbook.** Rules say `See docs/build/05-database-schema-plan.md` — not `See playbook §7`. The domain doc is self-contained and available at loop time. The playbook is not.
5. **Write the file to disk** in the correct `.mdc` format with YAML frontmatter. Actually produce the file — do not describe it or defer it to Phase 2.
6. **Self-sufficiency check:** could an AI agent following only this rule file's bullets, without opening any other document, make the right decision in every case the rule covers? If not, add the missing context or a cite to the right domain doc.

**Rule authoring principles** (distilled from playbook §16):

- Short, declarative bullets. One sentence per rule.
- "Do not" rules are more reliable than "do" rules — the AI is biased toward action; banning is more effective than encouraging.
- Cite the project domain doc, not the playbook.
- Use numbers, not vague guidance: "aim < 250 lines; review > 400; split > 600" beats "keep files small."
- `alwaysApply: true` only for rules whose violation class can appear in any file regardless of path. Everything else uses `globs:` to keep context focused — a bloated always-on rule degrades all other rules around it.
- Update rules when conventions evolve. A stale rule actively misleads.

**`011-build-loop.mdc` template** (write this verbatim into the rules folder):

```markdown
---
description: Build loop invariants — re-grounding, consultation, writeback, commit cadence
globs: ""
alwaysApply: true
---

## Build Loop Invariants (see docs/build/13-master-build-plan.md and docs/playbook/00-genesis-and-build-cadence.md)

- WHEN the human sends "Start Genesis" (or "start genesis", case-insensitive): open `docs/playbook/00-genesis-and-build-cadence.md`, run the Genesis Protocol from Step 1, and do not begin any implementation until the human explicitly approves the produced artifacts.

- ALWAYS re-ground at the start of every loop iteration: read `docs/build/13-master-build-plan.md` and the active phase doc fully. Do not skim. Do not proceed without identifying the next coherent task.

- NEVER bulk-load the playbook (`docs/playbook/*`) into context during the loop. Open specific sections on demand only.

- NEVER cite a playbook rule without opening the relevant section in the same turn. Read-don't-guess. If you cannot open the section, do not cite it.

- WHEN you open a playbook section to resolve a gap, write the resolution back into the active phase doc as a new step or `Do not` item with provenance: `<!-- inlined from playbook §X during Phase N mid-loop consultation YYYY-MM-DD -->`. Commit the phase doc update alongside the code change.

- NEVER guess at the contents of any referenced file. Open it. This applies to phase docs, ADRs, the watchlist, the pre-shipping handoff, and the playbook.

- WHEN a task hits something the active phase doc did not anticipate AND the playbook does not cover it, stop and ask the human. Do not invent architecture.

- WHEN you defer work, write a watchlist entry in `docs/build/14-code-quality-watchlist.md` using the Current state / When to clean up / Likely future task template. Do not leave deferred work in chat history or stale comments.

- WHEN you encounter an item only a human can resolve (developer-account values, real-device tests, store listing assets, production URLs), log it in `docs/build/15-pre-shipping-handoff.md`. Do not block the loop on it.

- COMMIT to local git after every meaningful completed step. The commit message references the phase and task.

- STOP at phase boundaries. When the active phase's verification list passes, do not auto-advance. The human queues the next loop run.
```

#### Step 8 — `CLAUDE.md`

Produce a condensed multi-agent compatibility layer (~100-200 lines) that mirrors the contents of `001-core-project.mdc` and the most critical patterns from `011-build-loop.mdc`. This serves Claude Code, parallel agents, or any other agent that doesn't natively read `.mdc` files.

Keep it factually consistent with the Cursor rules. Inconsistency between `CLAUDE.md` and `.cursor/rules/` produces inconsistent agent behavior.

#### Step 9 — `.cursorignore` / `.aiexclude`

Per playbook §16 AI session hygiene. Minimum exclusions:

```
.env*
**/*.pem
**/*.key
secrets/
keys/
**/service-account*.json
**/.supabase/
dist/
build/
*.sqlite
*.dump
node_modules/
```

#### Step 10 — Initial Watchlist and Pre-Shipping Handoff

Create `docs/build/14-code-quality-watchlist.md` with the entry template (see §5 of this protocol) and an empty `## Current Watchlist` section.

Create `docs/build/15-pre-shipping-handoff.md` with category headers but no entries:

```markdown
# Pre-Shipping Handoff

Items the human must resolve before public launch. The loop logs into this file
whenever it encounters something it cannot autonomously complete.

## Production Domain and Public URLs

(empty)

## Developer Account Values

(empty)

## Store Listing Decisions

(empty)

## Real-Device Verification

(empty)

## Legal and Support Approval

(empty)

## Other

(empty)
```

#### Step 11 — Genesis Verification

Before handing back to the human, run two sub-steps: a final holistic consistency check, then the artifact checklist. Do not mark Genesis complete until both pass.

**Step 11a — Final cross-artifact consistency check**

Now that all artifacts exist (domain docs, master build plan, cursor rules, ADRs), check that they are mutually consistent:

1. **Plan ↔ domain docs:** Does every domain doc cited in a phase's `Source docs:` actually exist? Does every domain doc that exists get cited by at least one phase?
2. **Plan ↔ cursor rules:** Do any cursor rules prohibit something the build plan asks the loop to do? Do any cursor rules require something the build plan never sets up?
3. **Plan ↔ ADRs:** Does the master build plan's stack match `docs/adr/001-stack.md`? Does the repo structure described in phases match `docs/adr/003-repo-topology.md`?
4. **Cursor rules ↔ domain docs:** Do the cursor rules cite `docs/build/…` paths (not `docs/playbook/…`)? Do those paths actually exist?
5. **Step 1 answers ↔ everything:** Re-read the Step 1 answers one more time. Is there anything the human said that is not reflected in the artifacts? Any hard constraint that has no corresponding cursor rule or `Do not:` item? Any anti-scope item that could be accidentally built if the loop doesn't have a guard against it?

Fix any inconsistencies found before proceeding to the artifact checklist.

**Step 11b — Artifact checklist**

Verify every item below. Do not mark Genesis complete until all pass.

**Domain docs:**
- [ ] Every domain doc exists on disk as a real file (not a plan or description of a file).
- [ ] Every domain doc is project-specific — it combines playbook content with the project's own requirements from Step 1. A doc that reads identically for two different projects is under-specified.
- [ ] Every domain doc is self-contained: a developer or agent reading it should not need to open the playbook.
- [ ] Every domain doc has provenance markers on inlined playbook content.

**Master build plan:**
- [ ] Every phase doc field is populated (Goal / Source docs / Steps / Current implementation status / Verification / Do not).
- [ ] Every phase's `Verification:` items are concrete and testable — not "make sure it works."
- [ ] Every phase's `Steps:` are granular enough that the loop can execute one step per commit without ambiguity.
- [ ] Every `Do not:` item is either project-specific or a distilled playbook anti-pattern with provenance.
- [ ] Every §21A Checkpoint that applies to this project is embedded as a dedicated verification phase with its checklist inlined and adapted.
- [ ] The master build plan would work as the loop's only non-rules input: a fresh agent reading the plan + cursor rules could complete every phase without opening the playbook.

**Cursor rules:**
- [ ] Every rule file exists on disk as a real `.mdc` file with correct YAML frontmatter — not a plan for a file.
- [ ] Every rule file contains project-specific content (actual paths, actual prohibited patterns, concrete globs matching this repo's structure) — not just generic playbook rewrites.
- [ ] Every rule file passes the self-sufficiency check: an AI working in the glob'd files can follow every bullet without opening any other document.
- [ ] Rules cite domain docs (`docs/build/…`), not the playbook.
- [ ] No rule file exists for a concern this project doesn't have.
- [ ] `011-build-loop.mdc` is verbatim from the template in Step 7.

**Infrastructure:**
- [ ] `docs/build/README.md` index lists every doc and explains which to open when.
- [ ] `.cursorignore` / `.aiexclude` is committed with minimum exclusions.
- [ ] ADRs 001 (stack), 002 (trust boundaries), 003 (repo topology) are written.
- [ ] `CLAUDE.md` is consistent with `001-core-project.mdc` and `011-build-loop.mdc`.
- [ ] `14-code-quality-watchlist.md` exists with the entry template and an empty `## Current Watchlist` section.
- [ ] `15-pre-shipping-handoff.md` exists with category headers and empty sections.

#### Step 12 — Hand Back

Print the following summary to the human. Fill in the bracketed values from the actual Genesis output. Then stop completely — do not begin any implementation, do not start Phase 0, do not make any file changes until the human explicitly says to proceed.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Genesis complete. Awaiting your approval.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Artifacts produced:
  docs/build/README.md                     ← index / router
  docs/build/01-… through [N]-….md         ← domain docs ([N] docs)
  docs/build/13-master-build-plan.md       ← [X] phases, Phase 0 through Phase [Y]
  docs/build/14-code-quality-watchlist.md  ← empty, ready for loop entries
  docs/build/15-pre-shipping-handoff.md    ← empty, ready for loop entries
  docs/adr/001-stack.md
  docs/adr/002-trust-boundaries.md
  docs/adr/003-repo-topology.md
  .cursor/rules/001-core-project.mdc … 011-build-loop.mdc
  CLAUDE.md
  .cursorignore

Before starting the loop, please review:
  1. docs/build/13-master-build-plan.md — does the phase order match your expectations?
  2. docs/build/01-product-and-architecture-plan.md — is the product vision captured correctly?
  3. .cursor/rules/001-core-project.mdc — are the prohibitions and anti-scope right?
  If anything is wrong, tell me now and I'll fix it before the loop starts.

Human setup window (do this before queuing the first loop prompt):
  □ Create external service accounts (see docs/adr/001-stack.md for the list)
  □ Add API keys / secrets to .env.local
  □ Run: supabase init && supabase link (or equivalent)
  □ Configure MCP servers in .cursor/mcp.json
  □ Confirm the testing ladder passes clean:
      pnpm typecheck && pnpm lint && pnpm test
  Full checklist: docs/playbook/00-genesis-and-build-cadence.md §8.6

When you're ready to start building, queue this prompt in Cursor on repeat:

  ┌─────────────────────────────────────────────────────────────────────────────┐
  │ Check the current repo state against docs/build/13-master-build-plan.md — │
  │ this is the primary controller. Study the active phase doc carefully: what  │
  │ has already been done, what is next — make sure we are aligned and not      │
  │ diverging in any way. Verify alignment with cursor rules. Check             │
  │ docs/build/14-code-quality-watchlist.md. Scan                              │
  │ docs/build/15-pre-shipping-handoff.md for any now-unblocked items.         │
  │                                                                             │
  │ Check docs/build/14-code-quality-watchlist.md and continue the next        │
  │ coherent phase-aligned task or batch of tasks. We want decent progress      │
  │ but not multiple unrelated things at the same time.                         │
  │                                                                             │
  │ Adhere to all coding practices and existing patterns. After each task,      │
  │ double-check for changed code smells, documentation regression, stale       │
  │ comments, or other errors. Check relevant logic and user flows. Check       │
  │ performance, security, DRY, KISS, YAGNI, separation of concerns,           │
  │ consistency, parallel code, maintainability, race conditions, divergence.   │
  │                                                                             │
  │ Make sure all relevant new tests pass. Do not stop for missing API keys;    │
  │ implement env-based wiring and no-op / dev-safe fallbacks.                  │
  │                                                                             │
  │ Any left-over optimizations → docs/build/14-code-quality-watchlist.md      │
  │ Any agent-blocked items → docs/build/15-pre-shipping-handoff.md            │
  │                                                                             │
  │ Create a local git commit after every big completed step.                   │
  │ If the active phase is finished, stop and don't do anything.                │
  └─────────────────────────────────────────────────────────────────────────────┘

  (This is the Medium loop prompt from §8.2. Use Large for big phases,
   Small when a phase is nearly done. See §8.5 for the decision table.)

First phase: Phase 0 — [title from master build plan]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Wait. Do not proceed until the human approves.

---

### 3. The Phase Doc Template

Every phase in `13-master-build-plan.md` uses this exact structure. Deviation breaks the loop's ability to re-ground efficiently.

```markdown
## Phase N: <Title> [optional suffix: (§21A Checkpoint M)]

**Goal:**

One sentence stating what this phase makes true.

**Source docs:**

- `docs/build/0X-<domain-doc>.md` — what to read for context
- `docs/build/0Y-<another-domain-doc>.md` — what to read for context

(Do not list playbook paths here. Provenance markers handle that inline.)

**Steps:**

1. Step description.
   <!-- provenance: playbook §X — short reason -->
2. Step description.
   <!-- provenance: project requirements — short reason -->
3. Step description.

**Current implementation status:**

(Empty at Genesis. The loop appends one bullet per completed slice, written in the form: "<Capability> now <does specific thing> through <specific mechanism>.")

**Verification:**

- Concrete, testable assertion.
  <!-- provenance: playbook §X -->
- Another concrete, testable assertion.

**Do not:**

- Specific prohibition with provenance if relevant.
  <!-- provenance: playbook §22 anti-pattern -->
- Another specific prohibition.
```

#### Provenance markers — purpose and format

Provenance markers are HTML comments embedded next to inlined content. They serve three purposes:

1. **Loop ignores them** — HTML comments don't change the AI's instructions, so zero token cost at loop time beyond a few characters.
2. **Human audit trail** — during PR review, the human can see exactly which playbook rule drove a step.
3. **Re-baseline grep target** — if the playbook updates, grep for `§N` across `docs/build/` to find affected phases.

Format: `<!-- inlined from playbook §N — short reason -->` or `<!-- inlined from playbook §N during Phase M mid-loop consultation YYYY-MM-DD -->` for runtime additions.

#### Current implementation status — how to write entries

This field is the episodic memory of the phase. The loop appends one bullet per completed slice. Bullets must be:

- **Specific** — name the route, file, capability, or contract.
- **Mechanism-cited** — say *how* the capability works (which function, package, RPC, etc.).
- **Past-tense statement of fact** — not a TODO, not a plan.

Good example: `/admin/events/[id] now supports a dedicated human-triggered publish action for draft, review, or approved events, with public/admin route revalidation, analytics tracking, and audit logging.`

Bad example: `Implemented publish flow.`

When the phase is complete, this field becomes the historical record. The loop does not re-read entries from completed phases — once the phase is closed, its `Current implementation status` is archival, not operational.

---

### 4. The Build Loop — Invariants and Operation

When the human queues a loop prompt (§8), execute this sequence.

#### 4.1 The eight invariants

1. **Re-ground first.** Read `docs/build/13-master-build-plan.md` index + the active phase doc + relevant cursor rules. Identify what's done (status field) vs. what's next (steps not yet reflected in status).
2. **Hard-link traversal.** Every file the phase doc references must be opened with `Read` before you act on its content. Never guess.
3. **Targeted consultation only.** If the phase doc is silent on a question, open the specific playbook section that covers it. Never bulk-load.
4. **Read-don't-guess.** When citing a playbook rule, open the section in the same turn. No memorized citations.
5. **Consultation-writeback.** Resolutions from on-demand playbook opens get inlined into the phase doc with provenance, in the same commit as the code change.
6. **Park-don't-carry.** Deferred work → watchlist entry. Agent-blocked items → pre-shipping handoff. Never leave them in chat or comments.
7. **Commit cadence.** Local git commit after every meaningful completed step. Commit message format: `phase N: <task summary>`. The commit must include code + status update + any watchlist/handoff additions.
8. **Stop at phase boundary.** When the active phase's verification list passes, stop. Do not auto-advance to phase N+1. Print a phase-complete summary and wait.

#### 4.2 The per-iteration sequence

```
1. Read docs/build/13-master-build-plan.md (index portion only)
2. Read the active phase doc fully
3. Read relevant docs/build/0X-…md cited in Source docs
4. Read relevant docs/build/14-code-quality-watchlist.md entries
5. Identify next coherent task or batch (not "do everything", not "do one trivial thing")
6. For each task in the batch:
   a. Implement
   b. Run typecheck, lint, unit tests, integration tests as applicable (playbook §24 ladder)
   c. If verification fails: fix, re-run; if stuck after 3 attempts, stop and ask
   d. If a playbook consultation was needed: writeback to phase doc with provenance
   e. Update active phase doc's Current implementation status
   f. If new deferred work emerged: add watchlist entry
   g. If new agent-blocked item emerged: add pre-shipping handoff entry
   h. Local git commit
7. Self-review the batch against the rubric in §4.3
8. If phase verification list now passes: stop and print phase-complete summary
9. Otherwise: end iteration (the human queues the next one)
```

#### 4.3 The per-batch self-review rubric

Before committing a batch, walk this list:

- DRY violations introduced? (Same string in >1 file → move to canonical type/enum.)
- KISS violations? (Wrapper around one call, abstraction with one consumer, defensive memoization.)
- YAGNI violations? (Code path with no current caller, "we'll need this later" comments.)
- Separation of concerns? (UI doing data fetching, services importing routing, etc.)
- File size? (Any file you touched now > 400 lines?)
- Single source of truth? (New raw string for a route / role / event / error code?)
- Performance? (New query without `EXPLAIN ANALYZE`? `SELECT *` introduced? N+1 risk?)
- Security? (Auth check missing? Idempotency key with `Date.now()`? Inline CDN URL? Raw Server Action without safe-action wrapper? User input concatenated into LLM system prompt?)
- Race conditions? (Two writes without ordering guarantee? Optimistic update without rollback?)
- Documentation regression? (Stale comments? Outdated `Current implementation status` from a previous slice?)
- **Doc sprawl introduced?** (Any new `.md` file created this batch? If yes, it must fit the taxonomy in playbook §17 Documentation Governance. Stray `TODO.md` / `STATUS.md` / `NOTES.md` / `SCRATCHPAD.md` / investigation writeups / `.md` files inside source folders / duplicate READMEs — all forbidden. If the doc need doesn't fit the taxonomy, stop and write an ADR to propose an extension; do not invent a new doc location.)
- Tests added for high-priority AI failure modes? (Auth rejection, schema rejection, edge cases from extraction?)
- Watchlist updated for any deferred work?
- Provenance markers added for any playbook content inlined this turn?

If any item fails, fix before commit. If you can't fix (e.g. the right fix is out of scope), add a watchlist entry.

#### 4.4 The stop conditions

The loop stops — and the human is consulted — when:

1. **Phase complete.** Active phase verification list passes.
2. **Same fix tried 3 times unsuccessfully.** Architecture is wrong somewhere; reassess with human.
3. **Scope creep detected.** A task is touching files outside the phase's Source docs without obvious justification.
4. **Architectural decision required.** Phase doc + playbook both silent on a non-trivial choice.
5. **Tests blocked or environment broken.** Cannot verify; do not commit unverified work.
6. **Pre-shipping handoff item is on the critical path.** Cannot proceed without human input (e.g. Apple Team ID needed and the next task is publishing AASA).

In all cases, write a clear stop summary stating which condition triggered and what input is needed.

---

### 5. The Watchlist Entry Template

`docs/build/14-code-quality-watchlist.md` is the parking lot. Every entry follows this template:

```markdown
### <Item Title — concise, search-friendly>

Current state:

- One or two bullets describing what is currently in place.
- Include file paths or specific routes where relevant.

When to clean up:

- A concrete trigger that determines whether the cleanup is needed yet.
- "Two or more screens share the same final layout"
- "Final Supabase Storage domains are stable"
- NOT "Eventually." NOT "When we have time."

Likely future task:

- A short statement of the eventual fix.
- May include file paths or refactor direction.
```

Rules for the watchlist:

- **Add an entry whenever you intentionally defer cleanup.** If a refactor would help but the trigger isn't met, that's a watchlist entry, not a TODO comment in code.
- **Update entries when state changes.** If the trigger gets closer or the item becomes more or less risky, update Current state.
- **Remove entries only when the underlying tradeoff is resolved.** Do not remove because "we decided not to."
- **The watchlist is re-read at phase boundaries.** At phase end, walk the watchlist and resolve any items whose triggers fired during the phase.
- **There is a dedicated Watchlist Cleanup phase late in the master plan** (modeled on Ibiza FOMO Phase 17.5). At that phase, walk the entire watchlist top-to-bottom and resolve every triggered item.

---

### 6. The Pre-Shipping Handoff Template

`docs/build/15-pre-shipping-handoff.md` collects items only a human can resolve. Categories (omit any that don't apply):

```markdown
## Production Domain and Public URLs

| Item | Required value | Current draft | Notes |
|---|---|---|---|
| Production web origin | `https://…` | … | Confirm before public submission. |
| Privacy URL | Full production URL | `/privacy` | Must be public, no auth, final reviewed copy. |
| …

## Apple Developer Values

| Item | Value to gather | Where it will be used |
|---|---|---|
| Apple Team ID | `TEAMID` | iOS AASA appID. |
| iOS bundle identifier | `com.example.app` | Must match `apps/mobile/app.json`. |
| …

## Android Developer Values

| Item | Value to gather | Where it will be used |
|---|---|---|
| Android package name | `com.example.app` | Must match Play Console. |
| Release signing SHA-256 | `AA:BB:…` | Android `assetlinks.json`. |
| …

## Store Listing Decisions

- App name and display name
- Short description
- Full description
- Keywords
- Categories
- Age/content rating
- Screenshots
- Review notes

## Real-Device Verification

- iOS deep-link verification
- Android deep-link verification
- iOS payment flow on real device (if applicable)
- Android payment flow on real device (if applicable)

## Legal and Support Approval

- Privacy policy final reviewed copy
- Terms of service final reviewed copy
- Support channel decision (email / form / Discord / etc.)

## Production Credentials

- Sentry DSN
- PostHog project key
- Production database connection string
- Payment gateway keys
- (Treat as configuration the human inserts via env vars; do not commit.)
```

Rules:

- **Log items the moment they become blocking.** Don't wait until pre-launch.
- **Include the route, file, or capability that will consume the value.** So when the human provides it, the loop knows what to wire up.
- **Mark items resolved when the human supplies them.** Move resolved items to a `## Resolved` section at the bottom for audit history.

---

### 7. The Domain Doc Template (`02-…` through `12-…`)

Each domain doc follows this structure. Length varies — keep it as long as needed, no longer.

```markdown
# <Domain Title>

## Purpose

One paragraph: what this doc covers, when to consult it.

## Rules (project-specific)

Numbered list of rules that apply to this project for this domain. Extracted
from the playbook + adapted to project context.

1. Rule statement.
   <!-- provenance: playbook §X -->
2. Rule statement.
   <!-- provenance: project requirement -->

## Anti-patterns (project-specific)

Bulleted list. Every entry is a specific thing not to do.

- Anti-pattern.
  <!-- provenance: playbook §22 -->

## Templates / Patterns

Code or structural templates for common operations in this domain.

## When to re-read this doc

- Trigger: when starting a phase that cites this doc in Source docs.
- Trigger: when a watchlist entry references this domain.
- Trigger: when a cursor rule for this domain is being violated and you need the full context.
```

---

### 8. Loop Prompt Variants

These are the prompts the human queues in Cursor (or any agent runner) to drive the loop. Each variant suits a different phase size. Pick based on scope; do not mix.

#### 8.1 Large — full active phase, multi-task progress

Use when a phase is sizable and you want substantial batch progress.

```
First, check the current repo state against docs/build/13-master-build-plan.md — this is the primary controller. Align with all project cursor rules and all docs in docs/build/. Study the active phase doc carefully: what has already been done, what is next — make sure we are aligned and not diverging in any way. Check docs/build/14-code-quality-watchlist.md. Quickly scan docs/build/15-pre-shipping-handoff.md to see if any previously-blocked items can now be resolved.

Then continue building it out in full detail, adhering to all coding practices, following existing patterns. After each completed step, double-check for any code smells, documentation regression, stale comments, or any other errors you can spot. Check all logic flows and user flows make sense. Check for performance, security, code sprawl, DRY, KISS, YAGNI violations, separation of concerns, consistency in code patterns, parallel code, maintainability, race conditions, divergence — per docs.

Make sure all relevant new tests pass. Do not stop for missing API keys; implement env-based wiring and no-op / dev-safe fallbacks where appropriate.

Any left-over optimizations for later should be noted in docs/build/14-code-quality-watchlist.md using the Current state / When to clean up / Likely future task template. Any agent-blocked items should be logged in docs/build/15-pre-shipping-handoff.md.

If you need to consult the playbook for a rule the phase doc didn't cover, open only the specific section. Write the resolution back into the active phase doc with a provenance marker. Never bulk-load the playbook.

Create a local git commit after every big completed step. If the active phase's verification list passes, stop and print a phase-complete summary. Do not auto-advance to the next phase.
```

#### 8.2 Medium — single phase, coherent batch

Use when the active phase is mid-progress and you want one or two coherent slices per run.

```
Check the current repo state against docs/build/13-master-build-plan.md — this is the primary controller. Study the active phase doc carefully: what has already been done, what is next — make sure we are aligned and not diverging in any way. Verify alignment with cursor rules. Check docs/build/14-code-quality-watchlist.md. Scan docs/build/15-pre-shipping-handoff.md to see if any previously-blocked items can now be resolved.

Continue the next coherent phase-aligned task or batch of tasks. We want decent progress but not multiple unrelated things at the same time.

Adhere to all coding practices and existing patterns. After each task, double-check for code smells, documentation regression, stale comments, or other errors. Check relevant logic and user flows. Check performance, security, DRY, KISS, YAGNI, separation of concerns, consistency in code patterns, parallel code, maintainability, race conditions, divergence — per docs.

Make sure all relevant new tests pass. Do not stop for missing API keys; implement env-based wiring and no-op / dev-safe fallbacks.

Any left-over optimizations should be added or updated in docs/build/14-code-quality-watchlist.md. Any agent-blocked items go into docs/build/15-pre-shipping-handoff.md.

If you need to consult the playbook for something the phase doc didn't cover, open only the specific section and write the resolution back with a provenance marker. Never bulk-load the playbook.

Create a local git commit after every big completed step. If the active phase is finished, stop — do not auto-advance.
```

#### 8.3 Small — single phase, single coherent task

Use when the phase is nearly done and you want one careful, conservative pass.

```
Check the current repo state against docs/build/13-master-build-plan.md. Study the active phase doc: what has already been done, what is the single next task — make sure we are aligned and not diverging in any way. Check docs/build/14-code-quality-watchlist.md.

Continue the next coherent phase-aligned task. One task only — not a batch.

Adhere to all coding practices and follow existing patterns. After the task, double-check for code smells, documentation regression, stale comments, or other errors. Check relevant logic and user flows. Check performance, security, DRY, KISS, YAGNI, separation of concerns, consistency in code patterns, parallel code, maintainability, race conditions, divergence — per docs.

Make sure all relevant new tests pass. Do not stop for missing API keys; implement env-based wiring and no-op / dev-safe fallbacks.

Any deferred work goes into docs/build/14-code-quality-watchlist.md. Any agent-blocked items go into docs/build/15-pre-shipping-handoff.md.

Create a local git commit when the task is complete. If the phase is finished, stop — do not auto-advance.
```

#### 8.4 Smallest — doc, rules, and alignment sweep

Use at a phase boundary when code is stable and you want to confirm everything is consistent before moving on.

```
Check all recent changes against all docs in docs/build/ — make sure every doc reflects the current state of the code accurately. Check the active phase doc's Current implementation status: is it complete and accurate for everything done this phase?

Then check the cursor rules: do any need updating to prevent patterns of drift you observed during this phase? Are there recurring issues that should become permanent rules? Check all rules against all domain docs to make sure they are mutually consistent.

No code changes unless they are required to make the docs accurate. Make a local git commit if anything was updated. Stop when done.
```

#### 8.5 Choosing the right variant

| Phase state | Variant |
|---|---|
| Phase just started, many tasks ahead | Large |
| Phase mid-progress, multiple coherent slices remaining | Medium |
| Phase near completion, one or two careful tasks left | Small |
| Phase boundary, code stable, want to sweep docs/rules | Smallest |

Do not run Large on a phase that's nearly complete — it can cause overshoot past the stop gate. Do not run Smallest mid-phase — it skips actual work.

#### 8.6 Human setup window — before the first loop run

The autonomous loop can wire services (write the call, the env key name, the no-op fallback) but it cannot create external accounts, generate API keys, or provision infrastructure. If you let these accumulate and try to resolve them mid-loop, you interrupt the build's momentum at unpredictable points. Instead, front-load all external wiring into two fixed windows:

**Window 1 — immediately after Genesis approval, before Phase 0**

Work through this checklist once. It takes 30–60 minutes but unlocks the entire loop to run without human interruption afterwards:

- [ ] Create all external service accounts the project needs (Supabase, Stripe, Cloudinary, Resend, Upstash, etc.) — Genesis Step 2's ADR lists them.
- [ ] Generate all API keys / secrets and add them to your `.env.local` (or equivalent secrets manager). The loop will have already written the key names as `process.env.SUPABASE_URL`-style references into the code — you're just providing the values.
- [ ] Run `supabase init` / `supabase link` (or equivalent BaaS CLI) so the loop can run migrations locally.
- [ ] Configure any MCP servers the loop will need (e.g. `supabase`, `stripe`, `github`). Add them to `.cursor/mcp.json`. The loop will use these to call service APIs autonomously — without this, it falls back to no-ops.
- [ ] Push the initial scaffold commit so CI can run from Phase 0 onwards.
- [ ] Confirm the testing ladder passes on the clean scaffold: `pnpm typecheck && pnpm lint && pnpm test`.

**Window 2 — after Phase 0 (repo scaffold) completes, before Phase 1**

Phase 0 is purely local — no external services needed. After Phase 0's verification passes:

- [ ] Apply the initial DB migration: `supabase db push` (or equivalent). This wires the schema the loop will build against from Phase 1 onwards.
- [ ] Set production/staging environment variables in your deploy platform (Vercel, Fly.io, etc.) so CI deployments don't fail on missing env.
- [ ] Verify the deploy pipeline runs: push a dummy commit, confirm CI passes end-to-end.

**After Window 2: the loop is fully autonomous.**

From Phase 1 onwards, the loop can call external services via MCP, run migrations, deploy previews, and verify against real data. The pre-shipping handoff register (`15-…md`) will accumulate items the loop structurally cannot do (App Store submission, real-device tests, production DNS cutover) — but those are launch-gate items, not build-blockers. The loop logs them and moves on.

**What to do if a new external service appears mid-build:**

If the loop hits a service it can't reach (MCP not configured, key missing), it should:
1. Log the item in `15-pre-shipping-handoff.md` with the key name(s) needed.
2. Implement the no-op / dev-safe fallback so the loop can continue.
3. Not stop or ask — the human will see the handoff entry at the next phase-boundary review.

When the human sees the entry, they add the service to MCP / `.env`, resolve the handoff item, and the next loop run picks up the real implementation path. One interrupt, one resolution, no mid-loop blocking.

---

### 9. Drift and Re-Baseline Protocols

#### 9.1 Drift — phase doc was under-specified

When mid-loop you find the phase doc is silent on something non-trivial:

1. **Stop the current task.** Do not improvise architecture.
2. **Open the relevant playbook section** with `Read`.
3. **If the playbook covers it:** apply the rule, write the resolution back into the phase doc with provenance, continue.
4. **If the playbook doesn't cover it:** stop and ask the human. Do not invent.
5. **Update the phase doc** so the same gap doesn't trigger a stop next iteration.

#### 9.2 Re-baseline — playbook updated post-Genesis

Since the playbook is copied as a snapshot at Genesis (not git-submoduled), it doesn't auto-update. To adopt newer playbook content into an existing project:

1. **Replace `docs/playbook/`** with the new playbook snapshot.
2. **Diff** the new playbook against the old one to identify changed sections.
3. **For each changed section §N:** grep `docs/build/` for `§N` provenance markers. Each marker is a phase doc location that inlined content from that section.
4. **For each affected phase doc:** open the new playbook §N, compare with the inlined content, update the phase doc if the change is material. Update the provenance marker date.
5. **For affected cursor rules:** review and update if the rule body needs new content.
6. **Commit the re-baseline** as a single logical change with a clear message: `re-baseline playbook §N — <reason>`.

Re-baseline is human-initiated. Do not run a re-baseline as part of the autonomous loop.

#### 9.3 Repeated drift — the meta-signal

If the same kind of drift recurs in multiple phases (e.g. the loop keeps consulting the playbook for §13 observability rules), it means Genesis under-extracted §13. The fix is a focused re-baseline of `11-observability-system.md` plus an updated cursor rule.

---

### 10. Quick Reference

#### Genesis kickoff prompts

**If playbook is already in the repo** (`docs/playbook/` exists — the cursor rule handles the rest):

```
Start Genesis.
```

**If starting from scratch** (no repo yet, or first-time setup):

```
Read https://raw.githubusercontent.com/[your-org]/engineering-playbook/main/00-genesis-and-build-cadence.md and start Genesis.
```

Both trigger the same protocol. The AI opens this file, begins the intake questions, produces all artifacts, waits for approval. It will not implement anything until the human approves.

#### Files the loop must always read at iteration start

1. `docs/build/13-master-build-plan.md` (index portion)
2. Active phase doc within `13-master-build-plan.md` (or its referenced file if phases are split)
3. Cursor rules auto-attached (no manual read)
4. `docs/build/0X-…md` files cited in the active phase's `Source docs`
5. `docs/build/14-code-quality-watchlist.md` (relevant entries)

#### Files the loop must never read at iteration start

1. `docs/playbook/*` (any file) — read on demand only
2. Completed phase docs / archived `Current implementation status` from previous phases
3. ADRs (read only when an architectural decision is being made)

#### Provenance marker forms

- `<!-- inlined from playbook §X -->` — Genesis-time inline.
- `<!-- inlined from playbook §X during Phase N mid-loop consultation YYYY-MM-DD -->` — runtime inline.
- `<!-- inlined from project requirements -->` — non-playbook inline.

#### Watchlist entry template

```markdown
### <Item Title>

Current state:
- …

When to clean up:
- …

Likely future task:
- …
```

#### Phase-complete summary template (loop prints this at stop)

```
Phase N — <Title> — complete.

Tasks completed this run:
  - <task 1>
  - <task 2>

Verification:
  - [pass] <criterion 1>
  - [pass] <criterion 2>

Watchlist additions:
  - <entry title>

Pre-shipping handoff additions:
  - <item>

Next phase: Phase N+1 — <Title>.
Awaiting human queue.
```

---

## Closing Principle

This system is one principle applied recursively: **externalize state to interpretable markdown files so the AI can re-ground itself without retaining context, and verify everything through layered tests so the externalized state is corroborated rather than trusted.**

Genesis converts the playbook into project-specific phase docs that fully describe what to build. The loop reads phase docs, not the playbook, and verifies its own claims through typecheck, lint, unit, integration, and E2E gates before committing. The watchlist captures intentional deferrals. The pre-shipping handoff captures agent-blocked items. Provenance markers preserve the audit trail back to the playbook for re-baseline.

When in doubt: re-read the active phase doc. If the phase doc is silent, open the specific playbook section. If the playbook is silent, ask the human. Never invent architecture.
