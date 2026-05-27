<!-- Split from general-engineering-playbook.md — content verbatim. See README.md for reading order. -->

## 7. Database Layer

> **Why now:** Two main ORM/query paths in 2026, plus one Supabase-specific option:
> - **Drizzle** — TypeScript-first, zero runtime overhead, schema-as-code. The community preference for new TypeScript apps. Types are inferred directly from the schema definition — no codegen step, no separate runtime client.
> - **Prisma** — still the most widely adopted (largest ecosystem, most tutorials, most hirable experience). Prisma 5+ improved cold-start times. Valid if your team knows it or you need the mature ecosystem.
> - **Supabase-generated types** (`supabase gen types typescript`) — **a separate concern from Drizzle/Prisma**. If you use the Supabase JS client (PostgREST under the hood, not raw SQL), Supabase generates TypeScript types directly from your live schema. This is not an ORM — it types the client SDK's response shapes. You can use it *alongside* Drizzle (Drizzle for SQL queries, Supabase client + generated types for auth helpers, realtime subscriptions, storage). Do not confuse the two: Drizzle types your SQL query results; Supabase-generated types type the PostgREST client.
>
> **Decision for new projects:**
> - Using Supabase and want the JS client for auth/realtime/storage → run `supabase gen types` and commit the output. For SQL queries, add Drizzle.
> - Not using Supabase (self-hosted Postgres) → Drizzle is the default. Prisma if team preference.
>
> RLS at the database level (not application level) is now the standard for multi-tenant and user-data safety.

> **AI impact:** Schema-generated types mean the AI always works from the real database shape. It cannot write a query that selects a column that doesn't exist — the TypeScript types prevent it at compile time. The migration discipline (never edit applied migrations) means the AI can add a migration without fear of corrupting existing data — it just creates a new numbered file and the pattern is obvious from the existing ones.

- **PostgreSQL** (Supabase or self-hosted). Row-Level Security enabled on every user-data table from migration 001. RLS is not an afterthought — design policies alongside the schema.
- **Schema is the source of types.** Drizzle inferred types, Prisma codegen, or Supabase-generated types for the PostgREST client. No hand-written types that can drift from the actual schema.
- **Database Seeding is mandatory.** Every table requires a deterministic seed script. Local DB state must be instantly reproducible via `pnpm db:seed`. Never hardcode "mock data" into UI components; if the UI needs data to render, put it in the seed. This guarantees the AI can build real, data-driven components instead of hardcoded illusions.
- **Migrations are numbered, sequential, checked into git, applied by CI** (not by hand). Every migration has up + down. Never alter an applied migration — create a new one.
- **Audit-critical writes go through PostgreSQL functions** (one `RETURNS TABLE`, one transaction) so the primary write + audit log + status update succeed or fail together.
- **Connection pooling** sized to the database plan (PgBouncer or Supabase pooler), configured at deploy time, not guessed.

### RLS Performance Patterns (Supabase)

RLS is correct by default, but naively written policies destroy query performance at scale. These patterns prevent the most common production failures.

**1. Always wrap `auth.uid()` in a `SELECT` subquery — always.**
```sql
-- ❌ WRONG — auth.uid() executes once per row. On 100K rows: ~171ms
USING (auth.uid() = user_id)

-- ✅ CORRECT — treated as an initPlan, executes once per query: ~0.1ms
USING ((SELECT auth.uid()) = user_id)
```
Without the `SELECT` wrapper, PostgreSQL re-evaluates `auth.uid()` for every row during the policy check. The same rule applies to `auth.jwt()` and any security-definer function that returns a fixed value per query. This is the single most impactful RLS optimisation — a 100-1000× difference at scale.

**2. Index every column referenced in a policy.**
```sql
-- Every policy that filters on user_id needs this:
CREATE INDEX ON posts (user_id);
-- Multi-tenant: composite index beats two single-column indexes
CREATE INDEX ON posts (tenant_id, status);
```
A correct policy on an unindexed column still sequential-scans the table. Add the index before RLS is enabled in production — `EXPLAIN ANALYZE` in staging confirms index use.

**3. Limit policies per table per operation.**
Multiple permissive policies on the same table/role/action are evaluated separately — each adds overhead. Keep to **5 or fewer policies per table per operation**. Consolidate with `OR` conditions before adding a new policy.

**4. Use `SECURITY DEFINER` helper functions for complex permission checks.**
```sql
-- Complex multi-table joins in a policy cascade RLS overhead
-- Extract to a security definer function that runs once:
CREATE OR REPLACE FUNCTION user_owns_post(post_id uuid)
RETURNS boolean LANGUAGE sql SECURITY DEFINER STABLE AS $$
  SELECT EXISTS (SELECT 1 FROM posts WHERE id = post_id AND user_id = (SELECT auth.uid()));
$$;
```

**5. Add `TO authenticated` to every policy.**
```sql
-- Short-circuits policy evaluation for unauthenticated requests entirely:
CREATE POLICY "users can read own posts" ON posts
  FOR SELECT TO authenticated
  USING ((SELECT auth.uid()) = user_id);
```

**6. Use the Supabase Advisors tab.**
The Supabase dashboard (Project → Advisors → Performance) includes an `auth_rls_initplan` lint that scans all policies and flags every direct `auth.uid()` call not wrapped in `SELECT`. Run it before each release. Other useful lints: `multiple_permissive_policies`, `policy_exists_rls_disabled`.

**7. Test policies from the client SDK, not the SQL Editor.**
The SQL Editor runs as the `postgres` superuser which bypasses RLS. Always test access control from a client authenticated as the relevant role. The `service_role` key also bypasses RLS — never expose it client-side.

### Multi-Tenancy Patterns (if building B2B SaaS)

Choose your isolation model before the first schema migration — it is expensive to change after data exists. Three patterns, ordered by operational cost:

