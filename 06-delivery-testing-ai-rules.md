<!-- Split from general-engineering-playbook.md — content verbatim. See README.md for reading order. -->

## 14. Testing Pyramid

> **Why now:** Vitest (2022+) replaced Jest as the standard for TypeScript projects — it shares Vite's config, is 2-5× faster on cold starts, and has native ESM support. Playwright (2020, stable 2021) replaced Cypress as the E2E standard — it's multi-browser, has better auto-waiting, and the Microsoft team actively maintains it. Testcontainers for Node.js (stable 2023) enables integration tests against a real PostgreSQL instance in Docker without a shared dev database — tests are isolated, reproducible, and run in CI without infrastructure setup. The testing pyramid ratio (many unit, moderate integration, few E2E) is now better understood after years of teams inverting it and ending up with slow, flaky CI pipelines.

> **AI impact — this is the most important section for AI development.** Tests are the primary way the AI verifies its own output. The specific order matters:
> 1. **Unit tests on pure functions** — the AI writes a function and its test in the same pass. If the test fails, the AI corrects the function. This loop is very fast (< 1s per test run).
> 2. **Integration tests with auth rejection** — prevents the AI from accidentally shipping an endpoint that works but is publicly accessible. The AI can't "forget" auth if the test enforces it.
> 3. **TypeScript typecheck** — the fastest feedback loop (2s). The AI runs this mentally while writing code because TypeScript errors appear inline in Cursor.
>
> A well-tested codebase means the AI can make changes confidently: if it breaks something, a test fails immediately. Without tests, the AI is writing blind and so are you.

Enforce the ratio with CI gates. The 2026 consensus: **70% unit / 20% integration / 10% E2E**.

| Tier | Ratio | What | Tool |
|---|---|---|---|
| **Unit** | ~70% | Pure functions, state machines, validation. No DB / HTTP / FS. | Vitest |
| **Integration** | ~20% | API route handlers + real test DB. Auth flows. Repository layer against a real schema. | Vitest + Testcontainers |
| **E2E** | ~10% | Critical user journeys only (signup, first purchase, core CRUD). | Playwright |

**Mock at the boundary, not inside the application:**
- **Database**: Testcontainers (real Postgres in Docker) — never mock the ORM or DB client.
- **External HTTP APIs** (third-party services, payment APIs, email): **MSW (Mock Service Worker)** — intercepts at the network level, not at the import level. This means your actual fetch/axios/ky code runs; only the server response is mocked.
- **Never mock internal application modules** (`vi.mock('./my-service')`) — if you're mocking your own code, you're testing the mock.

**Vitest Browser Mode for component tests:**
Vitest can run component tests in a real Playwright browser — not jsdom. Real CSS rendering, real browser APIs, real event handling. Use `@vitest/browser` for components that rely on DOM layout, CSS classes, or browser-specific behaviour. Use standard jsdom mode for pure logic tests (faster).

**E2E tests must run against a production build:**
```bash
# Correct — tests what you actually ship
npm run build && npm start
# Wrong — dev server has different code paths, HMR overhead, source maps
npm run dev
```
Configure Playwright's `webServer` to build and start before the test run. A test that passes in dev but fails in production is worse than no test.

**When adding a feature, write in this order:**
1. Service-layer business logic (unit)
2. API route — happy path + auth rejection + validation rejection (integration)
3. Auth guard behaviour (integration)

**Never skip the auth rejection test.**

---

## 14A. LLM Evaluations (the testing pyramid for non-deterministic features)

> **Why now:** The testing pyramid in §14 assumes deterministic code — same input, same output. LLM-powered features are non-deterministic by construction: a prompt change, a model version bump, or a temperature tweak silently degrades quality with no failing test. Without evals, AI features ship as black boxes that drift toward worse over time. This is now the single most common cause of "the chat used to be smart, when did it get dumb?" in production.

