<!-- Split from general-engineering-playbook.md — content verbatim. See README.md for reading order. -->

## 9. Async, Concurrency, and Lifecycles

> **Why now:** XState v5 (2023) dramatically simplified state machines — you no longer need a PhD to use them. The `useEffect` cleanup / AbortController pattern is now the React 18+ recommended pattern for async operations (the React docs were updated in 2023 to explicitly recommend this after years of `isMounted` hacks). React 18's Strict Mode double-invokes effects in development specifically to surface missing cleanup — if your code breaks in Strict Mode, it has a lifecycle bug. These aren't new ideas but the tooling now makes them the path of least resistance.

> **AI impact:** The AI is very good at writing happy-path async code and very bad at writing cleanup code — it tends to skip `finally` blocks and doesn't think about what happens if the component unmounts mid-fetch. Explicit patterns (state machines, AbortController-in-cleanup) give the AI a template to follow. When the AI sees existing code using `AbortController`, it will replicate the pattern for new async operations in the same file.

- **State machines for any flow with > 2 states.** Use XState v5 or a typed enum machine. Never `isLoading + isCancelled + isDestroyed` boolean combos — they create impossible states.
  ```ts
  type NavState = "idle" | "loading" | "cancelled" | "error";
  ```
- **Request deduplication**: two concurrent callers for the same resource → the second awaits the first. TanStack Query handles this for data; for imperative flows use a `Promise` registry keyed by resource id.
- **AbortController everywhere**: every fetch that can be superseded gets a signal. Controller created immediately before the fetch, stored on the state machine, aborted on state transition. Cleanup in `finally`, never only in the cancel path.
- **Component-mount safety**: every `await`-after-DOM-touch must guard with an AbortController in the effect cleanup. Never use a bare `isMounted` boolean — it leaks across rerenders.
- **Granular lifecycles**: every component that owns a resource (timer, subscription, DOM ref, IntersectionObserver) cleans it up explicitly in the unmount/cleanup function. The cleanup runs *before* a new resource is allocated on rerender. The ordering rule is: `cleanup old → allocate new`, never `allocate new → cleanup old`. Violating the order means two resources exist simultaneously, and the "old" cleanup may cancel the "new" resource if they share IDs or references.
  ```ts
  useEffect(() => {
    const controller = new AbortController();
    startLoad(controller.signal);
    return () => controller.abort(); // cleanup runs before next effect
  }, [dependency]);
  ```

---

## 10. Authentication & Authorization

> **Why now:** Three credible options in 2026, each with a clear use case:
> - **Supabase Auth** — best default when your backend is already Supabase. Zero extra infrastructure, RLS integrates directly with auth.uid(). Opaque sessions, no JWT footguns.
> - **Clerk** — best for apps that need pre-built auth UI (sign-in pages, user management, org switching) with minimal implementation work. Paid but generous free tier; the most polished out-of-the-box experience.
> - **Auth.js (NextAuth) v5** — stable, open-source, deep Next.js App Router integration (runs in `middleware.ts` for v15; compatible with `proxy.ts` in v16), largest provider ecosystem (40+ OAuth providers). Right choice for teams that want full control without Supabase or Clerk pricing.
>
> JWTs for sessions (2018-2022 trend) are now understood to have fundamental pitfalls — they can't be invalidated without a denylist, are frequently oversized for cookies, and are often misimplemented. All three options above use opaque sessions by default. Never implement JWT sessions yourself.

> **AI impact:** Single-middleware auth means the AI always knows where to enforce a role check — it's never scattered through route handlers. When the AI adds a new admin endpoint, the Cursor rule says "call `requireAdminAccess` at the top of the server action" — one line, done. The AI cannot "forget" to add auth because the integration test for every new route checks the auth rejection case.