| Pattern | Isolation | Cost | Best for |
|---|---|---|---|
| **Pool** — shared DB + `tenant_id` column + RLS | Logical, DB-enforced | Lowest | Default start for B2B SaaS; scales to thousands of tenants |
| **Bridge** — schema-per-tenant in one DB | Schema-scoped | Medium | Dozens of large enterprise tenants needing stronger separation |
| **Silo** — separate DB per tenant | Physical | 10–50× higher infra | HIPAA / PCI / FedRAMP / contractual data residency requirements |

**Start with Pool + RLS.** Every business-data table gets a non-nullable `tenant_id` column, indexed, and RLS-enforced using the patterns in the section above. RLS is the safety net — even if a bug exists in application code, the database refuses to return another tenant's rows.

```sql
-- Every business table needs this from migration 001:
ALTER TABLE posts ADD COLUMN tenant_id uuid NOT NULL REFERENCES tenants(id);
CREATE INDEX ON posts (tenant_id); -- required for RLS performance (see above)

-- RLS policy — same pattern as user_id above:
CREATE POLICY "tenant isolation" ON posts
  FOR ALL TO authenticated
  USING ((SELECT auth.uid()) IN (
    SELECT user_id FROM tenant_members WHERE tenant_id = posts.tenant_id
  ));
```

**PgBouncer caveat:** In connection pool mode, use `SET LOCAL app.tenant_id = '...'` per transaction — not session-level `SET`. Session-level settings leak across pooled connections and are a data isolation bug.

**Migration path:** Start Pool → if an enterprise customer requires physical isolation for compliance, promote *that tenant* to a Silo while everyone else stays in the Pool. Schema-per-tenant (Bridge) is rarely the right choice: PostgreSQL catalog overhead scales poorly beyond ~5,000 schemas, and every DDL migration must run N times.

### Migration Safety — the Expand–Contract Pattern

> **Why this matters:** "Never edit applied migrations" prevents corruption; expand–contract prevents downtime. The mistake: rename a column in one migration → 30s window where old code hits new schema → 500s in production. Always treat a schema change as a sequence, not a single migration.

The three phases, each a separate deploy:
1. **Expand** — add the new column nullable, deploy code that writes to BOTH old and new columns. Old reads still work; new writes populate both.
2. **Backfill** — batched job populates the new column for all existing rows. Run idempotently in chunks; never lock the whole table.
3. **Contract** — once 100% of traffic is on new column, drop the old column in a new migration. Deploy the code that only reads/writes new column first; drop after.

**Rule:** any rename, type change, or non-null constraint addition follows expand → backfill → contract. Never a single migration. The cost is two extra migrations; the avoided cost is a 5-minute outage.

### Backups & Disaster Recovery

> **A backup that has never been restored does not exist.** This is the single most common operational miss.

- **PITR enabled** (Supabase: Project → Settings → Backups; self-hosted: WAL-G or pgBackRest). Default retention: 7 days minimum, 30 days for billing-critical data.
- **Restore drill quarterly.** Spin up an isolated DB, restore from a backup taken yesterday, verify a known query returns expected results. Document the RTO actually achieved.
- **R2 / S3 object versioning ON** for any bucket holding non-reproducible data (user uploads, paid licensed media). Set a lifecycle rule to expire non-current versions after 30-90 days.
- **Document RTO and RPO** — Recovery Time Objective (how long to restore) and Recovery Point Objective (how much data loss is acceptable). Numbers in `docs/runbooks/disaster-recovery.md`, not in someone's head.

### N+1 Queries — the most common Postgres killer in app code

ORMs make this dangerous to write by accident:

```ts
// ❌ N+1 — one query for posts, then N queries for users (one per post)
const posts = await db.posts.findMany({ limit: 50 });
const enriched = await Promise.all(posts.map(p =>
  db.users.findUnique({ where: { id: p.userId } })  // 50 extra round-trips
));

// ✅ One query with join (Drizzle / Prisma / Supabase all support this)
const posts = await db.posts.findMany({
  with: { author: true },   // Drizzle relations
  limit: 50,
});
```

**Rules:**
- Any list endpoint that fans out a per-row query is N+1. Use `with`, `include`, or an explicit `IN (...)` batch query.
- For cross-domain fetches that can't be joined in SQL, use a DataLoader pattern to batch+dedupe within one request.
- Detect in CI: most ORMs can log query counts per test; assert ≤ N queries per integration test for list endpoints.

### Query Performance Discipline

The N+1 rule above catches the most common AI failure mode, but it's not the only one. The AI will write queries that pass tests on a 100-row seed and tank in production on 1M rows.

- **`EXPLAIN ANALYZE` is mandatory** for any new query that touches a table likely to grow past 10K rows. Paste the plan into the PR description. If you see `Seq Scan` on a large table, add the index *before* merging.
- **`pg_stat_statements` enabled in staging + prod** from migration 001. Review the top-10 slowest queries weekly. This is the single cheapest way to find production performance regressions.
- **Every list endpoint has an integration test asserting query count ≤ N.** ORMs expose query counts; without this assertion the AI can refactor a fixed query into an N+1 and no test will fail.
- **No `SELECT *` in application code.** Select only the columns you render. The AI defaults to `SELECT *` because it's shorter — explicit column lists keep the wire payload small and document the contract.

---

## 8. Caching

> **Why now:** Next.js 16 (October 2025) is the current stable release and represents the biggest caching model shift since App Router launched. v15 moved to opt-in caching. v16 introduced **Cache Components** — the `"use cache"` directive is now stable (no longer experimental), Turbopack is the default bundler, and the old `experimental.ppr` / `experimental.dynamicIO` flags are removed and replaced by the unified `cacheComponents: true` config. Understanding the current model is essential — every stale-data, never-refreshes, and always-fresh bug in Next.js apps comes from misunderstanding which layer owns what.

