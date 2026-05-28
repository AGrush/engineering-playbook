<!-- Split from general-engineering-playbook.md — content verbatim. See README.md for reading order. -->

## Appendix: Recommended Default Stack

This is a starting stack that satisfies every rule above. Substitute components only with reasoned ADRs.

| Layer | Choice | Notes |
|---|---|---|
| Language | TypeScript strict | `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` |
| Repo | pnpm + Turborepo | Task graph caching, phantom dep prevention |
| Web framework | **Next.js 16** App Router | Turbopack default (2-5× faster builds), Cache Components (`"use cache"` stable), React 19.2, React Compiler (opt-in) |
| CSS | **Tailwind CSS v4** | CSS-native, zero config, `@import "tailwindcss"` — no `tailwind.config.js` needed. Utility-first eliminates specificity wars and dead CSS. Default for all new projects. |
| Component primitives | **shadcn/ui** (built on Radix UI) | Headless, accessible, composable — you own the source code (copied into `packages/ui/`, not an npm dependency). AI-friendly: consistent file structure, predictable patterns. Never write raw HTML + Tailwind inline in feature code — always go through `packages/ui/`. Accessibility (WCAG 2.1 AA) is built into Radix primitives. |
| Animation | **Motion (Framer Motion v12)** | `import { motion } from "motion/react"` — the 2026 name. Use for page transitions, micro-interactions, and gesture-driven UI. Do not use CSS-only keyframes for anything more complex than a simple fade. |
| Icons | **Lucide React** | Tree-shakeable, consistent stroke-width, maintained. `import { IconName } from "lucide-react"`. One icon library per project — never mix sets. |
| Fonts | **`next/font`** | Zero layout shift, self-hosted, automatic subset. Always via `next/font/google` or `next/font/local` — never a `<link>` tag or CSS `@import` from Google Fonts in production. |
| Mobile | Expo + React Native | Managed workflow; Expo SDK handles native APIs |
| Server state (client) | TanStack Query v5 | For Client Components only; Server Components fetch directly |
| Client state | Zustand (slices) | One slice per domain; no God store |
| Backend (if custom) | **Hono** (edge/serverless) or **Fastify** (Node.js) or **Express 5** (familiar) | |
| BaaS | Supabase (Postgres + Auth + Storage + Realtime) | RLS from day one |
| ORM/Query | **Drizzle** (new projects) or **Prisma** (team preference); `supabase gen types` for PostgREST client types if on Supabase | |
| Validation | **Zod v4** | 14× faster than v3; same `z.infer<>` pattern |
| Background jobs (serverless) | **Inngest** | No Redis, Vercel-native, 50K free runs/month |
| Background jobs (long-running / AI) | **Trigger.dev v3** | Hours-long jobs, durable, self-hostable |
| Background jobs (high-throughput) | **BullMQ** + Upstash Redis | Only when > 100K/day and on persistent infra |
| AI streaming | Server-Sent Events (SSE) | Simpler than WebSocket for one-directional token streaming |
| Real-time fan-out | Supabase Realtime / Pusher / Ably | |
| Bidirectional real-time | WebSocket (`ws`) | Auth on HTTP upgrade, never after |
| Cache (app layer) | Upstash Redis | HTTP-based, works in serverless/edge |
| Logging | pino + Axiom or Better Stack (Vercel) / Grafana Loki (self-hosted) | pino is the fastest Node logger; Axiom/Better Stack have first-party Vercel integrations |
| Metrics | Prometheus + Grafana, or Grafana Cloud, or Datadog | |
| Tracing | OpenTelemetry | Provider-agnostic |
| Errors | Sentry | Frontend + backend; source maps at deploy |
| Analytics | PostHog | Open-source, self-hostable, GDPR-friendly |
| Auth | **Supabase Auth** (Supabase projects) / **Clerk** (best DX) / **Auth.js v5** (open-source Next.js) | Never roll your own |
| File storage | Cloudflare R2 (HMAC-signed URLs) | |
| Security headers (Next.js) | `next.config.ts` `headers()` for static headers; `proxy.ts` (v16) or `middleware.ts` (v15) for nonce-based CSP — cannot have both | Nonce + `'strict-dynamic'`; deploy Report-Only first |
| Security headers (Express/Fastify) | Helmet | |
| Payments (one-sided) | **Stripe** (hosted Checkout or Elements) | App-as-merchant — user pays the app for a subscription, plan, or one-time product. Hosted Checkout for fastest integration; Elements when checkout UI must live in-app. |
| Payments (marketplace / money routing) | **Stripe Connect** (destination charges by default; separate charges + transfers if platform handles disputes) | When money flows *between users through the product* (marketplace, tipping, payouts). Picks the Connect topology in ADR — `destination_charges` for simplest payout model; `separate_charges_and_transfers` when the platform must hold funds, handle disputes, or split across multiple destinations. Account onboarding via Express (default) or Standard (host-controlled). Never roll a marketplace topology by hand — Connect topologies are a one-way decision; pick deliberately in Genesis. |
| Payments (merchant-of-record alternative) | **Paddle** or **LemonSqueezy** | When the app sells globally and wants the processor to handle VAT/sales tax, invoicing, and merchant-of-record obligations. Picks an MOR over Stripe when international tax compliance is the dominant concern and revenue share is acceptable. |
| CI/CD | GitHub Actions | |
| Hosting | Vercel (web) + Fly.io / Render (custom server) + Supabase (DB) | |
| E2E | Playwright | |
| Unit/Integration | Vitest + Testcontainers (real DB) + MSW (network boundary mocking) | 70% unit / 20% integration / 10% E2E |
| Lint | **ESLint 9+** (flat config, `typescript-eslint` v8, `eslint-plugin-import-x/no-cycle`) or **Biome** (faster, all-in-one, simpler) | `next lint` removed in Next.js 16 — run the linter directly |