- Use a battle-tested provider: **Supabase Auth, Auth.js, or Clerk**. Never roll your own session token generation or password hashing.
- **Passkeys (WebAuthn / FIDO2)** are the 2026 standard for new user-facing auth — phishing-resistant, no shared secrets, supported natively on all major platforms and browsers. All three providers above support passkeys. For new projects, prefer passkeys over passwords as the primary credential.
- **RBAC**: roles as a TS union (`type UserRole = "guest" | "user" | "admin"`).
- **One typed middleware** `requireRole(role: UserRole)`. Route handlers never read `req.user.role` directly.
- **Frontend guards mirror backend guards.** Both derive from the same shared role union. A mismatch is a bug class, not an edge case.
- **Auth-guard failure must never silently fail.** If the guard can't redirect, it must log a warning, emit an observability event, and return the *safe default* (show login, not grant access).
- **OAuth callbacks** go through a single server-side route that validates state, exchanges the code, sets the session, and redirects. Never handle OAuth tokens client-side.

---

## 11. Real-time & Background Work

> **Why now:** HTTP request handlers in serverless environments (Vercel) timeout at ~30 seconds. Anything longer — email, image processing, AI generation, exports — must move to a background worker. The tooling landscape shifted in 2024-2025: **Inngest** became the default recommendation for Vercel-hosted Next.js apps because it requires no Redis, no persistent worker process, and integrates directly with serverless functions. BullMQ remains the right choice for high-throughput workloads on persistent infrastructure.

> **AI impact:** The AI's instinct is to do work inline in the API handler. The rule "never process background work in a request handler" combined with a typed job interface means the AI knows the exact pattern: define a `JobPayload` type, enqueue it, handle it in the worker. Zod validation at enqueue time means the AI can't enqueue malformed data.

Use the right tool per pattern — don't build one mechanism for everything.

**Background jobs — choose by deployment:**

| Scenario | Tool |
|---|---|
| Vercel / serverless (most Next.js apps) | **Inngest** — no Redis, no worker process, step-level retries, native Vercel integration. 50K free runs/month. Start here. |
| Long-running AI jobs (> 30 seconds) | **Trigger.dev v3** — durable execution, survives deploys, Bun-based workers, hours-long jobs. Best for AI pipelines. Open-source, self-hostable. |
| High-throughput (100K+ jobs/day), existing Redis | **BullMQ** — battle-tested, fine-grained concurrency, cost-effective at scale. Only worth the Redis infrastructure overhead at volume. |

**Hard rules regardless of tool:**
- Never process background work synchronously in an HTTP request handler.
- Never use `setTimeout` as a job queue.
- Every job type is a TypeScript interface. Job data is validated with Zod at enqueue time.

### Idempotency — first-class, not just for payments

Idempotency keys are covered in §20 for payment API calls, but the principle applies to any at-least-once delivery system.

**Why at-least-once matters:** Background job queues, webhook delivery systems, and message queues all guarantee at-least-once delivery. "At least once" means: if a worker crashes mid-execution, the job restarts. If a webhook HTTP call times out before acknowledgement, it retries. Your code must produce the same result whether it runs once or ten times.

The formula: **at-least-once delivery + idempotent consumer = effectively-once behaviour.**

```ts
// Idempotent job handler — check-and-act pattern:
export async function sendWelcomeEmail(payload: WelcomeEmailPayload) {
  // Check if we've already processed this job
  const alreadySent = await db.emailLog.findUnique({ where: { jobId: payload.jobId } });
  if (alreadySent) return; // deduplicated — do nothing

  await emailService.send({ to: payload.userEmail, template: 'welcome' });

  // Record completion — ideally in the same transaction as the side effect
  await db.emailLog.create({ data: { jobId: payload.jobId, sentAt: new Date() } });
}
```

**Apply this pattern to:**
- Background jobs that send emails, push notifications, or SMS.
- Webhook handlers from third-party providers (payment events, email bounces, etc.).
- Any write that could be triggered more than once on retry.

Where the side effect and the idempotency record can't be in the same transaction, make the side effect itself idempotent (e.g. upsert instead of insert, Stripe's own idempotency key).

**Real-time — choose by pattern:**

| Pattern | Tool |
|---|---|
| Multi-user fan-out (chat, presence) | Supabase Realtime (if already on Supabase), Pusher, or Ably |
| 1-to-1 streaming (AI tokens) | **Server-Sent Events (SSE)** via `Response` + `ReadableStream` — simpler than WebSocket for one-directional streaming. Next.js supports this natively. |
| Bidirectional real-time (voice, multiplayer) | WebSocket on your own server. **Auth on the HTTP upgrade**, never after. |

---