> **AI impact:** The AI knows to call `revalidatePath` or `revalidateTag` at the end of every Server Action that mutates data. Without this, pages serve stale content silently after a mutation — one of the hardest bugs to diagnose. A Cursor rule explicitly stating this prevents the AI from forgetting it.

### Next.js App Router — Four Framework-Managed Layers

These are **not HTTP headers you set manually**. Next.js and the platform manage them. Your job is to configure the right behaviour per route and invalidate correctly after mutations.

| Layer | What it stores | Where | Owner |
|---|---|---|---|
| **Request Memoization** | Identical `fetch()` results within one render | Server memory | Automatic — deduplicated per render, no config needed |
| **Data Cache** | `fetch()` results / `"use cache"` scope outputs across requests | Server (persistent) | You control with `cache`, `revalidate`, or `"use cache"` directive |
| **Full Route Cache** | Rendered HTML + RSC payload per route | Server (persistent) | Determined by route config (`dynamic`, `revalidate`, or `"use cache"`) |
| **Router Cache** | RSC payloads of visited routes | Client memory | Automatic; invalidated by `revalidatePath`/`revalidateTag`/`updateTag` |

**What "RSC payload" means:** Client-side navigation in Next.js does NOT reload HTML. It fetches a serialised component tree (RSC payload) from the server. This is what gets cached in the Router Cache and what Skew Protection pins to the correct deployment.

### Version history that matters for debugging

- **Next.js v14:** `fetch()` cached by default (opt out with `cache: 'no-store'`).
- **Next.js v15:** `fetch()` **not cached by default** (opt in with `cache: 'force-cache'` or `revalidate: N`).
- **Next.js v16 (current, October 2025):** Same default as v15 for existing projects. If you enable `cacheComponents: true`, data fetching becomes dynamic by default and you explicitly opt-in caching with `"use cache"`. **Know your version before debugging stale/fresh data.**

### Route Caching Strategy — Three Patterns (applies to all versions including v16)

Choose one per route segment:

```ts
// 1. STATIC — built once at deploy time. Best for: docs, landing, marketing.
// Default when no dynamic functions (cookies, headers, searchParams) are used.
export default async function Page() { ... }

// 2. ISR — rebuilt in background after N seconds.
// Best for: public content that changes infrequently (blog, events list).
export const revalidate = 3600;
export default async function Page() { ... }

// 3. DYNAMIC — fresh on every request. Best for: auth pages, admin, dashboards.
// Also triggered automatically by cookies(), headers(), or searchParams usage.
export const dynamic = 'force-dynamic';
export default async function Page() { ... }
```

### Cache Components (Next.js 16 — opt-in via `cacheComponents: true`)

The `"use cache"` directive is **stable in Next.js 16**. Enable it with one config line:

```ts
// next.config.ts
export default { cacheComponents: true } satisfies NextConfig;
```

When enabled, **all data fetching is dynamic by default** (runs at request time). You explicitly mark what should be cached:

```ts
// Cache a Server Component's output
'use cache';
export default async function ProductList() {
  const products = await db.products.findMany();
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// Cache a data function, not a whole component
async function getProducts() {
  'use cache';
  cacheTag('products');          // tag for invalidation
  cacheLife('hours');            // built-in duration preset
  return db.products.findMany();
}

// Invalidation still uses revalidateTag — same as before
revalidateTag('products');       // invalidate by tag
updateTag('products');           // new v16 API — fine-grained control
```

> **Migration note:** `"use cache"` replaces `unstable_cache`, `experimental.dynamicIO`, and `experimental.ppr`. Do not mix the new and old APIs. Enable `cacheComponents: true` on staging first — it changes default rendering behaviour, so pages that relied on v15 implicit static generation may start running dynamically.

### After Mutations — Always Invalidate

This is the most common source of bugs in Next.js apps. Every Server Action that writes to the database must invalidate the relevant cache entries:

```ts
'use server';

export async function updatePost(id: string, data: PostData) {
  await db.posts.update(id, data);
  revalidateTag('posts');          // invalidate by tag — precise
  // or:
  revalidatePath('/blog');         // invalidate by path — broader
}
```

**Rule:** No Server Action that mutates data ends without a `revalidatePath` or `revalidateTag` call. This is enforced by code review and a Cursor rule.

**React 19 form integration (use this pattern — not raw `useState + fetch`):**
```ts
// Return errors as data, not thrown exceptions
export async function createItem(prevState: ActionState, formData: FormData) {
  const result = itemSchema.safeParse(Object.fromEntries(formData));
  if (!result.success) return { errors: result.error.flatten().fieldErrors };
  await db.insert(result.data);
  revalidateTag('items');
  return { success: true };
}
// In the component:
const [state, formAction, isPending] = useActionState(createItem, {});
```
`useActionState` gives you validation errors, pending state, and progressive enhancement (works without JS) in one hook. This is the React 19 standard — it replaces the `useState + onSubmit + loading` pattern entirely.

### Server Component Data Fetching — Avoid Waterfalls

Sequential `await` calls in a Server Component create a waterfall — each call waits for the previous before starting. For independent data sources, always fetch in parallel:

```ts
// ❌ WATERFALL — 600ms total (3 × 200ms sequential)
export default async function DashboardPage() {
  const user    = await getUser();
  const events  = await getEvents();
  const metrics = await getMetrics();
  return <Dashboard user={user} events={events} metrics={metrics} />;
}

// ✅ PARALLEL — 200ms total (all three start simultaneously)
export default async function DashboardPage() {
  const [user, events, metrics] = await Promise.all([
    getUser(),
    getEvents(),
    getMetrics(),
  ]);
  return <Dashboard user={user} events={events} metrics={metrics} />;
}
```

**Rule:** Any Server Component that calls more than one independent data source must use `Promise.all`. If one source depends on another's result, that's a sequential dependency — sequential is correct there. Use `Suspense` boundaries to stream slow data sources independently rather than blocking the whole page.