---

## Appendix B: Project Archetype Lenses

Most projects fit broadly into one of the archetypes below. During Genesis Step 5 (Domain Doc Enumeration) and Step 6 (Master Build Plan), pick the archetype(s) that match and use the lens to drive **additional domain docs, cross-cutting concerns, and phase ordering** beyond the generic Step 5 table.

Multiple archetypes can apply simultaneously (e.g. a marketplace that is also AI-native and multi-tenant). Combine the lenses.

The playbook does not codify everything in these lenses — they're starting prompts. Genesis must extract domain knowledge from the human (Step 1) and from publicly known patterns for each archetype.

### Archetype: Marketplace / two-sided platform

Examples: holiday rental booking, freelancer platforms, vehicle rental, peer-to-peer commerce, ride-sharing-style services.

| Concern | Why it matters | Where it lands |
|---|---|---|
| Stripe Connect topology (destination vs. separate charges + transfers) | Defines who holds funds, who owes refunds, who handles disputes — one-way decision | ADR + `NN-payments-and-marketplace.md` domain doc |
| Listing inventory + availability model | Date-range overlap prevention, hold/lock during checkout, race-condition windows | Cross-cutting concern — must precede booking flow phase |
| Multi-currency display + storage | Most marketplaces span regions — every price needs a currency code | Cross-cutting concern — Phase 1 / Phase 3 |
| Tax / VAT / GST per jurisdiction | Property/booking marketplaces specifically — country and city tax rates | Domain doc + integrated into pricing functions from day one |
| Host/provider KYC + payout schedule | Stripe Connect requires identity verification before payouts | Phase in plan + handoff entries for compliance |
| Review + rating system | Almost always present; affects schema (review tables, computed averages) | Dedicated phase |
| Calendar sync / channel manager (iCal in/out) | Property rentals specifically — guests come from Airbnb/VRBO during migration | Domain doc + dedicated phase |
| Anti-fraud (chargebacks, account takeover, fake listings) | Marketplaces are fraud magnets — moderation queue, velocity checks, identity signals | Phase or watchlist item depending on scale |