> **AI impact:** Evals are to LLM features what `tsc` is to TypeScript code — a fast feedback loop the AI uses to know if a prompt or pipeline change made things better or worse. Without evals, the AI is changing prompts blind and so are you. With evals, prompt changes become regression-tested like any other code change.

**The four-layer eval stack** (mirror of §14 testing pyramid):

| Layer | Ratio | What | Tool |
|---|---|---|---|
| **Unit-style evals** | ~60% | Deterministic input → deterministic output checks (structured outputs, JSON shapes, refusal triggers). Run on every PR. | Vitest assertions on `aiService.generate()` |
| **Reference evals** | ~25% | Pinned input → expected output examples. Pass/fail via exact match, regex, or semantic similarity. | [Promptfoo](https://promptfoo.dev/) or [OpenAI Evals](https://github.com/openai/evals) |
| **LLM-as-judge evals** | ~10% | Open-ended outputs scored by a second LLM on rubric criteria (tone, accuracy, refusal correctness). | Promptfoo's `llm-rubric` assertion, [OpenAI evals graders](https://github.com/openai/evals) |
| **Production sampling** | ~5% | Sample real production traffic, judge offline, surface regressions. | PostHog LLM analytics or DIY |

**Rules:**
- **Prompts are versioned artifacts.** System prompts live in `packages/ai/prompts/` as `.md` or `.ts` files — never inline string literals in service code. Each prompt has a version constant; prompt changes are commits.
- **Eval suite runs in CI** on any PR touching `packages/ai/**`. Below the regression threshold (e.g. pass rate drops > 5 points), block merge.
- **Cost and latency are first-class metrics in evals**, not just correctness. A prompt that's 10% more accurate but 4× more expensive is usually wrong.
- **Refusal cases are tested explicitly** — known prompt-injection payloads, jailbreak attempts, abuse prompts. Assert refusal, not just "no error" (see §12 LLM Product Security).
- **Eval datasets live in the repo** (`packages/ai/evals/`) under git. Pull-request review covers test changes the same as code changes.

**The eval workflow when changing a prompt:**
1. Capture current behaviour: run evals on `main` and save the report.
2. Change the prompt.
3. Re-run evals. Diff the reports.
4. If pass rate or cost regressed beyond threshold, the prompt change is rejected.
5. If the change is intentional ("we accept lower clarity for higher refusal correctness"), document the trade in the PR description and update the baseline.

This is the same discipline as performance budgets in §8 — you measure, you set a target, you regress against the target, not against intuition.

---

## 15. CI/CD & Deployment

> **Why now:** GitHub Actions became the de-facto standard CI by 2022, replacing Jenkins and CircleCI for new projects. Vercel (web) and Fly.io (custom servers) are the 2026 deployment defaults for JS/TS apps — zero-config deploys, preview environments per PR, and automatic rollbacks. The "DB migrations first, then app deploy" order is a hard-won lesson: deploying the app before the migration runs means the new code hits the old schema, which causes runtime errors. Automating this sequence is now table stakes. The manual approval gate before production prevents a bad PR merge from reaching users without a human decision.

> **AI impact:** The CI pipeline is the AI's automated reviewer. If the AI writes code that fails typecheck, lint, or tests, the PR is blocked before any human reviews it. This creates a tight feedback loop: AI writes → CI runs → feedback → AI corrects. Teams that skip CI on AI-generated PRs accumulate technical debt very fast because the AI optimises for "code that compiles" not "code that is correct."

### Pipeline (GitHub Actions)

| Trigger | Steps |
|---|---|
| Every PR | typecheck, `eslint` or `biome lint` (incl. `import-x/no-cycle`), unit tests, build, integration tests |
| Merge to `main` | deploy to staging, integration tests against staging, **manual approval** gate |
| Production deploy | DB migrations first → application deploy second → health check → automatic rollback if health fails within 60s |

### Environments
- `development → staging → production`. Staging is a **complete mirror** (same CDN rule structure, same env-var structure, separate DB). One exception: staging must have an explicit position-1 cache bypass rule that production does not — see §8A "Staging Must Structurally Bypass Cache". "Same structure" means the same types of rules configured, not identical settings.

### DNS & Domain Strategy
Get this right on day 1; retrofits break deep links, CSP, and CORS.

- **Decide apex vs www early.** Pick one as canonical, 301-redirect the other. Both work; mixing creates duplicate-content SEO issues and cookie scope confusion.
- **Subdomain split for clarity:**
  - `yourdomain.com` — main app (or `app.` if marketing lives on apex)
  - `media.yourdomain.com` — R2/CDN (subject of §8A)
  - `api.yourdomain.com` — if you expose a public API
  - `staging.yourdomain.com` — staging app; `media-staging.yourdomain.com` — staging media
- **TLS auto-renewal.** Vercel/Cloudflare/Fly handle this automatically. If you manage certs by hand (legacy), set an expiry alarm 30 days out; expired certs are the most preventable outage.
- **CAA record** restricting which CAs can issue certificates for your domain — prevents BGP-hijack TLS attacks.
- **DNSSEC enabled** at the registrar — protects against DNS spoofing. Most registrars expose a one-click toggle.
- **Email DNS** (SPF / DKIM / DMARC) — see §19 Email Deliverability.

### Browser Support Matrix
Declare it explicitly in `package.json` `browserslist` field. The default is too permissive and ships polyfills for IE11.

```json
"browserslist": [
  ">0.5%",
  "last 2 versions",
  "Firefox ESR",
  "not dead",
  "not IE 11"
]
```
Browserslist drives Babel/SWC transforms, Autoprefixer, and Next.js's modern-vs-legacy bundle decisions. A tighter list = smaller bundles. Revisit annually.

### Application deploy vs media deploy
- **Application pipeline** (§15 table above): ships code — typecheck, test, deploy app, migrations.
- **Media pipeline** (§8A Maturity A): ships bytes — `upload:media --env staging` after git asset changes; `promote:media` staging → prod R2 after sign-off. Optional later: CI runs upload when `public/assets/**` changes on merge to staging branch.
- These are independent. Merging code without uploading media leaves staging/prod serving old R2 objects while local git looks correct.

### Linting setup (2026)

**Important Next.js 16 change:** The `next lint` command was removed in v16. Run `eslint` (or `biome lint`) directly from your CI scripts and `package.json`. `next build` no longer runs linting.

**Option A — ESLint (most common, most hirable):**
ESLint 9+ uses a single `eslint.config.mjs` file (flat config) — `.eslintrc.*` files are deprecated and removed in ESLint 10. Required packages:
- `typescript-eslint` v8 (consolidated — replaces the old `@typescript-eslint/eslint-plugin` + `@typescript-eslint/parser`)
- `eslint-plugin-import-x` (flat-config compatible fork of `eslint-plugin-import`)
- `eslint-config-prettier`, `@eslint/js`

**Option B — Biome (newer, faster, simpler):**
[Biome](https://biomejs.dev/) is a Rust-based all-in-one formatter + linter. 10-30× faster than ESLint + Prettier. Zero plugins, one config file (`biome.json`), growing ecosystem (10K+ GitHub stars in 2024-2025). Good choice for new projects where you want to eliminate toolchain complexity. Tradeoff: no `import-x/no-cycle` equivalent yet (use `madge` for cycle detection in CI if needed). Most commonly adopted for formatting; linting adoption is growing.

For most teams hiring in 2026: **ESLint** is still the safe default (every React developer knows it). **Biome** is the forward-looking bet for new projects prioritising speed.

### Process (self-hosted servers only)
> This section applies to long-running Node.js processes deployed on Fly.io, Render, or Kubernetes. **Skip for Vercel / other serverless platforms** — the platform handles health checks and rollbacks automatically, and there is no persistent process to shut down.
- App runs as a non-root user.
- **Graceful shutdown** on SIGTERM: close new connections → drain in-flight (30s timeout) → close DB connections → exit.
- **`/health` returns 200** only if DB is reachable. Otherwise 503. The orchestrator's liveness probe owns rollout decisions.

### Horizontal scaling prerequisites
Must all be true *before* adding replicas:
- Sessions in Redis (not MemoryStore)
- Rate limits in Redis (not in-memory)
- No singleton state in the app process
- All file storage external (S3/R2, never local disk)
- WebSocket sticky sessions at the LB (WS only)

### Feature Flags & Progressive Delivery

Feature flags are not just for A/B tests — they are the primary mechanism for safe deploys at any scale. Code that is in production but behind a flag is deployed (not a rollback risk), tested against real infrastructure, and can be disabled in seconds with a toggle instead of a redeploy.

**Four uses, in order of adoption:**
1. **Kill switch** — instantly disable a broken feature without touching code or triggering a deploy.
2. **Staged rollout** — 1% of users → 10% → 50% → 100%. Surface regressions before full blast.
3. **Beta / early access** — enable a feature for specific user IDs or segments before general release.
4. **A/B experiment** — compare two variants and measure conversion against your analytics tool.

**Tooling:**
| Tool | Best for | Notes |
|---|---|---|
| **PostHog Feature Flags** | Teams already on PostHog analytics | Zero extra infra; flags and analytics in one tool |
| **LaunchDarkly** | Enterprise; complex targeting rules | The gold standard; pricing scales with usage |
| **OpenFeature** SDK | Vendor-agnostic abstraction | 2024 CNCF standard — abstract behind OpenFeature and swap providers without code changes |
| **DB-backed flag table** | Earliest stage; handful of flags | A `features` table with a `boolean` column. Migrate to a real tool before you have > 10 flags. |

**Rules:**
- Every flag has an **owner** and a **removal date**. Flags are temporary by design — a flag with no end date is a permanent conditional that no one remembers.
- Evaluate flags **server-side** for anything security- or billing-related. Client-side evaluation is for UX optimisation only.
- A flag does not mean the code is incomplete. Code behind a flag should be fully tested; the flag controls *when* users see it, not *whether it works*.
- Treat flag cleanup as part of the feature work — remove the flag and the dead branch as soon as the rollout completes.

---

## 16. Cursor Rules & AI-First Development

> **Why now:** Cursor's `.cursor/rules/` system (`.mdc` files with YAML frontmatter, introduced in 2024) is the current standard for project-scoped AI instruction. Before this, teams relied on comments, README sections, or system prompts that the AI frequently ignored. Rules in `.mdc` files are automatically attached to AI context based on file globs, so the AI always has the relevant rules in context when editing a specific file type — it doesn't need to "remember" to check a README. This is the 2026 equivalent of a linter: automatic, context-aware, and enforced.

> **AI impact — second-most important section.** Cursor rules are the mechanism by which your architecture governs the AI. Without rules, each AI session starts fresh with no knowledge of your conventions. With rules:
> - The AI knows which imports are banned (`no-restricted-imports`).
> - The AI knows the file-size limits and will split rather than grow.
> - The AI knows which dependencies are forbidden ("do not add Redis unless explicitly approved").
> - The AI knows to write tests in the specific order your team uses.
> - The AI knows to record ADRs for architectural decisions.
>
> Rules are the highest-leverage investment you can make in AI-assisted development. Spend 30 minutes writing good rules at project start and you save hours of correction per week.

This is the single biggest force-multiplier for a small team. The codebase teaches the AI agent how to be a good contributor.

### Folder
```
.cursor/rules/
  001-core-project.mdc          ← always-applied: project scope, do-not-touch list
  002-typescript-standards.mdc  ← TS strict, Zod, file size, imports
  003-frontend.mdc              ← framework conventions, route ownership
  004-mobile.mdc                ← (if applicable)
  005-database.mdc              ← RLS, migrations, RPC discipline
  006-backend.mdc               ← layering, env validation
  007-testing-quality.mdc       ← test pyramid, what to write first
  008-styling-design.mdc        ← canonical UI stack (Tailwind v4, shadcn/ui, Motion, Lucide, next/font), tokens, three-layer styling model
  009-observability-caching.mdc ← log levels, cache layers
  010-assets-cdn.mdc            ← asset taxonomy (three-bucket model), URL construction, CDN rules
```

### Rule file `008-styling-design.mdc` — what it must contain

Styling violations happen everywhere — components, layouts, feature routes, shared packages — which is why this rule uses `alwaysApply: true`. The canonical UI stack is non-negotiable: any AI that reaches for a different library, a hardcoded hex value, or a Google Fonts `<link>` tag must be stopped before the commit.

```yaml
---
description: Canonical UI stack, design tokens, and three-layer styling model (see Appendix and 07-design-system.md)
globs: ""
alwaysApply: true
---
```

Bullet content for the rule body (copy verbatim into the `.mdc` file, then add project-specific token names and `packages/ui` paths):

```markdown
## Canonical UI Stack

- CSS framework: **Tailwind CSS v4** only. `@import "tailwindcss"` in the root CSS file. No `tailwind.config.js` or `tailwind.config.ts` (v4 is config-free by default). Never use CSS Modules, styled-components, Emotion, or inline `style={}` for anything Tailwind can express.

- Component primitives: **shadcn/ui** (Radix UI). Components live in `packages/ui/` as owned source code — they are NOT an npm dependency. Import from `packages/ui/` only. Never install `@shadcn/ui` as a package. Never write raw HTML + Tailwind utility strings directly in feature code (`apps/**`) — always go through `packages/ui/` primitives.

- Animation: **Motion (Framer Motion v12)** via `import { motion } from "motion/react"`. Never `import { motion } from "framer-motion"` (old import path). Never use plain CSS `@keyframes` for anything more complex than a simple fade or skeleton shimmer.

- Icons: **Lucide React** only (`import { IconName } from "lucide-react"`). Never mix icon libraries. Never inline SVGs directly in feature components — add them to `packages/ui/icons/` if Lucide doesn't have them.

- Fonts: **`next/font`** only (Google fonts via `next/font/google`, local fonts via `next/font/local`). Never add a `<link rel="stylesheet" href="https://fonts.googleapis.com/...">` tag anywhere. Never use CSS `@import url("https://fonts.googleapis.com/...")`. Font classes are applied at the root layout — not per-component.

## Design Tokens

- NEVER hardcode a color (`#3B82F6`, `rgb(...)`, `hsl(...)`), spacing value (`16px`, `1.5rem`), or border-radius in a component. Always use the CSS custom property from `packages/design-tokens/`.
- NEVER use Tailwind's arbitrary value syntax (`text-[#3B82F6]`, `mt-[17px]`) for design-system values. Arbitrary values are for genuinely one-off layout math only.
- Design tokens are the single source for any visual value. A theme change must be a one-file change in `packages/design-tokens/`.

## Three-Layer Styling Model

- **Phase 1 (design system foundation):** tokens, shadcn/ui setup, base layout, fonts — done before any feature work.
- **Feature phases:** build with real `packages/ui/` primitives using token values. Never leave placeholder/unstyled markup with a comment "style this later."
- **Visual polish phase (pre-launch):** pixel-perfect pass — micro-interactions, hover/focus/active states, motion, responsive edge cases, dark mode, WCAG contrast. Only this phase does polish work; do not gold-plate features during earlier phases.

## Prohibited Substitutions

- NEVER install MUI, Mantine, Ant Design, Chakra UI, or any other component library alongside shadcn/ui. If a constraint requires a substitute, that is an ADR decision, not a unilateral install.
- NEVER use `styled-components`, `@emotion/react`, or CSS Modules as the primary styling layer.
- NEVER import from `framer-motion` (old package) — always `motion/react`.
```

### Rule file `010-assets-cdn.mdc` — what it must contain

Asset taxonomy violations happen everywhere — components, API routes, config files, utility modules — not just in dedicated asset files. For that reason this rule uses `alwaysApply: true`. This is one of the rare cases that justifies it: the rule body is short, entirely declarative, and the class of mistake it prevents is universal. Do not be tempted to scope it to a narrow glob; the whole point is that an AI writing a new component should never reach for an inline CDN URL.

```yaml
---
description: Asset taxonomy, CDN URL construction, and cache contract rules (see §8A of general-engineering-playbook.md)
globs: ""
alwaysApply: true
---
```

Bullet content for the rule body (copy verbatim into the `.mdc` file):

```markdown
## Asset Taxonomy — Three-Bucket Model (§8A)

- NEVER place product images, character art, personality cards, user media, or any asset that can change without a code deploy inside the framework `public/` directory with a stable filename.
  - `public/` files are copied verbatim by the bundler — they are not hashed. Pairing `Cache-Control: immutable` with a stable filename is a broken promise. Use object storage (Bucket 3) instead.
  - The two acceptable uses of `public/`: files that MUST retain their exact filename (`robots.txt`, `favicon.ico`, `apple-touch-icon.png`) and assets that are truly never changed.

- Every asset class belongs to exactly one bucket. If you are adding a new asset class, declare its bucket before writing any code:
  - **Bucket 1 (Bundler decides URL):** JS, CSS, fonts, icons imported in source code — bundler hashes filename automatically. Use `import` statements; never reference from `public/`.
  - **Bucket 2 (Deploy decides URL):** HTML entry points, config JSON, build manifests — always fresh, never CDN-cached.
  - **Bucket 3 (Content decides URL):** Everything that can change without a code deploy — object storage + `MEDIA_BASE`. Maturity A: `getAssetUrl(path)` with stable keys mirrored from git. Maturity B: DB stores `publicUrl` (often UUID keys). Never inline CDN URLs in components.

- NEVER construct a CDN or R2 asset URL inline in a component, page, or service. All asset URL construction must go through the designated helper in `packages/assets/urls.ts` (or the project equivalent). One construction site per asset class, everywhere else calls the helper.

- NEVER use `?v=N` as the primary cache-busting strategy for product images or character assets. `?v=N` relies on human memory. The structurally safe options are: new R2 key per change (Maturity B), or stable key + explicit `Cache-Tag` purge in the upload/promote script (Maturity A).

- NEVER apply the same CDN cache rule to assets with different change frequencies (e.g. thumbnails that update on character refresh vs. video segments that are immutable after upload). Different change frequencies require different cache rules.

- NEVER run `purge_everything` zone-wide as a deploy step. Use `purge_cache` with `tags` or `prefixes` only. Zone-wide purge hits Cache Reserve, causes a thundering herd on origin, and invalidates unrelated assets.

- Staging environments MUST have an explicit CF Cache Rule at position 1 that bypasses cache (match: `http.host contains "-staging." OR "-beta." OR ".preview."`). A staging environment that is edge-cached is not a staging environment.

- When adding image processing or thumbnail generation, check whether Cloudflare Image Resizing (`/cdn-cgi/image/width=...,height=...,fit=crop,gravity=face/...`) can replace a pre-generation pipeline before writing any build scripts. One source file + transform URL = no pipeline, no size-variant storage, automatic WebP/AVIF negotiation per browser.

- Before shipping a new asset class: (1) add a row to `docs/adr/NNN-asset-taxonomy.md`, (2) confirm it has a bucket, a key scheme, and a cache rule. Undocumented asset classes are a future incident.
```

### What each rule file contains
A `.mdc` file has YAML frontmatter and a short body. Example header:
```yaml
---
description: TypeScript, validation, file size, and import standards
globs: "**/*.{ts,tsx}"
alwaysApply: false
---
```

### Authoring rules — principles
- **Short, declarative bullets.** Each rule is a single sentence the AI can act on without inference.
- **Always include "do not" rules** — the AI is biased toward action, so banning actions is more useful than encouraging them.
- **Cite the doc the rule comes from** — e.g. `See docs/build/14-code-quality-watchlist.md`. The AI follows citations.
- **Include cleanup triggers** — "Aim < 250 lines; review > 400; split > 600." Numbers > vague guidance.
- **`alwaysApply: true` for core-project scope and universal architectural rules.** The two valid cases: (1) `001-core-project.mdc` — project-wide conventions that apply in every file; (2) short, declarative rules whose violation class is universal and not predictable from a filename alone (e.g. asset taxonomy — an inline CDN URL can appear in any component, route, or utility). Everything else attaches via `globs:` so context stays focused.
- **Update rules when conventions evolve.** Stale rules misguide both humans and AI.

### Multi-agent governance: Cursor rules + CLAUDE.md

Teams using both Cursor and Claude Code (Anthropic's CLI agent) should maintain **both** files — they serve different agents but enforce the same conventions:

| File | Used by | Format |
|---|---|---|
| `.cursor/rules/*.mdc` | Cursor IDE | YAML frontmatter + markdown, glob-scoped |
| `CLAUDE.md` | Claude Code CLI | Plain markdown, loaded at session start |

The content should be consistent: same bans, same file-size limits, same test patterns. `CLAUDE.md` is typically a condensed version of the core Cursor rules (100–200 lines) focused on code-generation patterns rather than IDE-specific features.

With Cursor 3.3+ parallel agents, each agent that spawns in a worktree loads your `.mdc` rules independently. Good rules compound — every agent generates code that fits your codebase. Poor rules multiply inconsistency.

### Companion documentation
- `docs/build/00-genesis-and-build-cadence.md` — entry-point protocol (Genesis intake + Build Loop) when an AI agent drives the build. Defines the operating system on top of this playbook.
- `docs/build/13-master-build-plan.md` — phased execution sequence produced by Genesis. The loop reads this every iteration.
- `docs/build/14-code-quality-watchlist.md` — deferral register with `Current state` / `When to clean up` / `Likely future task` entries. Prevents "why is this still like this" mysteries.
- `docs/build/15-pre-shipping-handoff.md` — items only a human can resolve (developer-account values, real-device tests, store listing, production URLs).
- `docs/build/0X-…md` — long-form domain docs (DB plan, AI governance, screen map, asset system, etc.) produced by Genesis.
- `docs/adr/NNN-*.md` — Architecture Decision Records (see §17).
- `docs/tasks/T-NNN-*.md` — off-loop task briefs (urgent fixes, spikes that don't fit the master plan). See "Phase Docs and Task Briefs" above.
- `docs/runbooks/*.md`, `docs/postmortems/YYYY-MM-DD-*.md` — operational procedures and incident records. See §17 Documentation Governance for the full taxonomy.

> The full taxonomy of allowed `.md` locations (and the banned-pattern list — `TODO.md`, `STATUS.md`, `NOTES.md`, investigation writeups, `.md` files inside source folders, etc.) lives in playbook §17 Documentation Governance. The build loop's self-review (§4.3 of `00-genesis-and-build-cadence.md`) and the `001-core-project.mdc` cursor rule both enforce it. If a new doc need doesn't fit the taxonomy, write an ADR proposing the extension — do not invent a new doc category by writing the file.

### AI-friendly conventions
- **One file per feature module.** AI agents grep for the file name from the feature description — predictable names reduce search cost.
- **Consistent naming prefixes** instead of nested subfolders: `admin-review-*.tsx`, `actions-draft.ts`, `actions-media.ts`. Flat is greppable.
- **Server actions split by domain** when they grow: `actions-draft.ts`, `actions-media.ts`, not one 1500-line file.
- **Result types are explicit unions** (`{ status: "success"; data } | { status: "error"; message }`). The AI knows exactly what to return.

### AI session hygiene (secrets, supply chain, scope)

The AI sees everything you let it see and installs everything you let it install. These four controls close the most common 2026 IDE-agent failure modes.

- **`.cursorignore` / `.aiexclude` on day 1.** Without these, the AI IDE uploads `.env*`, `secrets/`, `keys/`, large binaries, `node_modules/`, build output, and DB dumps to the vendor every prompt. This is the #1 secret-leak vector in 2026. Minimum exclusions: `.env*`, `**/*.pem`, `**/*.key`, `secrets/`, `keys/`, `**/service-account*.json`, `**/.supabase/`, `dist/`, `build/`, `*.sqlite`, `*.dump`.
- **Hallucinated-dependency guard.** AI agents invent plausible-looking package names ("slop-squat"). Attackers register the invented names with malicious code. Rule: never accept an AI's `pnpm add X` without verifying (1) the package exists on npm, (2) > 1K weekly downloads or it's on an internal allow-list, (3) maintained in the last 12 months. Add `pnpm install --frozen-lockfile` to CI so an unreviewed package can't slip in via a side commit.
- **Dependency budget — no dep > 50 KB gzipped without an ADR.** AI adds libraries casually (`moment`, full `lodash`, four date libraries). Without friction, your bundle doubles per quarter. Pair with the §8 bundle-size budget in CI.
- **Renovate or Dependabot enabled day 1.** Grouped, automated dep PRs. Without it, supply-chain CVEs sit in your repo for months while the AI confidently builds on top.

### Phase Docs and Task Briefs — the unit of AI work

Long AI sessions drift. Chat history degrades. "The AI rewrote half the codebase" failures are context-loss failures, not capability failures. Externalized, structured specs are the fix.

There are two unit-of-work artifacts depending on how you're operating:

**Phase docs (primary, for build-loop work)** — when you've adopted the Genesis & Build Cadence system from `docs/build/00-genesis-and-build-cadence.md`, the unit of AI work is a phase in `docs/build/13-master-build-plan.md`. Each phase has Goal / Source docs / Steps / Current implementation status / Verification / Do not. The loop reads the phase doc + cursor rules and executes one batch per run. See `00-…md §3` for the full phase doc template.

**Task briefs (fallback, for off-loop work)** — for ad-hoc tasks that don't fit the master build plan (urgent CVE patches, exploratory refactors, one-off cleanups, debugging investigations), use a task brief.

- **Brief file:** `docs/tasks/T-NNN-feature-slug.md`. Sections: **Goal**, **Inputs** (files/data the AI may read), **Outputs** (files the AI may create/edit, with paths), **Out of scope** (files NOT to touch), **Acceptance criteria** (concrete, testable — the eventual unit-test names). The brief is the spec; chat is the conversation.
- **When to write one:** every off-loop task that touches > 5 files, > 2 packages, or introduces a new pattern. Smaller off-loop tasks can live in chat.
- **The brief survives the session.** New session, same brief, identical scope. This is the only reliable way to resume AI work without re-explaining everything.
- **The brief becomes the PR description.** One artifact, traceable from goal → diff → tests → reviewer notes.
- **Brief → tests → AI.** Combined with the §14/§24 "write test first" rule: the brief lists test names; you stub the failing tests; the AI's job is to make them pass without touching out-of-scope files.

**Which to use, when:**

| Scenario | Use |
|---|---|
| Implementing a feature scoped in the master build plan | Phase doc (already exists) |
| Multi-iteration build progress on a known capability | Phase doc + queued loop prompt |
| Urgent bug fix outside the current phase | T-NNN task brief |
| Exploratory spike before deciding architecture | T-NNN task brief (with "exploratory" flag in Goal) |
| Cleaning up a triggered watchlist item | Either phase doc's `Current implementation status` (if part of dedicated watchlist phase) or T-NNN brief (if isolated) |
| One-line fix the human is making themselves | Neither — direct edit is fine |

---