### For Non-Next.js / Custom Servers (Express, Fastify)

If you're running a traditional server (or a Vite SPA with a separate API), the rules are different — you set these headers yourself:
- **API responses**: `Cache-Control: no-cache` (ETag revalidation). Never CDN-cache dynamic API responses.
- **Static JS/CSS assets**: `Cache-Control: public, max-age=31536000, immutable` (content-hashed by build tool).
- **HTML**: `Cache-Control: no-cache`. If you CDN-cache HTML, rollbacks are invisible to users.
- **Private media**: HMAC-signed short-expiry URLs from a private bucket. Never serve through the app server.

### Application Cache (Redis) — When Justified

For expensive DB queries or external API calls that can't use Next.js data cache (e.g., in background jobs, webhooks, non-fetch data sources):
- **Cache-aside**: read → check Redis → miss → DB → write Redis. Write → DB write → **delete Redis key** (not update, to avoid stale-write races).
- **Explicit invalidation** — TTL-only is wrong for mutable data. Tag the key and delete it on write.
- Don't add Redis until a specific query is measurably slow. Next.js's Data Cache covers most data-fetching needs.

### Database Indexes
- Every foreign key has an index. Every `WHERE`-clause column hit at user-request time has an index.
- Add indexes *before* the query is slow in production. Investigate with `EXPLAIN ANALYZE` in staging.

### Stale-Chunks Strategy

**The problem:** A user has your app open in a tab. You deploy. New chunks have new hashed filenames. They navigate to a lazy-loaded route — the old chunk filename is gone — `ChunkLoadError`. That's the entire problem.

**This is a solved problem in 2026 — don't re-implement it manually if you don't have to.**

**On Vercel (the default stack):** Enable **Skew Protection** in your project's Settings → Advanced. Generally available since March 2024, default for all new projects since November 2024. It routes each client session to the exact deployment it loaded from, so old chunk filenames remain valid for the deployment's lifetime. Zero implementation work. This also covers Next.js Server Actions and RSC — the right server code always responds to the right client version.

**On self-hosted deployments (Fly.io, Docker, Cloudflare Pages without Vercel):** Vercel's platform-level fix isn't available, so you implement it:
1. Build must emit content-hashed chunks. Vite and Next.js do this by default — never disable it.
2. Keep old chunk files on your CDN/origin for 48h after deploy before deleting them.
3. Add a `ChunkLoadError` boundary that catches lazy-load 404s and triggers a hard reload.
4. Optionally: fetch a `/build-id.txt` on tab focus to show a non-blocking "Update available" banner for UX — **never auto-reload mid-interaction**.

> Note: Next.js App Router is not a pure SPA — client navigation issues RSC server requests, so the server is always involved in route transitions. Next.js also added built-in chunk load retry logic (automatically retries a failed chunk fetch once before surfacing an error). The residual risk for self-hosted Next.js is genuinely small; the risk is higher for pure Vite SPAs.

**Two non-negotiables regardless of platform:**
- HTML must never be CDN-cached. `Cache-Control: no-cache` + ETag always. If you cache HTML, rollbacks are invisible to users.
- Content-hashed filenames for all JS/CSS assets with `Cache-Control: public, max-age=31536000, immutable`. Build tools handle this automatically.

### Bundling Strategy

> **Why now:** Turbopack is the **default bundler in Next.js 16** (stable, production-ready). It replaces Webpack with 2-5× faster builds and up to 10× faster Fast Refresh. The main mistake teams make is not the lack of code splitting — it's bundling heavy, rarely-used dependencies (admin panels, chart libraries, rich text editors) into the shared entry chunk that every page visitor pays for.

**Rules:**
- **Route-level splitting is automatic.** Next.js App Router (Turbopack) and Vite's router both create per-route chunks. Never override this.
- **Lazy-load non-critical UI.** In Next.js use `dynamic(() => import(...), { ssr: false })`. In Vite use `React.lazy` + `Suspense`. Target: any dependency > 50KB needed on only one route.
- **Do not manually configure Webpack `splitChunks` in a Next.js project.** Turbopack handles vendor extraction automatically. If you're on Vite (non-Next.js), `manualChunks` in `vite.config.ts` is the lever, but only after measuring.
- **Analyse before adding a dependency.** Use `@next/bundle-analyzer` (Next.js) or `vite-bundle-visualizer` (Vite). A dependency that adds > 50KB to the entry chunk requires an explicit decision — lazy-load it or find a lighter alternative.
- **Tree-shaking requires ESM.** Prefer named imports (`import { Button } from "ui"` not `import * as UI from "ui"`). Avoid CommonJS-only packages in the client bundle.
- **No duplicate implementations.** If a utility exists in `packages/validation`, import it everywhere. Writing the same logic twice means two bundle weights, two bug surfaces, and eventual drift.

### Performance Budgets & Core Web Vitals

Bundle rules without targets are aspirational. Set numbers and enforce them in CI:

| Metric | Target (2026 Google "Good") | How to measure |
|--------|------|------|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | Vercel Speed Insights / PageSpeed Insights / `web-vitals` lib |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | same |
| **INP** (Interaction to Next Paint) | ≤ 200ms | same (replaced FID in March 2024) |
| **JS per route (compressed)** | ≤ 200 KB | `@next/bundle-analyzer` or `bundlewatch` in CI |
| **Image: dimensions declared** | always | prevents CLS — use `next/image` or explicit width/height |

**CI gate:** `bundlewatch` (or equivalent) fails the PR if route bundles exceed budget by > 10%. Without this gate, bundle size only goes up.

### SEO & Metadata (Next.js)

Public-facing products lose months of organic traffic skipping this.

