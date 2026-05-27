<!-- Split from general-engineering-playbook.md — content verbatim except where noted. See README.md for reading order. -->

<!-- File renamed from 01-start-here.md to reflect role: this is the action-first reference content (checkpoints, PR sanity test, anti-patterns, onboarding). The actual entry point for AI-driven projects is 00-genesis-and-build-cadence.md. -->

## 21A. Project Milestone Checkpoints

When you start a new project, work through these in order. Each checkpoint maps to specific sections — the playbook is the source of truth; this is the lookup table.

### Checkpoint 1 — Day 1 (Foundation, before any features)
Locking these in now is cheap. Retrofitting any of them later costs days to weeks.

- [ ] Stack chosen from Appendix; first ADR (`docs/adr/001-stack.md`) written (§17)
- [ ] pnpm + Turborepo monorepo with `apps/`, `packages/`, `tooling/` (§2)
- [ ] TS strict + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` (§3)
- [ ] Zod schema-as-types in `packages/validation/` (§3)
- [ ] Env validated at boot in `packages/config/env.ts`; nothing else reads `process.env` (§6, §17)
- [ ] `packages/assets/urls.ts` + `getAssetUrl(path)` helper with `MEDIA_BASE` env var (§8A)
- [ ] Component library (e.g. shadcn/ui) initialized in `packages/ui` — no raw HTML + Tailwind strings in feature code (§2)
- [ ] Database seeding script (`pnpm db:seed`) wired up so local development is deterministic (§7)
- [ ] Auth provider integrated (Supabase Auth / Clerk / Auth.js) — passkeys enabled (§10)
- [ ] `.cursor/rules/001-core-project.mdc` + `.cursor/rules/010-assets-cdn.mdc` + `CLAUDE.md` minimal (§16)
- [ ] `.cursorignore` / `.aiexclude` committed — `.env*`, `secrets/`, keys, dumps, build output excluded (§16)
- [ ] `docs/adr/NNN-trust-boundaries.md` lists the four boundaries (HTTP, DB, AI, third-party) (§1)
- [ ] Renovate / Dependabot enabled with grouped PRs; `pnpm install --frozen-lockfile` in CI (§16)
- [ ] `import-x/no-cycle` + flat ESLint config (or Biome) in CI (§15)
- [ ] First failing test runs; `pnpm typecheck` + `pnpm test` wired (§14)

### Checkpoint 2 — Pre-launch (Before the first real user)
Public surface; mistakes here are visible to users and search engines.

- [ ] Asset taxonomy ADR (`docs/adr/NNN-asset-taxonomy.md`) — every asset class assigned a bucket + cache rule (§8A)
- [ ] RLS enabled on every user-data table from migration 001; `(SELECT auth.uid())` pattern (§7)
- [ ] DB migrations applied via CI, not by hand (§7, §15)
- [ ] Every Server Action that mutates data calls `revalidatePath` or `revalidateTag` (§8)
- [ ] CSP deployed as `Content-Security-Policy-Report-Only`; R2 media domain in `img-src` (§12, §8A)
- [ ] Sentry frontend + backend; source maps uploaded on deploy (§13)
- [ ] PostHog typed events in `packages/observability/events.ts` (§13)
- [ ] Staging CF cache bypass rule at position 1; verified by CI health check (§8A)
- [ ] Two pipelines exist: code deploy + media upload (`upload:media`) — separate (§8A, §15)
- [ ] DNS records live: apex/www canonical, SPF + DKIM + DMARC (`p=none` to start), CAA, DNSSEC (§15, §19)
- [ ] SEO basics: `generateMetadata`, `sitemap.ts`, `robots.ts`, OG images per route (§8)
- [ ] `error.tsx`, `not-found.tsx`, `global-error.tsx` defined — no stack traces to users (§13)
- [ ] Cookie consent if any EU traffic; data export + deletion endpoints implemented (§12)
- [ ] LLM Product Security: prompt injection tests, tool auth checks, per-user token cap if the product calls an LLM (§12)
- [ ] LLM eval suite in CI for any feature in `packages/ai/**`; pass-rate regression blocks merge (§14A)
- [ ] §8A Pre-deploy Checklist all green
- [ ] §23 Five-Minute Sanity Test passes for every PR

### Checkpoint 3 — First paying user
Money flowing means double-charges, refund disputes, and chargebacks become real.

- [ ] Payment state machine, not boolean (§20)
- [ ] HMAC-validate every payment webhook before DB writes (§20)
- [ ] Webhook replay protection: timestamp window + event-ID idempotency table (§20)
- [ ] Idempotency keys generated once + stored in DB; reused on retry (§20)
- [ ] Subscription access gate is one server-side function — never client-side (§20)
- [ ] Trial enforcement server-side + IP ledger (§20)
- [ ] Transactional email separated from marketing domain; Google Postmaster Tools enrolled (§19)
- [ ] DMARC graduated from `p=none` to at least `p=quarantine pct=25` after 2-week monitoring (§19)
- [ ] Refund / dispute response runbook documented
- [ ] CSP switched from Report-Only to enforcement (§12)
- [ ] File upload validation (magic-number sniffing + size + dimension probe) for any user upload (§12)

### Checkpoint 4 — First 1K users (Scale + reliability)
Reactive firefighting becomes proactive only when these are measured and budgeted.

- [ ] Three SLOs defined; error budget tracked weekly (§13)
- [ ] Vercel Skew Protection enabled (or self-hosted chunk-recovery handler smoke-tested) (§8)
- [ ] Feature flag tool wired; every flag has owner + removal date (§15)
- [ ] Per-user debug logging admin UI in place (§13)
- [ ] DB PITR enabled + **first quarterly restore drill performed** (§7)
- [ ] R2/S3 object versioning ON for non-reproducible buckets (§7)
- [ ] Cost alerts: Vercel function-minutes, Supabase DB CPU/connections, R2 egress, LLM spend (§13)
- [ ] Cache tags deployed; `deploy-{buildId}` on HTML (§8A)
- [ ] Bundle-size budget enforced in CI (`bundlewatch` or equivalent) (§8)
- [ ] Core Web Vitals monitored in production; LCP/CLS/INP targets met on top 5 routes (§8)
- [ ] N+1 query check in integration tests for list endpoints (§7)
- [ ] First expand-contract migration completed cleanly (or pattern documented when needed) (§7)
- [ ] a11y baseline: `axe-core` in Playwright CI; WCAG AA contrast verified (§17)

---

## 23. The Five-Minute Sanity Test (run before every PR)

1. `pnpm typecheck` — clean?
2. `pnpm lint` (runs `eslint` or `biome lint` directly, **not** `next lint`) — clean (incl. `import-x/no-cycle`)?
3. `pnpm test` — passing?
4. Did I add a raw string for a route / role / event name / error code? Move it to the canonical types file.
5. Is any file I touched over 400 lines now? Split it.
6. If I added an architectural decision, is there an ADR for it?
7. If I added a silent fallback, is there an observability event for it?
8. If I added or changed an asset: declared bucket in `docs/adr/NNN-asset-taxonomy.md`? URL via `getAssetUrl` / `packages/assets` only? If Bucket 3 on staging/prod: did I run `upload:media` (and `promote:media` if prod-bound) — not only a git commit?
9. If I touched `packages/ai/**`: did evals run and pass? Did I change a prompt without versioning it? Did I add a new tool without an auth check on the handler?
10. If I added a new query on a growable table: is the `EXPLAIN ANALYZE` plan in the PR description? Did I use `SELECT *` (no — name the columns)?

If any of these is "no", the PR isn't ready.

---

## 22. Anti-Patterns to Avoid

- ❌ Raw `async function` exported as a Server Action. They lack structural auth checks. Use a safe action client (e.g. `next-safe-action` or a custom wrapper) that guarantees `auth.userId` is present and validates input before the handler runs.
- ❌ Concatenating user-controlled text directly into an LLM **system prompt**. That is the prompt-injection door. Untrusted text goes in user-role messages, wrapped in delimiters the system prompt instructs the model to treat as data. (§12)
- ❌ LLM tool/function-call handlers that trust the model's claimed identity or permissions. The model is an untrusted client — each tool runs the same auth check as a normal API route. (§12)
- ❌ Rendering model output as HTML without DOMPurify, or building URLs / shell commands / SQL from model output without parameterization. The AI edge is a third-party edge. (§12)
- ❌ Shipping an LLM-powered feature without an eval suite. Prompt drift will silently degrade quality and no test will fail. (§14A)
- ❌ Inline string-literal system prompts in service files. System prompts are versioned artifacts in `packages/ai/prompts/`. (§14A)
- ❌ Repo without `.cursorignore` / `.aiexclude`. The AI IDE will upload `.env`, secrets, and dumps to the vendor on every prompt. (§16)
- ❌ Accepting an AI's `pnpm add X` without verifying the package exists, is maintained, and has reasonable download volume. Slop-squat attacks ship malware via hallucinated package names. (§16)
- ❌ A long AI session (> 5 files touched, > 2 packages) without an externalized spec. For build-loop work, the spec is a phase in `docs/build/13-master-build-plan.md` (see `00-genesis-and-build-cadence.md`). For off-loop work, the spec is a `docs/tasks/T-NNN.md` task brief. Chat history degrades; the externalized spec is the only reliable scope guard. (§16)
- ❌ `SELECT *` in application code. Select only what you render — explicit columns keep the wire payload small and document the contract. (§7)
- ❌ A new query on a table likely to grow past 10K rows merged without an `EXPLAIN ANALYZE` plan in the PR description. (§7)
- ❌ Custom client-side router or component loader. The problem is solved.
- ❌ MemoryStore for sessions. Cannot survive restart or scale.
- ❌ `setTimeout` as a job queue.
- ❌ In-memory rate-limiter in a multi-process deployment.
- ❌ Standard Redis client in a Vercel/Edge serverless function. Use Upstash (HTTP-based Redis) instead.
- ❌ BullMQ on Vercel serverless without persistent infrastructure. Use Inngest instead.
- ❌ Helmet middleware in a Next.js app. Use `headers()` in `next.config.ts` for static headers; `middleware.ts` (v15) or `proxy.ts` (v16) for dynamic headers like CSP nonces.
- ❌ Server Action that mutates data without calling `revalidatePath` or `revalidateTag`. Pages will serve stale content silently.
- ❌ Manual `useState + loading + error` pattern for Server Action forms. Use `useActionState` (React 19 standard).
- ❌ `useFormState` — this is the deprecated React 18 canary name. The stable name is `useActionState`.
- ❌ `experimental.dynamicIO` or `experimental.ppr` flags — removed in Next.js 16. Replaced by `cacheComponents: true`.
- ❌ `unstable_cache` from `next/cache` — deprecated in Next.js 16. Use `"use cache"` directive instead.
- ❌ `next lint` as a CI command — removed in Next.js 16. Run `eslint` or `biome lint` directly.
- ❌ `middleware.ts` for network routing in a new Next.js 16 project. Use `proxy.ts` for network-level control.
- ❌ Adding `useMemo` / `useCallback` defensively without measuring a perf problem. With React Compiler enabled, this is dead code. Without it, it's noise.
- ❌ `.eslintrc.json` / `.eslintrc.js` in a new project. ESLint 9+ uses flat config (`eslint.config.mjs`). The old format is removed in ESLint 10.
- ❌ `@typescript-eslint/eslint-plugin` + `@typescript-eslint/parser` as separate packages. Replaced by the consolidated `typescript-eslint` v8.
- ❌ `Date.now()` or timestamps in payment idempotency keys. Keys must be stable across retries — generate a UUID once, store it, reuse it.
- ❌ Client-side enforcement of paid features. Always re-check server-side.
- ❌ JWT sessions rolled by hand. Use Supabase Auth, Clerk, or Auth.js.
- ❌ Logging raw email, passwords, or tokens.
- ❌ TTL-only cache invalidation for mutable data.
- ❌ Generated types and hand-written types for the same domain.
- ❌ Drizzle + `supabase gen types` for the same query. Drizzle types your SQL result; Supabase-generated types type the PostgREST client. They cover different code paths — use both, not one for both.
- ❌ `auth.uid()` directly in an RLS policy USING clause without a `SELECT` wrapper. Use `(SELECT auth.uid())` — the difference is 0.1ms vs 171ms on a 100K-row table.
- ❌ RLS policy on a column with no index. Always index the column being filtered.
- ❌ Server Component fetching multiple independent data sources sequentially with separate `await` calls. Use `Promise.all` — sequential awaits create waterfalls.
- ❌ Importing server-side code (DB queries, secrets) into Client Components without `import 'server-only'`. Add this guard to every server-only file.
- ❌ Host-allowlist-only CSP (e.g. `script-src 'self' https://cdn.example.com`). 95% of such policies are bypassable. Use nonce + `'strict-dynamic'`.
- ❌ Deploying a new strict CSP directly to production enforcement without a `Content-Security-Policy-Report-Only` testing phase first.
- ❌ Having both `middleware.ts` and `proxy.ts` in a Next.js 16 project root. The build fails — choose one.
- ❌ Business logic in `apps/` that could live in `packages/`. Apps should be thin routing/composition layers.
- ❌ Running E2E Playwright tests against the dev server. Always run against a production build (`next build && next start`).
- ❌ Mocking internal application modules (`vi.mock('./my-service')`) in integration tests. Mock at the boundary: Testcontainers for DB, MSW for external HTTP.
- ❌ CLAUDE.md absent from a project that uses Claude Code alongside Cursor. Both need their equivalent rule file.
- ❌ `any` or unwrapped `unknown`. Narrow or parse.
- ❌ A single `actions.ts` / `utils.ts` / `helpers.ts` that grows past 400 lines.
- ❌ A `shared/`, `common/`, or `lib/` package that becomes a graveyard.
- ❌ Architectural decisions deferred to "we can add this later."
- ❌ Product images, character art, personality cards, or any content-team-managed asset placed in `public/` with a stable filename and `Cache-Control: immutable`. Stable filename + immutable = human memory as the cache contract. Will eventually fail silently.
- ❌ `?v=N` as the primary cache-busting mechanism for product images. Works until someone forgets. Then a user is pinned on a stale image for up to a year with no diagnostic signal.
- ❌ Constructing CDN or R2 asset URLs inline in components or services (e.g. `` `https://media.domain.com/${charId}/${name}.webp` ``). Change the CDN domain or path structure and chase 20+ files. One construction site in `packages/assets/urls.ts`, zero exceptions.
- ❌ Pre-generating multiple image size variants (thumbnail, small, medium, large) at upload time when Cloudflare Image Resizing is available. Pipeline complexity with no benefit — one source file + URL transform parameter covers every size.
- ❌ Applying the same CDN cache rule to asset classes with different change frequencies (thumbnails vs. immutable video segments). Different change frequencies must have different rules; conflating them means either over-caching mutable assets or under-caching immutable ones.
- ❌ Using `purge_everything` zone-wide as a post-deploy step. Hits Cache Reserve, causes thundering herd on origin, invalidates unrelated assets. Use tag-based or prefix-based purge only.
- ❌ Staging environment excluded from long-cache rules but not covered by an explicit bypass rule. Falls through to a "respect origin Cache-Control" rule that still caches `immutable` R2 objects. Staging must have an explicit position-1 bypass rule.
- ❌ A new asset class shipped without a row in the asset taxonomy table (`docs/adr/NNN-asset-taxonomy.md`). Undocumented asset classes default to whatever pattern the next developer (or AI) finds nearby — which may be the wrong bucket.

---

## 21. Onboarding Checklist (for a new team member, week 1)

Day 1:
- [ ] Read `docs/build/README.md` (or equivalent index)
- [ ] Read `.cursor/rules/001-core-project.mdc`
- [ ] Read `docs/adr/` in order
- [ ] Clone, install (`pnpm install`), run the local stack (`pnpm dev`)
- [ ] Run the full test suite (`pnpm test`)

Day 2-3:
- [ ] Open the smallest open issue, ship a PR
- [ ] PR review against the rules: typecheck clean, no new cycles, no raw role/route strings, tests added (unit + auth rejection), file < 400 lines

Day 4-5:
- [ ] Pair with someone on a feature touching auth, database, and observability — the three layers that hide the most footguns
- [ ] Run a staging deploy end-to-end (DB migration → deploy → health check → rollback drill)

By end of week 2 they should be able to:
- Add a new route + service + repository + tests + ADR (if needed) without asking
- Write a useful Cursor rule from a recurring mistake they themselves made

---