### Archetype: Multi-tenant SaaS / B2B with scoped admin

Examples: agency dashboards, property management software, healthcare records, CRM, project tools.

| Concern | Why it matters | Where it lands |
|---|---|---|
| Tenant model (organizations / workspaces / accounts) | Decides whether `tenant_id` is on every table (recommended) or workspace is implicit by user | ADR + domain doc + cross-cutting in Step 6a |
| Membership / assignment graph | Users belong to N tenants with R roles; queries always join through membership | `005-database.mdc` rule + domain doc |
| RLS that handles tenant + role + assignment (3 dimensions) | Naive RLS leaks data across tenants — a critical security failure | Dedicated phase before any feature that displays scoped data |
| Invite flow + role assignment UI | Almost always needed — admins invite teammates with specific permissions | Phase before admin-team features |
| Billing per tenant (seat-based, usage-based, or platform fee) | Connects to Q6 — but billing-per-tenant has different mechanics than per-user | Domain doc + dedicated phase |
| Audit log per tenant | Enterprise prospects always ask; SOC2 requires it | Cross-cutting from day one |
| Tenant impersonation for support | Required for any B2B product — staff must be able to view a tenant's UI without their password | Phase + handoff |
| Data export per tenant (GDPR / contractual) | Legal requirement in most jurisdictions for B2B | Pre-shipping handoff or dedicated phase |

### Archetype: Content / publishing / SEO-driven site

Examples: blogs, marketing sites with editorial content, listings directories, news, knowledge bases.

| Concern | Why it matters | Where it lands |
|---|---|---|
| CMS decision (headless vs. database vs. MDX-in-repo) | Determines who can edit content and how, plus deploy/preview model | ADR + domain doc |
| SEO from day one (metadata, structured data, sitemap, OG images) | Retrofitting SEO is feasible but inefficient — schema.org / OpenGraph in every layout | Cross-cutting; dedicated SEO phase |
| Static generation / ISR strategy | Affects framework features used (Next.js `revalidate`, ISR tags, cache headers) | Domain doc |
| Image optimization for content (CDN, formats, responsive) | Content sites are image-heavy; CDN + `next/image` planned from day one | Domain doc + asset system |
| Editorial workflow (draft / review / publish / schedule) | If editors are non-technical, they need a UI for this | Phase |
| Internationalization | Almost universal for content sites; URL strategy decided in Phase 1 | Cross-cutting (Q9) |
| Performance budgets (LCP, CLS, INP) | SEO ranking depends on Core Web Vitals — explicit budgets per page type | `009-observability-caching.mdc` + verification items |

### Archetype: AI-native product

Examples: AI-first SaaS, chat products, document analysis, AI agents.

| Concern | Why it matters | Where it lands |
|---|---|---|
| LLM provider topology (OpenAI / Anthropic / multi-provider / self-hosted) | Affects cost, fallback strategy, prompt portability, output schemas | ADR + `12-ai-and-ingestion-strategy.md` |
| Prompt injection defenses | Any user input that lands in a prompt is a security surface — must follow playbook §12 LLM Product Security | Cross-cutting (Trigger 4) + `006-backend.mdc` rule |
| Eval harness (Promptfoo / Inspect / hand-rolled) | Without offline evals, prompt changes cannot be safely shipped | Dedicated phase (playbook §14A) |
| Token budgeting + cost guardrails | Production AI costs spiral — per-user rate limits, max-token caps, model routing | Cross-cutting + observability |
| Streaming vs. batch + cancellation | UX expectation drives architecture; cancellation must be wired or costs leak | Domain doc |
| Caching layer for deterministic queries | LLM responses for identical prompts can be cached — big cost lever | `009-observability-caching.mdc` |
| RAG pipeline (chunk / embed / index / retrieve) — only if applicable | Each step has its own gotchas; cannot be retrofitted into a non-RAG architecture | Dedicated phase |
| Output schema validation (Zod against model output) | LLMs lie; every structured output goes through validation + repair loop | Cross-cutting |
| Per-user / per-tenant memory model | Long conversations exhaust context — decide truncation / summarization / vector recall upfront | Domain doc |