- **`generateMetadata()` per route** in App Router — title, description, OG image, canonical URL.
- **`app/sitemap.ts`** generates sitemap.xml dynamically; submit to Google Search Console.
- **`app/robots.ts`** controls crawler access; never `Disallow: /` in production.
- **Structured data (JSON-LD)** for products, articles, FAQs — eligible for rich results.
- **OG image** per route — use `@vercel/og` to generate dynamically at the edge.
- **Canonical URLs** explicitly set when filters/sort params shouldn't index separately.

---

## 8A. Asset Taxonomy & Edge Delivery

> **Stack scope:** The three-bucket model and construction-site pattern in this section are universal — they apply regardless of CDN or storage provider. The implementation details (R2, Cloudflare Image Resizing, CF Cache Rules, CF Cache Tags) are Cloudflare-specific and apply when Cloudflare is your CDN and object storage layer. For AWS S3 + CloudFront, substitute: S3 presigned PUTs, CloudFront Functions for image transforms, `Cache-Tag` → CloudFront cache invalidation via tag (same concept, different API). For Vercel Blob + Vercel CDN, substitute: `@vercel/blob` presigned uploads, `next/image` for transforms, `revalidateTag` for CDN invalidation. The three-bucket model and construction-site rule are the same on every stack.

> **Why now:** The single most common source of caching bugs in production — "users still see the old image / file / character after we deployed" — is not a CDN misconfiguration. It is a taxonomy failure: the wrong caching contract was applied to an asset class because nobody wrote down the contract before the first feature shipped. Cloudflare R2 (zero egress fees, 2022), Cloudflare Images (on-demand transforms, 2022), and cache tag purging on all CF plans (April 2025) remove every excuse for not getting this right from day one.

> **AI impact:** Without an explicit taxonomy, an AI agent adds a new asset by following the nearest existing pattern — which may be the wrong bucket. With a taxonomy table committed to the repo, the AI always knows which bucket a new asset class belongs in and what cache rule to apply. It cannot accidentally put a product image in `public/` with an `immutable` header and a stable filename, because that combination is explicitly banned in the rules.

---

### The Three-Bucket Model

Every asset in every project belongs to exactly one of three buckets. The bucket determines the URL strategy, the cache rule, and where the file lives. There are no exceptions and no fourth bucket.

```
┌──────────────────────────────────────────────────────────────────┐
│ BUCKET 1 — Bundler decides the URL                               │
│ What:    App JS, CSS, fonts, images imported in source code      │
│ How:     Build tool emits content-hashed filenames automatically  │
│ Cache:   immutable, max-age=31536000 — zero manual work          │
│ Rule:    Use import statements; never reference these via /public/│
│ Examples: main-a3f8b2.js  styles-c4d9.css  logo-fe21.svg        │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ BUCKET 2 — Deploy decides the URL                                │
│ What:    HTML entry points, config JSON, build manifests         │
│ How:     Stable URL, always fresh from origin                    │
│ Cache:   no-cache (HTML) / no-store (config, manifests)         │
│ Rule:    Never CDN-cache HTML. No exceptions.                    │
│ Examples: index.html  /config/app.json  /build-id.txt           │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ BUCKET 3 — Content decides the URL                               │
│ What:    ANY asset that can change independently of a code deploy │
│ How:     Object storage, UUID or hash key — new file = new URL   │
│ Cache:   immutable, max-age=31536000 — structurally safe         │
│ Rule:    Object storage; key is UUID, content hash, or git-mirrored path │
│ Examples: r2.com/.../personality1.webp  or  r2.com/.../uuid.webp │
└──────────────────────────────────────────────────────────────────┘
```

Bucket 3 has **two maturity levels** (same bucket, different ops) — see "Bucket 3 maturity" below. Do not skip straight to UUID keys + CMS on day one if a dev-owned git → R2 pipeline is enough.

**The diagnostic question for any asset:** "Can this file change without a code deploy?"
- No → Bucket 1 (if build-tool-importable) or Bucket 2 (if it needs a stable URL).
- Yes → Bucket 3, always. No exceptions.

**The fatal anti-pattern:** Bucket 3 content (product images, character art, user media) placed in a framework's `public/` directory with `Cache-Control: immutable` and a stable filename. This is immutable + stable URL — a promise to browsers and CDNs that the bytes will never change, backed only by human memory. It will eventually fail. See "Why `public/` is not Bucket 3" below.

---

### Why `public/` is not Bucket 3

Every major bundler (Vite, Webpack, Turbopack/Next.js) has a `public/` directory. Files there are **copied verbatim** to `dist/` — no processing, no hashing, no transforms. This is intentional and documented. Vite's own docs: *"files in the public directory lack automatic cache-busting mechanisms."* Next.js behaves identically.

`public/` is correct for exactly two things:
1. Files that must retain their exact filename (`robots.txt`, `favicon.ico`, `apple-touch-icon.png`).
2. Files that truly never change and whose staleness has no user-visible consequence.

Everything else that might change — product images, character thumbnails, promo banners, personality cards — belongs in Bucket 3 (object storage via the media CDN; see Maturity A/B below for key scheme) or behind Bucket 1's import pipeline. If you are using `public/assets/product-image.webp` in production and serving it from the app origin with `immutable`, you have already violated the model.

---

### Write the Taxonomy Table Before Shipping Feature One

Takes 30 minutes. Saves weeks of debugging. Commit it as `docs/adr/NNN-asset-taxonomy.md`.

Every row below is an example — fill in your actual asset classes:

