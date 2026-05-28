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
| Payments | Stripe (hosted checkout or Elements) | |
| CI/CD | GitHub Actions | |
| Hosting | Vercel (web) + Fly.io / Render (custom server) + Supabase (DB) | |
| E2E | Playwright | |
| Unit/Integration | Vitest + Testcontainers (real DB) + MSW (network boundary mocking) | 70% unit / 20% integration / 10% E2E |
| Lint | **ESLint 9+** (flat config, `typescript-eslint` v8, `eslint-plugin-import-x/no-cycle`) or **Biome** (faster, all-in-one, simpler) | `next lint` removed in Next.js 16 — run the linter directly |

---

**Closing principle:** This document is a default, not dogma. Every rule has an exit ramp — but the exit ramp is an ADR with a clear reason, not a Slack message at 11pm.