### Archetype: Real-time / collaborative / sync-heavy

Examples: collaborative editors, live dashboards, multiplayer apps, chat, presence systems.

| Concern | Why it matters | Where it lands |
|---|---|---|
| CRDT / OT / last-write-wins choice | Defines conflict resolution semantics; not interchangeable | ADR + domain doc |
| Presence channel topology | Pub/sub vs. WebSocket vs. CRDT awareness — affects backend choice | ADR |
| Offline / reconnect / sync resolution | Almost always required if mobile or unreliable networks; affects every write path | Cross-cutting |
| Optimistic update + rollback contract | Every UI write needs a rollback path; the patterns must be standardized | `008-styling-design.mdc` (UI) + `006-backend.mdc` (server) |
| Rate limiting on real-time channels | Easy DoS vector if not capped per connection | Backend rule + observability |
| Persistence + replay model | If clients fall behind, do they replay events or re-fetch state? | Domain doc |

### Archetype: Mobile-first or mobile-only

Examples: native consumer apps, on-the-go field tools, fitness, dating.

| Concern | Why it matters | Where it lands |
|---|---|---|
| Apple / Google developer accounts + Team IDs | Pre-shipping handoff items from day one — they take weeks to provision | `15-pre-shipping-handoff.md` |
| Push notification topology (APNs + FCM + Expo Push + back-end orchestration) | Cross-cutting; affects backend, schema, and consent flows | Dedicated phase + domain doc |
| Deep linking + universal links (AASA, assetlinks.json) | Requires production domain, Team ID, signing SHA — handoff items | Phase + handoff |
| App store review constraints (no external payment, age rating, in-app purchase rules) | Affects whether the app can even ship — must be confirmed in Genesis | Q9 hard constraints |
| Over-the-air updates (Expo Updates / CodePush) vs. full store releases | Affects release cadence and CI/CD | ADR |
| Permissions UX (camera, location, notifications, contacts) | Each requires explicit flows + fallback when denied | Domain doc per permission used |
| Offline-first architecture | Mobile apps need degradation paths for poor connectivity | Cross-cutting |

### Archetype: Internal tool / admin dashboard / one-client product

Examples: client-specific dashboards, internal ops tools, single-customer SaaS deployments.

| Concern | Why it matters | Where it lands |
|---|---|---|
| Auth simplicity (often just SSO with the company's IdP) | Don't over-engineer for "future users" who may never exist | Q9 — explicit decision |
| Lighter cursor rule budget | Solo team / few maintainers — fewer always-on rules, more glob-scoped | Step 7 cap |
| YAGNI is the dominant pressure | Internal tools accrete features without users to validate — anti-scope is critical | Q14 |
| Onboarding docs lighter than public products | One README + one runbook often enough | Step 4 conditional folder creation |

---

**How to use these lenses during Genesis:**

After Step 1 (Project Context Capture), open this appendix. Match the project to one or more archetypes based on the Q1, Q5, Q6, Q9, and Q11 answers. Walk the matched table(s). For each concern in the matched table:

- Add a domain doc if it doesn't already exist in the Step 5 list.
- Confirm it appears as a phase or as a step within a phase in the Step 6 plan.
- If it's a cross-cutting concern, confirm Step 6a's check #7 will catch it — that is, the phase that establishes it precedes any consuming phase.
- If it's a pre-shipping handoff item, add it to `15-…` during Step 10.

The archetype lens is the most reliable defense against the "playbook covers engineering but not domain patterns" gap. Use it deliberately at Genesis time; do not skip it on familiar-looking projects.

---

**Closing principle:** This document is a default, not dogma. Every rule has an exit ramp — but the exit ramp is an ADR with a clear reason, not a Slack message at 11pm.