| Asset class | Bucket | Lives in | Key scheme | Changes how | Cache rule |
|---|---|---|---|---|---|
| App JS / CSS | 1 | Bundler `dist/assets/` | Content hash in filename | Every build | `immutable, max-age=31536000` |
| Imported images / fonts | 1 | Bundler `dist/assets/` | Content hash in filename | Every build | `immutable, max-age=31536000` |
| HTML entry points | 2 | App server | Stable URL | Per deploy | `no-cache` |
| Config / manifests | 2 | App server or KV | Stable URL | Per build | `no-store` |
| Product thumbnails | 3 | R2 (git mirrors paths pre-CMS) | Stable path or UUID | Upload / new key | `immutable, max-age=31536000` |
| User-uploaded media | 3 | R2 / object storage | UUID at upload | Immutable after upload | `immutable, max-age=31536000` |
| Marketing/promo images | 3 | R2 / object storage | UUID at upload | New campaign → new key | `immutable, max-age=31536000` |
| Voice / audio files | 3 | R2 / object storage | UUID at upload | Immutable after upload | `immutable, max-age=31536000` |

**Critical invariant:** two asset classes with different change frequencies must have different cache rules. Do not give the same `max-age` to a thumbnail that gets updated and a video that is immutable after upload. Separate them into separate CDN rules.

---

### Bucket 3 maturity — two operating models

| | **Maturity A — Dev-owned (pre-CMS)** | **Maturity B — DB / content-team (scale)** |
|---|---|---|
| **Who changes assets** | Engineers | Content ops, CMS, or users |
| **Source of truth** | Git (paths + bytes in repo) | DB row (`publicUrl`) + object storage |
| **R2 key** | Stable path mirroring git (`characters/lana/...`) | UUID (or content hash) per upload |
| **App resolves URL via** | `getAssetUrl(path)` + `MEDIA_BASE` | DB lookup; helper only for transforms |
| **Promote to prod** | Upload to staging R2 → sync/promote to prod bucket | New object + DB update; no bucket clone |
| **When to adopt** | Launch → first content team | Non-devs publish, or cache-safe URLs required |

Maturity A is professional and common for early teams. Maturity B is the end state — not a rejection of A. Graduate when **who** changes assets changes, not because local-first was wrong.

---

### Bucket 3 in Practice: R2 + Cloudflare CDN

**Maturity A (dev-owned pipeline)** — full runbook in "Dev-owned content pipeline" below.

**Maturity B (default at scale)** — for any project on the Cloudflare stack:

```
Upload           App server generates a presigned PUT URL (5-min TTL)
                 Client uploads directly to R2 — never proxied through app server
                 R2 object key = UUID (e.g. ulid() or crypto.randomUUID())
                 R2 object metadata: Cache-Control: public, max-age=31536000, immutable

Store            App server records { assetId, r2Key, publicUrl } in database
                 publicUrl = https://media.yourdomain.com/<uuid>.<ext>

Serve            Custom domain on R2 bucket → Cloudflare CDN in front
                 CF Cache Rule: Cache Everything, Respect Origin TTL, Cache Reserve ON
                 Result: edge-cached immutable object, zero egress fees

Update           Upload new R2 object (new UUID) → update one database record
                 Old URL remains valid for any client that has it cached
                 New URL is immediately live for all new requests
                 No CDN purge. No manifest. No version bump. Structurally safe.
```

**Why presigned PUT (never proxy through app server):**
- Workers and Node processes have memory limits (128 MB on CF Workers, practical limits on Node).
- Client → R2 is one network hop; client → server → R2 is two.
- Direct upload goes to the nearest R2 PoP (Local Uploads, 2026 beta: ~75% lower write latency globally).
- The app server stays stateless and horizontally scalable.

---

### Cloudflare Images: Eliminate Pre-generated Thumbnail Variants

> **Why this matters:** Pre-generating multiple size variants of every product image (thumbnail, small, medium, large, blur placeholder) at upload time requires a pipeline (ffmpeg, cwebp, ImageMagick, Python scripts), storage for each variant, and re-running the pipeline every time the crop or quality changes. Cloudflare Images (and the free `cf-resizing` transform via Image Resizing on Pro+) eliminates this entirely.

**How it works:** Store one source file at full resolution in R2. Request any transform via URL parameters. Cloudflare performs the transform on first request, caches the result at every edge PoP, and serves WebP or AVIF automatically based on the browser's `Accept` header.

```
Source stored once in R2:
  https://media.yourdomain.com/a3f8b2c1-uuid.jpg  (original, full resolution)

Transforms via URL — Cloudflare serves and caches each variant:
  /cdn-cgi/image/width=625,height=399,fit=crop,gravity=face/a3f8b2c1-uuid.jpg
  /cdn-cgi/image/width=320,height=204,fit=crop,gravity=face/a3f8b2c1-uuid.jpg
  /cdn-cgi/image/width=64,height=64,fit=cover,blur=20/a3f8b2c1-uuid.jpg

Browser receives:
  - WebP if browser supports it (Chrome, Firefox, Edge, Safari 14+)
  - AVIF if browser supports it (Chrome 85+, Firefox 93+) — ~30% smaller than WebP
  - JPEG/PNG fallback otherwise
  All served from nearest edge PoP. Zero app server involvement.
```

**`gravity=face`** — Cloudflare's Image Resizing supports face detection for crop centering. The CDN finds the face and centers the crop automatically. This replaces custom Python/OpenCV/YuNet face detection pipelines.

**The transform URL is the cache key.** Different widths produce different cache entries. Each variant is cached `immutable` at the edge. No pre-generation pipeline. No storage for size variants. No re-processing when you change a crop — change the URL parameter.

**Setup:** Enable Image Resizing in the CF dashboard (Zones → Speed → Optimization → Image Resizing). On Pro plan and above. Serve source images from R2 via a custom domain.

**Cloudflare Images product** (separate from Image Resizing): Paid image management product with a delivery URL scheme (`imagedelivery.net`), variant presets, and its own storage. Use this if you need a CMS-style image pipeline or want managed storage + transforms in one product. For most projects, Image Resizing on your own R2 bucket is sufficient.

---

### One Construction Site Per URL Class

This is the single rule that costs nothing to implement on day one and costs everything to retrofit later.

