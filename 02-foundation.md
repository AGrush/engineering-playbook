<!-- Split from general-engineering-playbook.md — content verbatim. See README.md for reading order. -->

## 1. Core Philosophy

- **DRY, KISS, YAGNI** — in that priority. Repeating yourself is cheaper than the wrong abstraction. Build the thing the product needs now; encode every architectural decision *now*, not later.
- **Fail loud, never silent** — silent failures are invisible bugs in production. Every catch block either re-throws or emits an observability event.
- **Single source of truth** for every domain concept (routes, roles, error codes, event names, query keys). If a concept appears as a raw string in more than one file, it is not being managed correctly.
- **Trust boundaries are explicit.** Data crossing a boundary is untrusted until validated. There are exactly four boundaries — **HTTP edge** (network → server), **DB edge** (server → DB), **AI edge** (user/system → LLM, LLM → tools), **third-party edge** (your code → external API). At every boundary: parse with Zod, check auth, sanitize output. Enumerate them in `docs/adr/NNN-trust-boundaries.md`. Naming the boundaries makes the AI defend them; leaving them implicit means the AI will route around them on the first opportunity.
- **AI-first development** — the codebase is written so an AI agent can extend it safely without breaking layering. That means tight boundaries, strict types, explicit contracts, and machine-readable rules.

> **Why now:** AI coding assistants (Cursor, Copilot, Claude) became first-class team members in 2024-2025. The practices that make code readable to humans — single responsibility, explicit types, predictable file names — turn out to be exactly the practices that make AI agents accurate and safe. A codebase that was written for AI-assisted development is also better for humans. The two goals are now the same goal.

> **AI impact (overall):** Every principle in this document either reduces the surface area an AI agent has to understand, makes the contract between modules machine-verifiable, or makes a wrong change immediately visible in CI. If the AI breaks a rule, a test fails or the typecheck fails — you find out in 30 seconds, not in production.

---

## 2. Repo Topology

> **Why now:** pnpm + Turborepo is the 2024-2026 standard for JS/TS monorepos. pnpm's symlink-based `node_modules` is 3-5× faster than npm, prevents phantom dependencies (where a package uses something it didn't declare), and Turborepo's task graph caches builds/tests at the package level so you never re-run what hasn't changed. The deprecated alternatives — Lerna (archived), Yarn Workspaces alone (no task graph) — are either dead or incomplete. **Nx** is a credible alternative at larger scale: it has more powerful project graph analysis and integrated plugins, but requires more initial config. Turborepo is the right default for most teams; Nx is worth evaluating when you have 50+ packages or need fine-grained affected analysis. Turborepo requires almost zero config to start.

> **AI impact:** When the AI adds a file, the monorepo structure tells it exactly which package it belongs in. "This is a database query" → `packages/database/`. "This is a Zod schema" → `packages/validation/`. The AI never has to decide where types live because the topology decides for it. Circular dep detection in CI means the AI can't accidentally create an import loop that would compile but fail at runtime.

### Before you start: What kind of system are you building?

The most impactful architectural decision happens before you write a single line of code, and most teams make it implicitly rather than explicitly.

**The 2026 default: start with a Modular Monolith.**

A modular monolith is a single deployable unit with strict internal module boundaries — the structure this entire playbook is designed around. One deployment, one CI pipeline, one database connection pool. But internally, code is organised into isolated domain packages that own their responsibilities, expose explicit APIs, and never reach across each other's boundaries.

This is not the "big ball of mud" monolith. It is the intentional middle ground.

| Architecture | Deploy units | When to use |
|---|---|---|
| **Modular monolith** (default) | 1 | Team < 50 engineers; boundaries still forming; deployment cost matters |
| **Microservices** | N | Proven scaling bottleneck; stable domain boundaries; dedicated platform team |
| **Big ball of mud** | 1 | Never intentionally — absence of boundaries, not an architecture |

**Extract a service only when you hit a concrete gate:**
1. One module needs 10× the resources of the rest — independent scaling is required.
2. Multiple autonomous teams are blocked by the same deploy pipeline.
3. Compliance requires physical data isolation (HIPAA, PCI, FedRAMP).
4. A module needs a fundamentally different runtime (e.g. ML inference in Python alongside a Node.js API).

**The most expensive mistake: distributed monolith.** Services split before domain boundaries are stable. You get all the operational overhead of microservices — network hops, distributed tracing, deployment coordination, partial failures — with none of the scaling or autonomy benefits. Most teams that rushed to microservices in 2018-2021 are re-consolidating today.

**If you do extract a service:** use the **Strangler Fig pattern** — route traffic through the new service while keeping the old code path live, verify at scale, then retire the old path. Never do a big-bang rewrite.