**Never write an asset path directly in a component. Always go through a function.**

```js
// ❌ what kills you later — scattered across 40 components
<img src="/assets/characters/lana/thumbnail.webp" />

// ✅ what saves you — one call site, one place to change
<img src={getAssetUrl('characters/lana/thumbnail.webp')} />
```

**How it evolves across three stages — zero component changes at each transition:**

```js
// Stage 1 — local dev. Written on day 1. Takes 2 minutes.
export const getAssetUrl = (path) => `/assets/${path}`;

// Stage 2 — staging / production read from CDN/R2.
// Change one env var per environment. Do NOT use NODE_ENV for this — it conflates
// "minify JS" with "where images live". Use MEDIA_BASE or APP_ENV instead.
export const getAssetUrl = (path) => `${process.env.MEDIA_BASE}/${path}`;
// local:   MEDIA_BASE=/assets
// staging: MEDIA_BASE=https://media-staging.yourdomain.com
// prod:    MEDIA_BASE=https://media.yourdomain.com

// Stage 3 — R2 + Cloudflare Image Resizing (on-demand transforms).
// Still one function. Components still unchanged.
export const getAssetUrl = (path, opts = {}) => {
  const base = process.env.MEDIA_BASE;
  if (!opts.width) return `${base}/${path}`;
  const { width, height, fit = 'cover', gravity } = opts;
  const params = `width=${width},height=${height},fit=${fit}${gravity ? `,gravity=${gravity}` : ''}`;
  return `${base}/cdn-cgi/image/${params}/${path}`;
};
```

**The practical point:** you can start with Stage 1 on the first day of the project using local files, and migrate to Stage 2 or Stage 3 by changing one function. Because no component ever constructed a URL directly, nothing else needs to change. This is the only architectural decision about assets that is genuinely expensive to retrofit — everything else (R2 migration, CDN config, image transforms, admin panels) can be added incrementally when you actually need it.

**In a typed TypeScript codebase** (the production version of this pattern):

```ts
// packages/assets/urls.ts — the ONLY place asset URLs are constructed

export function getMediaUrl(key: string): string {
  return `${Config.MEDIA_CDN_BASE}/${key}`;
}

export function getImageTransformUrl(
  key: string,
  opts: { width: number; height: number; fit?: string; gravity?: string }
): string {
  const params = new URLSearchParams({
    width: String(opts.width),
    height: String(opts.height),
    fit: opts.fit ?? 'cover',
    ...(opts.gravity ? { gravity: opts.gravity } : {}),
  });
  return `${Config.MEDIA_CDN_BASE}/cdn-cgi/image/${params}/${key}`;
}
```

**Why this is non-negotiable:**
- CDN domain change = one constant, not a find-and-replace across 40 files.
- Moving from local to R2 = one environment variable, not a refactor.
- AI agents follow the existing pattern — if the construction site is one function, they will call it. If construction is scattered, they will add a 21st variant.

Enforce as a lint rule: flag any file outside `packages/assets/urls.ts` (or wherever the construction site lives) that contains a raw string matching `https://media\.|cdn-cgi/image` or a hardcoded `/assets/` path in a JSX attribute. The violation is caught in CI, not in a post-deploy incident.

---

### Dev-owned content pipeline (Bucket 3, Maturity A)

Use this from first launch until a CMS or non-dev publishers are required. It implements Bucket 3 delivery without UUID keys or a media admin on day one.

**Principle:** Git holds what you intend to ship. Object storage holds what each environment actually serves. The app never hardcodes paths — only `getAssetUrl(path)` + `MEDIA_BASE`.

```
SOURCE (git)     Files under public/assets/... (or a dedicated assets/ tree in repo)
                 Commit + PR = reviewable history of art changes

LOCAL            MEDIA_BASE=/assets  →  reads filesystem, no R2 required for UI work

STAGING          After asset change:  pnpm upload:media --env staging
                 MEDIA_BASE=https://media-staging.yourdomain.com
                 Separate R2 bucket (or prefix); staging CDN has cache bypass (§8A)

PRODUCTION       After staging sign-off:  pnpm promote:media
                 → sync changed keys from staging bucket → prod bucket
                 (full bucket mirror is OK when libraries are small; prefer delta sync as you grow)
                 MEDIA_BASE=https://media.yourdomain.com
                 App redeploy NOT required if paths unchanged — only bytes on CDN changed
```

**Two pipelines — keep them separate:**

| Pipeline | Trigger | Delivers |
|---|---|---|
| **Application** | PR merge, CI | JS/CSS/HTML, API, config |
| **Media** | Asset files changed in git | Bytes to staging R2 → promote to prod R2 |

Same person may run both; they are still two steps. Forgetting `upload:media` after a git commit is the main failure mode — mitigate with a PR checklist, CI job on paths under `public/assets/`, or a script that warns when git hashes ≠ R2 ETags.

**Environment variable — not `NODE_ENV`:**

| Environment | Typical `MEDIA_BASE` | Serves from |
|---|---|---|
| Local | `/assets` | Repo filesystem |
| Staging | `https://media-staging...` | Staging R2 + CDN |
| Production | `https://media...` | Production R2 + CDN |

`NODE_ENV=production` on staging builds is normal; it must not imply "use prod media bucket."

**Cache when overwriting the same R2 key:** If keys stay stable (`personality1.webp`), replacing bytes in place with `immutable` CDN caching can leave users on old art. The upload script or promote step must include an explicit bust strategy: new key in R2, `Cache-Tag` purge for that path, or documented rename — not "hope someone remembers."

**Do not promote to production from staging:**

- User-generated / licensed tip media (prod-only bucket or DB URLs).
- Staging database or test users (unless intentional).