The monorepo structure below (`apps/` + `packages/` + `tooling/`) maps directly to a modular monolith. The packages are your module boundaries. The import rules (apps import packages, never the reverse, no cycles) are your boundary enforcement. You are already practising modular monolith architecture by following this playbook.

Use a **monorepo with workspace packages** (pnpm + Turborepo, or Nx for large enterprises). Even a "single product" is multiple deployables: web, mobile, admin, ingestion, background workers.

The 2026 standard is a **three-layer structure**:

```
apps/           ← THIN: routing, composition, env vars only. ~20% of code.
  web/          - Next.js (landing, marketing, admin)
  mobile/       - React Native / Expo
  workers/      - Background jobs, scheduled tasks

packages/       ← THICK: 80% of the codebase lives here, isolated by domain.
  validation/   - Zod schemas — shared types & runtime guards
  database/     - DB queries, mutations, generated types
  ui/           - shadcn/ui primitives copied here as owned source (not an npm dep). Adopt on Day 1. Feature code imports from here only — never raw HTML + Tailwind utilities in `apps/`.
  ai/           - LLM/embedding/extraction clients
  observability/- Logging, error capture, analytics wrapper
  design-tokens/- Colors, spacing, typography CSS custom properties (one source for web + mobile). Token values only — no component logic here.
  assets/       - Asset URL construction helpers (see §8A). One function per asset class.

tooling/        ← INFRASTRUCTURE: shared configs extended by everything else.
  typescript/   - Base tsconfig.json (extended by all apps and packages)
  eslint/       - Shared ESLint flat config (extended or imported)
```

**"Keep apps thin" is the key discipline.** If a file in `apps/web/` could be moved to `packages/` without changing its logic, it belongs in `packages/`. App directories handle routing and page composition; everything with business logic or shared utility lives in packages. This is what makes features reusable across web, mobile, and worker targets.

**TypeScript project references for large monorepos:**
Each package should have its own `tsconfig.json` with:
```json
{ "compilerOptions": { "composite": true, "declaration": true, "incremental": true } }
```
This lets `tsc` check only changed packages — not the entire repo on every run. For CI, separate the type-check step (`tsc --noEmit`) from the build step (esbuild/swc transform) so they run in parallel and don't block each other.

**Rules:**
- Apps may import from packages; packages **never** import from apps.
- Packages must not import from each other in a way that creates a cycle. Enforce with `eslint-plugin-import-x/no-cycle` in CI (ESLint 9+ compatible).
- No catch-all `shared/`, `utils/`, or `common/` packages. Every package has a single responsibility.
- Configs (TypeScript, ESLint) live in `/tooling` and are extended — never duplicated across packages.
- **Tailwind CSS v4:** no `tailwind.config.js` — add `@import "tailwindcss"` in the root CSS file per app. CSS custom property tokens live in `packages/design-tokens/`; Tailwind reads them via `@theme` or CSS variables. No shared Tailwind config file is needed or wanted.
- **Icons:** `lucide-react` only. One icon library per project. Never mix sets.
- **Fonts:** `next/font` in the root layout. No `<link>` tags or CSS `@import` from Google Fonts in production.
- **Animation:** `motion/react` (Motion v12, formerly Framer Motion). Import as `import { motion } from "motion/react"` — not `"framer-motion"`.
- **AI impact:** `packages/ui/` + `packages/design-tokens/` give the agent a single, bounded styling surface. An AI writing a new component knows exactly where to look and what to import — it can't introduce a parallel styling approach without violating the cursor rule.`

---

## 3. Type System (the Contract)

> **Why now:** TypeScript strict mode with `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` became practical defaults in 2023-2024 as tooling matured. These two flags catch the most common runtime crashes (`arr[0]` returning `undefined`, optional props being treated as present) that plain strict mode misses. Zod v4 (stable July 2025) is the current version — 14× faster parsing than v3, smaller bundle via the `zod/mini` subpath, and a cleaner error API. The schema-as-type pattern (`z.infer<typeof schema>`) is unchanged from v3. If you're starting a new project, install `zod@^4.0.0`. If upgrading from v3, consult the migration guide — breaking changes are mostly in the error customisation API.

> **AI impact:** When the AI writes or modifies a function, the TypeScript compiler verifies its output immediately. The AI doesn't need to "know" what a `ReviewItem` looks like — it imports the Zod schema and the type is inferred. If the AI returns the wrong shape, `tsc --noEmit` fails in 2 seconds. Explicit union result types (`{ status: "success"; data } | { status: "error"; message }`) mean the AI always knows what to return and what the caller must handle — no guessing.

- **TypeScript strict mode everywhere.** `tsconfig.base.json` extends to all apps/packages:
  ```json
  {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitAny": true
  }
  ```
- **Domain types live in `packages/validation`** as Zod schemas. Types are inferred via `z.infer<typeof schema>`. The schema *is* the type. Never duplicate.
- **No `as SomeType` on raw HTTP/API responses.** Always parse with Zod or narrow with an explicit runtime check first.
- **String-literal domains are unions or enums, never raw strings:**
  ```ts
  type UserRole = "guest" | "user" | "editor" | "admin";
  type ModalName = "auth" | "payment" | "age_gate";
  ```

---

## 4. State Management — Two Concerns, Two Tools

> **Why now:** TanStack Query v5 (2023) and Zustand v5 (October 2024) are the settled consensus for React client state in 2026. Zustand v5 dropped deprecated APIs and ES5 support; it now uses the native `useSyncExternalStore` (smaller bundle, React 18+ required). Redux has been declining since 2022 — it creates more problems than it solves for most apps. **React 19 (stable December 2024) is the important context here** — `useActionState`, `useOptimistic`, and direct Server Action integration change how mutations and form state are handled, reducing the need for TanStack Query in many mutation flows. **Next.js App Router note:** Server Components handle data fetching directly — `async/await` at render time. TanStack Query's role is narrower than in pre-2024 React: real-time data, polling, and client-triggered queries in Client Components only.

> **AI impact:** Clear separation means the AI knows exactly where state belongs. "Did this come from a server?" — yes → Server Component fetch or `useQuery` in a Client Component. "Is this UI state?" — Zustand slice. "Is this a Server Action form?" — `useActionState`. The AI follows these three rules and almost never puts state in the wrong place.

| Concern | Tool | What it owns |
|---|---|---|
| **Server state** (anything from an API) | TanStack Query | Dedup, stale-while-revalidate, background refetch, optimistic updates |
| **Client state** (UI, modals, prefs) | Zustand (slices) | Modal open/close, theme, local preferences |

**The React 19 mutation pattern (new standard — replaces manual `useState` + loading):**

> **AI Security Rule (Safe Actions):** Never write a raw `async function` for a Server Action. Raw functions do not enforce authentication, making them the #1 security vulnerability when an AI writes them. Always use a safe action client (e.g., `next-safe-action` or a custom wrapper) that guarantees `auth.userId` is present and validates input via Zod *before* the handler runs.

```ts
// Server Action using a Safe Action client
'use server';
export const updatePost = authActionClient
  .schema(postSchema)
  .action(async ({ parsedInput, ctx: { userId } }) => {
    // 1. Input is already Zod-validated (parsedInput)
    // 2. User is guaranteed authenticated by the wrapper (userId)
    await db.posts.update({ ...parsedInput, updatedBy: userId });
    revalidateTag('posts');
    return { success: true };
  });

// Client Component — useActionState replaces useState + loading + error
// (Syntax adapts slightly depending on safe-action library choice, but semantics hold)
const [state, formAction, isPending] = useActionState(updatePost, {});
// useOptimistic — instant UI before server confirms, auto-reverts on failure
const [optimisticPosts, addOptimistic] = useOptimistic(posts, (state, newPost) => [...state, newPost]);
```

**Decision tree for state:**
- Data from server, read in Server Component → direct `async/await`, no hook needed
- Data from server, needed in Client Component → `useQuery` (TanStack Query)
- Mutation via form → `useActionState` + Server Action
- Mutation needing instant feedback → `useOptimistic` + Server Action
- UI state (modals, theme, preferences) → Zustand slice
- Never put server state in Zustand. Never put UI state in TanStack Query.

Define query keys as typed constants in `packages/database/query-keys.ts`. Every `useQuery` imports a key. Never inline.

**React Compiler (stable opt-in, Next.js 16):**
React Compiler automatically memoizes components and hooks — it replaces the manual `memo()`, `useMemo`, and `useCallback` boilerplate that teams add defensively. Enable in Next.js 16 with one line:
```ts
// next.config.ts
export default { reactCompiler: true } satisfies NextConfig;
```
The compiler analyses your code statically and adds memoization only where it's provably safe. **Rule: do not add `useMemo` / `useCallback` defensively before you've measured a render perf problem.** With React Compiler enabled, most manual memoization becomes redundant. Without it, the same rule applies — premature memoization adds noise without benefit.

---

## 5. Module Boundaries & Dependency Direction

> **Why now:** This is not a new idea — it is the Application of Clean Architecture / Hexagonal Architecture principles, which date back decades. What's new is that AI agents make boundary violations far more likely: the AI will take the shortest path to make the code compile, and the shortest path is often a direct import across a layer boundary. Without enforced rules, a fast AI development pace creates a big ball of mud faster than a slow human team. The `import-x/no-cycle` ESLint rule and the directory structure enforce boundaries that the AI cannot accidentally cross.