**Graduate to Maturity B (brief):** When non-devs need to publish without git: CI uploads on merge to a **manifest of changed keys**; uploads use **UUID keys**; **DB stores `publicUrl`**; optional headless CMS (Sanity, Contentful) or internal admin → presigned PUT → R2. Stable-path git mirror retires for classes that move to CMS; Bucket 1 (imported UI chrome) unchanged.

---

### Cache Tags for Surgical Invalidation

> **Naming disambiguation — two entirely different systems share the term "cache tag":**
> - **`Cache-Tag` HTTP response header** (this section) — a Cloudflare CDN feature. You set this header on origin responses; Cloudflare strips it before delivery but indexes the content by tag. One CF API call purges every CDN edge entry carrying that tag. This operates at the **HTTP/CDN layer** and works for any content, including images, video, and JSON.
> - **`cacheTag()` / `revalidateTag()` Next.js API** (see §8) — a Next.js Data Cache feature. You call `cacheTag('products')` inside a `"use cache"` function; calling `revalidateTag('products')` invalidates that function's cached output on the **Next.js server**. This is server-side only and has nothing to do with the CDN or browser cache.
> They are not in conflict — a single response can be tagged at both layers — but they are completely separate mechanisms and confusing them is a common debugging mistake.

> **Why now:** Cache tag purging was Enterprise-only on Cloudflare until April 1, 2025. It is now available on all plans at no extra cost.

Add a `Cache-Tag` response header from the origin. Cloudflare strips it before delivery but indexes content by tag. One API call purges every CDN edge entry carrying that tag, globally, in under 150ms.

```ts
// Express example — tag character assets by character ID
res.setHeader('Cache-Tag', `char-${charId}, char-thumbnails, product-assets`);

// CF API call in GitHub Actions deploy job (after origin is live):
// POST https://api.cloudflare.com/client/v4/zones/{zone}/purge_cache
// { "tags": ["deploy-2026-05-27"] }
```

**Tag taxonomy for a content product:**
- `char-{id}` — purge all cached assets for one character
- `product-thumbnails` — purge all thumbnail variants
- `deploy-{buildId}` — purge everything from this exact deploy
- `r2-media` — purge entire media CDN (nuclear, but targeted to R2 subdomain only)

**Rules:**
- Always purge **after** the new origin content is live and health-checked. A purge against an origin still serving old bytes refills the cache with stale content.
- Never use `purge_everything` zone-wide. `tags` or `prefixes` only. Zone-wide purge hits Cache Reserve too and causes a thundering herd on origin.
- Purge HTML paths on every deploy. Purge asset tags only when the content of that class changed.

---

### Staging Must Structurally Bypass Cache

A staging environment that is edge-cached is not a staging environment — it is a delayed production environment. This is a common misconfiguration: staging is excluded from the "long-cache" rule but still falls through to a "respect origin cache-control" rule that honors R2's `immutable, 1y` metadata.

**The fix:** one explicit named CF Cache Rule at position 1, no conditions:

```
Rule name: "Bypass — staging and preview environments"
Match:     http.host contains "-staging." OR http.host contains "-beta." OR http.host contains ".preview."
Action:    Bypass cache (Cache eligibility: Bypass cache)
Position:  1 (first rule, cannot be overridden by anything below it)
```

**Verify in CI health check:**
```bash
STATUS=$(curl -sI "https://media-staging.yourdomain.com/any-path" | grep -i cf-cache-status | awk '{print $2}')
if [[ "$STATUS" != "DYNAMIC" && "$STATUS" != "BYPASS" ]]; then
  echo "❌ Staging is being edge-cached (cf-cache-status: $STATUS). Deploy blocked."
  exit 1
fi
```

If this check fails, the deploy is blocked. Not documented. Not a manual step. Blocked.

---

### The Pre-deploy Checklist (for any new project)

Before shipping feature one, confirm each item:

- [ ] **Taxonomy table written** and committed to `docs/adr/`. Every asset class has a bucket, a key scheme, and a cache rule.
- [ ] **Bucket 3 delivery on staging/prod is object storage** (R2/S3 via `MEDIA_BASE`), not long-cached `public/` on the app origin. Git may hold asset files locally; environments that serve users read from the media CDN.
- [ ] **Bundler verifiably hashes Bucket 1.** Run a build, inspect `dist/` — confirm `[hash]` in JS/CSS/font filenames.
- [ ] **HTML returns `no-cache`.** Verify with `curl -sI yoursite.com | grep -i cache-control`.
- [ ] **One URL construction site per asset class.** No raw string asset paths outside the designated helper module.
- [ ] **Staging bypass rule in CF.** Verified with `curl -sI staging.yoursite.com | grep -i cf-cache-status`.
- [ ] **Stale chunk recovery.** If on Vercel: Skew Protection enabled. If self-hosted: chunk-not-found recovery handler deployed and smoke-tested.
- [ ] **Cache tags defined.** At minimum: a `deploy-{buildId}` tag on all HTML responses, so you can surgically purge HTML after any deploy.

---

### Anti-Patterns

- ❌ `public/assets/product-image.webp` with `Cache-Control: immutable`. Stable filename + immutable = human memory as the cache contract. Will fail.
- ❌ `?v=N` bumping as the primary busting strategy for product images. Works until someone forgets. Then a user is stuck on a one-year-old image.
- ❌ Constructing R2/CDN URLs inline in components (`https://media.domain.com/${charId}/${name}.webp`). Change the domain, chase 20 files.
- ❌ Pre-generating thumbnail size variants at upload time when Cloudflare Image Resizing is available. Pipeline complexity with no benefit.
- ❌ Same CDN cache rule for thumbnails (updated on new character art) and video segments (immutable after upload). Different change frequencies must have different rules.
- ❌ Relying on `purge_everything` as a deploy step. Hits Cache Reserve, causes thundering herd, nukes caches for unrelated assets.
- ❌ Staging excluded from a long-cache rule but not covered by an explicit bypass rule. Falls through to a "respect origin" rule that still caches.

---