> **AI impact:** The AI's search space for "where does this logic go?" is dramatically reduced by layer rules. When told to add an email-sending feature, it knows: the trigger is in a service, the job is enqueued in a repository call, the email template is in the workers package. It doesn't need to guess. It also can't accidentally import the Next.js router from inside a shared package, because the package doesn't depend on Next.js.

Strictly layered, lower never imports from higher:

```
pages/routes  →  features  →  services  →  repositories  →  types
```

If a lower layer needs to trigger a higher-layer behaviour (e.g. an auth guard needs to navigate), it calls an **injected function**, not a direct import. This is dependency inversion — depend on abstractions.

**Circular dependencies are banned.** CI fails on any cycle. If two modules need each other, one of three things is true:
1. They should be merged.
2. The shared logic should be extracted to a third module they both import.
3. One should call the other via a registered interface.

**Every file has exactly one owner module.** If you can't name the feature that owns it, the file is in the wrong place.

**Use the `server-only` package to prevent accidental client bundling of server code:**
```ts
// At the top of any file containing secrets, DB queries, or server-only logic:
import 'server-only';

// If a Client Component ever imports this file, the build fails immediately
// with a clear error — rather than silently shipping secrets to the browser.
```
Add `server-only` to: every file in `packages/database/`, any file that accesses `process.env` secrets, service/repository files that must never reach the browser. **Do not add it to Server Action files** — Server Actions (`"use server"`) must remain importable by Client Components so they can be called. `server-only` is for files that should never be imported anywhere in the client bundle, not for callables.

---

## 6. Backend Layering

> **Why now:** Three options in 2026 depending on where your server runs:
> - **Fastify** — TypeScript-first, schema-based validation built in, best throughput for a traditional long-running Node.js server. Production-proven at scale.
> - **Express 5** (stable 2024) — native async error handling, largest ecosystem, best hiring pool. Slightly more boilerplate than Fastify.
> - **Hono** (2022, rapidly adopted 2024-2025) — ultralight, runs on Node.js, Cloudflare Workers, Bun, Deno, and Vercel Edge Functions. TypeScript-first, ~14KB bundle. The right choice if you're running on edge/serverless runtimes or need a lightweight API layer alongside a Next.js app.
>
> For most new products in 2026, Supabase or similar BaaS handles auth, storage, and database — the question is "how much custom server do I need?" The layering rules apply regardless.

> **AI impact:** When the AI adds a new endpoint, the layer rules tell it exactly which file to create in which folder and what it's allowed to import. It cannot "go rogue" and write a 400-line route file that does DB queries and sends emails — the structure prevents it. Smaller, single-responsibility files are also much easier for the AI to generate correctly because the context window stays focused.

If you write a custom backend (Fastify / Express 5 / Hono):

```
routes/         - HTTP boundary only. No business logic.
controllers/    - Orchestration. No SQL. Calls services.
services/       - Business logic. No req/res, no HTTP knowledge.
repositories/   - DB access only. No business logic.
middleware/     - Cross-cutting (auth, rate limit, logging).
```

- A route never calls a repository directly.
- A service never imports from `express`.
- Layering violations are a **PR rejection reason**, not a suggestion.

**Backend-as-a-service alternative** (Supabase, Firebase): the layering still applies, but `services/` and `repositories/` move into a `packages/database/` package wrapping the BaaS client. Apps call typed query/mutation functions, never the raw BaaS client directly. This keeps RLS, audit logs, and validation in one place.

**Env config**: validate at boot with Zod. If a required env var is missing, **the process exits at startup** with a clear error. Never silently fall back to an insecure default.

### API Style — choose at the boundary, not the implementation

The layering above is protocol-agnostic. The question of *what API surface to expose* is a separate, explicit decision.

| Protocol | Best for | 2026 status |
|---|---|---|
| **REST** | Public APIs, partner integrations, universal compatibility | Dominant for external APIs — most hirable, widest tooling |
| **tRPC** | Full-stack TypeScript monorepos, internal calls only | Growing fast — zero schema boilerplate, end-to-end type inference with no codegen |
| **GraphQL** | BFF layer when web, mobile, and third parties need different data shapes from the same backend | Right tool for heterogeneous clients; more setup cost |
| **gRPC** | Service-to-service internal calls at high throughput | Microservices; rarely needed in a modular monolith |

**Decision tree:**
- TypeScript monorepo, you control both frontend and backend → **tRPC** (no schema files, types flow end-to-end, mutations are just typed function calls)
- Public API, webhooks, or partner integrations → **REST** with an OpenAPI spec
- Multiple client types needing meaningfully different data shapes → **GraphQL** BFF layer
- **The winning default for most products: tRPC internally + REST for any external-facing API.** This is not inconsistent — it is using the right tool per context.

The anti-pattern: one protocol for everything regardless of use case. Picking a single API style "for consistency" forces you to use a protocol that doesn't fit every interaction.

---

