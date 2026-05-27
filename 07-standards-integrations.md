<!-- Split from general-engineering-playbook.md — content verbatim. See README.md for reading order. -->

## 17. Documentation Discipline

### ADRs — Architecture Decision Records
Every significant decision gets a numbered file in `docs/adr/`:
```
docs/adr/
  001-monorepo-with-pnpm.md
  002-supabase-over-self-hosted-postgres.md
  003-tanstack-query-for-server-state.md
```
Format:
- **Context** — what problem
- **Decision** — what we chose
- **Consequences** — what we gain and lose
- **Alternatives considered** — what we rejected and why

Write ADRs *when* the decision is made, not after. Six months later you will not remember why you chose option B.

### Documentation Governance — every doc has one home

> **Why now:** AI agents are biased toward creating new files. Without a positive taxonomy, a year of AI-assisted work produces dozens of stray `.md` files — investigation writeups, status snapshots, scratchpad notes, half-finished design memos — each containing a fragment of context the agent (or a human) needed at the moment. Stray docs become decoy attention sinks for future AI sessions: agents read them, find them stale or partial, and either propagate the staleness or get confused. A positive taxonomy means new docs land in the right place by default, and anything that doesn't fit is a signal that the taxonomy itself needs an extension (via ADR), not that someone should drop a fresh `NOTES.md` somewhere.

> **AI impact:** When the AI is asked to "document this," the taxonomy answers "where" automatically. Adding an architectural decision → ADR. Adding deferred cleanup → watchlist entry. Adding a new build domain → new domain doc within the build cadence. Adding a runbook → `docs/runbooks/`. The AI never has to guess. Combined with the build cadence system (`docs/build/00-genesis-and-build-cadence.md`), every new piece of documentation either lands in the master build plan as work to do, in the watchlist as work to defer, or in one of the canonical categories below.

**The canonical doc taxonomy** (every `.md` file in a repo must fit into exactly one row):

| Category | Location | Purpose | Owner / lifecycle |
|---|---|---|---|
| Engineering playbook snapshot | `docs/playbook/` | Frozen reference for Genesis & re-baseline | Copied at adoption; updated by re-baseline (`00-…` §9.2) |
| Build cadence operating layer | `docs/build/00-genesis-and-build-cadence.md` | Genesis Protocol + Build Loop spec | Replaced on re-baseline |
| Build index / router | `docs/build/README.md` | "Which doc to use" table | Updated when domain docs added/removed |
| Domain docs | `docs/build/0X-<domain>.md` (typically `01-…` to `12-…`) | Project-specific contracts (product, DB, design, caching, AI, etc.) | Created at Genesis; updated by drift writeback |
| Master build plan | `docs/build/13-master-build-plan.md` | Phased execution sequence | Loop appends per-phase status |
| Code quality watchlist | `docs/build/14-code-quality-watchlist.md` | Deferred cleanup with triggers | Loop appends entries; cleanup phase resolves them |
| Pre-shipping handoff | `docs/build/15-pre-shipping-handoff.md` | Items only a human can resolve | Loop appends; human resolves |
| Late-added domain docs | `docs/build/16-…md`, `17-…md`, etc. | Domains discovered mid-build (scope expansion) | Created via drift; numbered to preserve fixed canonical positions of `13-15` |
| Architecture decisions | `docs/adr/NNN-<slug>.md` | Why we chose X over Y | One per decision, written when decision is made |
| Off-loop task briefs | `docs/tasks/T-NNN-<slug>.md` | Ad-hoc work outside the master build plan (urgent fixes, exploratory spikes) | One per off-loop task; archived after completion |
| Operational runbooks | `docs/runbooks/<slug>.md` | Deploy, rollback, incident response, on-call procedures | One per procedure; reviewed quarterly |
| Postmortems | `docs/postmortems/YYYY-MM-DD-<incident-slug>.md` | Real incidents (one per) | Date-stamped, written within 5 days of incident |
| Cursor rules | `.cursor/rules/*.mdc` | AI guardrails, glob-attached | Created at Genesis; updated when conventions evolve |
| Multi-agent compat | `CLAUDE.md`, `AGENTS.md` if applicable | Non-Cursor agent alignment with same conventions | Mirrors `001-core-project.mdc` + loop invariants |
| Repo orientation | Repo root `README.md` | First impression: what this repo is, how to run it, where docs live | One per repo, max ~50 lines, links out for detail |
| Package/app orientation | `apps/X/README.md`, `packages/X/README.md` | One paragraph + link to relevant `docs/build/0X-…md` | One per app/package, max ~30 lines |
| Inline API documentation | TSDoc / JSDoc comments | Public function/type/class contracts | Adjacent to the code; never a parallel `.md` file |

**Banned patterns — never create these:**

- `TODO.md`, `BACKLOG.md`, `IDEAS.md` anywhere — use the watchlist (deferred cleanup with triggers) or the master build plan (new work as a phase). The watchlist has cleanup triggers; "TODO.md" doesn't.
- `STATUS.md`, `PROGRESS.md`, `WIP.md` — use the master build plan's per-phase `Current implementation status` field. The loop maintains it; status files do not.
- `NOTES.md`, `SCRATCHPAD.md`, `meeting-notes-YYYY-MM-DD.md` — meeting notes are not engineering artifacts. Decisions become ADRs; deferrals become watchlist entries; tasks become T-NNN briefs. Raw notes have no home in the repo.
- `<feature>-investigation.md` in source folders — investigations are throwaway. Their *outputs* become runbooks, ADRs, postmortems, or get incorporated into the relevant domain doc.
- Multiple competing `README.md` files in the same folder.
- `.md` files inside `src/`, `app/`, `pages/`, or any source folder — use TSDoc adjacent to the code, not parallel markdown.
- A new top-level `docs/<something>/` folder for a use case that fits an existing category — extend the existing taxonomy first; propose a new category via ADR only if the use case truly doesn't fit.
- Migration guides as standalone markdown — these belong as ADRs (decision provenance) or as phases in the master build plan (execution).
- Duplicate documentation: the same concept documented in two places. Pick one canonical location; the other becomes a one-line link.

**Routing decision table — "where does this doc go?":**

| What you want to document | Where it goes |
|---|---|
| "How to build feature X" | New phase in `13-master-build-plan.md` |
| "Why we chose Y over Z" | New ADR |
| "Defer this cleanup until trigger fires" | New watchlist entry |
| "Apple / Google / legal needs to provide value Z" | Pre-shipping handoff entry |
| "How the database schema works" | Domain doc (e.g. `05-database-schema-plan.md`) |
| "A new domain area not anticipated at Genesis" | New numbered domain doc (`16-…md`, `17-…md`, …) plus drift protocol writeback |
| "How to deploy / roll back / respond to alert X" | New runbook in `docs/runbooks/` |
| "What happened during incident on date X" | New postmortem in `docs/postmortems/` |
| "Off-loop one-shot fix or exploratory spike" | T-NNN task brief in `docs/tasks/` |
| "Quick orientation for a new contributor" | Repo root `README.md` (link out for detail) |
| "What does this package do?" | Package-level `README.md` (max ~30 lines, link to relevant domain doc) |
| "What does this function do?" | TSDoc comment on the function |
| "New contributor week-1 onboarding" | The onboarding checklist in `01-checkpoints-and-sanity.md` of the playbook is canonical; project README points to it |
| "A meeting happened" | Outcomes go to ADR / watchlist / T-NNN. Notes do not belong in the repo. |

**Enforcement — three layers catch sprawl before it compounds:**

1. **Cursor rule.** The `001-core-project.mdc` rule contains a doc-governance bullet (see playbook §16): "Before creating a new `.md` file, route it through the taxonomy in playbook §17. If it doesn't fit, propose an ADR to extend the taxonomy — do not invent a new doc category."
2. **Build loop self-review.** The per-batch rubric (`00-genesis-and-build-cadence.md` §4.3) checks for new `.md` files outside the canonical locations. Stray docs trigger a stop-and-ask before commit.
3. **CI or pre-commit hook.** A grep pass fails the build on banned filenames (`TODO.md`, `STATUS.md`, `NOTES.md`, `SCRATCHPAD.md`, etc.) and on `.md` files inside source directories. Cheap to add, prevents drift over months.

**When the taxonomy genuinely doesn't fit:**

If you have a real documentation need that no row above accommodates, do not create a new category by writing a `.md` file in a new location. Stop, write an ADR proposing the extension, and either:

1. Extend the taxonomy in this section and commit the ADR alongside the first doc of the new type, or
2. Discover during the ADR that the need actually fits an existing category, and route it there.

The taxonomy itself is documentation, and every extension should be deliberate.

### Single source of truth
| Concept | File |
|---|---|
| Route names | `packages/validation/routes.ts` |
| User roles | `packages/validation/roles.ts` |
| API error codes | `packages/validation/errors.ts` |
| Analytics event names | `packages/observability/events.ts` |
| Query keys | `packages/database/query-keys.ts` |

If a concept appears as a raw string in more than one file, it is not being managed correctly. Grep for raw strings before each PR.

### Constants & Configuration Management

No hardcoded values in application code. Every constant has an owner and a location:

| What | Where | Example |
|---|---|---|
| Design tokens (colors, spacing, radii) | `packages/design-tokens/` | `--color-brand-primary` |
| App-wide typed constants (timeouts, limits, pagination sizes) | `packages/config/constants.ts` | `SESSION_TIMEOUT_MS`, `MAX_UPLOAD_BYTES` |
| Domain string literals (routes, roles, error codes, event names) | `packages/validation/` (Zod enums or TS unions) | `UserRole`, `Route`, `ApiErrorCode` |
| Per-feature constants | `constants.ts` adjacent to the feature file | `ITEMS_PER_PAGE = 20` |
| Environment variables | Zod-validated at startup in `config/env.ts` — never accessed via `process.env` inline | `Config.DATABASE_URL` |

**Hard rules:**
- A number that controls behaviour (`timeoutMs`, `maxRetries`, `pageSize`) is never an inline literal — it lives in `constants.ts` with a name explaining its purpose.
- A string that appears in more than one file is a type union or Zod enum, not a string literal.
- `process.env.ANYTHING` is only accessed inside `config/env.ts`. Everywhere else imports from `Config`. This means a missing env var surfaces at startup, not mid-request.
- Design tokens are the single source for any visual value. Never hardcode `#3B82F6` or `16px` in a component — always use the token variable. This is what makes a theme change a one-file change.

### Code comments
- Comments explain **why**, not **what**. The code explains what.
- Bad: `// increment counter` next to `counter++`
- Good: `// must call cleanup before starting new load — carousel holds a detached DOM reference otherwise`

### Time, Timezones, i18n, Accessibility — short rules

These are small individually but each one is a frequent cross-cutting bug.

**Time and timezones:**
- **Store every timestamp as UTC** (`TIMESTAMPTZ` in Postgres). Convert to user timezone only at display.
- **Never trust client clocks** for anything billing, security, or expiry-related. `Date.now()` on the client can be anywhere.
- **Display times via `Intl.DateTimeFormat`** with an explicit timezone — never string-concatenate dates.

**i18n (internationalization):**
- If you might ship in more than one language, set up locale routing **on day 1**. Next.js: `next-intl` is the 2026 standard. Retrofitting i18n into 100 components is a multi-month project.
- All user-facing strings live in locale files keyed by ID — never inline string literals in JSX.
- Currency, dates, numbers — always `Intl.NumberFormat` / `Intl.DateTimeFormat` with locale param.

**a11y (accessibility):**
- WCAG 2.1 AA is the legal baseline in most jurisdictions (US ADA, EU EAA from 2025).
- Semantic HTML first; ARIA only when semantic HTML can't express the intent.
- Every interactive element keyboard-reachable (Tab order) and screen-reader labelled.
- Color contrast ≥ 4.5:1 for text — enforce via design tokens, check in CI with `axe-core` Playwright integration.
- Focus traps in modals; visible focus ring on all interactive elements; never `outline: none` without a replacement.

---

## 18. File-Size & Code-Quality Guardrails

Build these into the team's habits and lint config:

- **Files < 250 lines** — aim for.
- **Files 250-400** — review whether they're doing one thing.
- **Files > 400** — split unless there's a real reason not to.
- **Files > 600** — almost always a code smell.
- **Single-responsibility test**: can you summarise the file in one sentence? If no, split.
- **Don't extract shared code on the first duplication.** Wait until the third use; the right abstraction is rarely visible at use #2.
- **Don't add a subfolder just because the folder has many files.** A flat namespace with a consistent prefix (`admin-review-*`) is easier to grep than nested folders.

---

## 19. Third-Party Service Abstraction

> **Why now:** LLM provider landscape (OpenAI, Anthropic, Google Gemini, xAI) is still shifting. Payment gateways, email providers, and SMS vendors change or add pricing tiers. A codebase that directly calls `openai.chat.completions.create(...)` throughout its feature code will require changes in 10+ files if the provider changes. The adapter pattern — a provider-agnostic wrapper that the rest of the codebase depends on — costs one hour to set up and saves days when you need to switch.

**The two-tier abstraction pattern (applies to AI, payments, email, SMS, storage):**

```
packages/ai/src/
  llm-client.ts     ← raw API calls, retry, timeout, token counting, error normalisation
  ai-service.ts     ← business logic: personality, context, character selection, prompt assembly

packages/email/src/
  email-client.ts   ← raw API calls to Resend/SendGrid/SES
  email-service.ts  ← business logic: template selection, personalisation, unsubscribe checks
```

**Rules:**
- The `*-client.ts` layer knows the provider SDK. Nothing else does. Switching providers means rewriting one file.
- The `*-service.ts` layer knows your business domain. It never imports provider SDKs directly.
- The provider is injected via config, not hardcoded:
  ```ts
  // packages/config/constants.ts
  export const LLM_PROVIDER: "openai" | "anthropic" | "gemini" = Config.LLM_PROVIDER;
  ```
- All external API calls have: **timeout** (fail fast — never hang a user request waiting for an LLM), **retry with backoff + jitter** (see Resilience Patterns below for the correct implementation), **fallback provider** (if configured), **cost/token tracking per call** (you will get surprised by LLM costs otherwise).
- Error types from the external service are **normalised** at the client layer into your internal error types before they reach the service layer. The service layer never catches `OpenAIError` — it catches `LlmRateLimitError`, `LlmTimeoutError`, etc.

> **AI impact:** The AI can add a new AI feature without knowing which LLM provider you're using — it calls `aiService.generateResponse(...)`. If you ever switch providers, you update one file. Without this pattern, "switch from OpenAI to Gemini" is a refactor; with it, it's a config change.

### Resilience Patterns — wrapping external calls correctly

Every call to an external service is a failure waiting to happen. The order of defence layers matters:

```
Bulkhead → Timeout → Circuit Breaker → Retry (with backoff + jitter)
```

Apply from outermost to innermost:

**1. Bulkhead** — isolate resources by dependency. Give your Stripe calls a separate concurrency pool from your LLM calls. If an AI provider is slow, it cannot exhaust the thread/connection pool and block payment processing.

**2. Timeout** — every external call has a hard deadline (e.g. 5s for REST APIs, 30s for LLM completions). Never `await` an external call without a timeout — a hung provider hangs your request indefinitely, consuming server resources until it times out at the platform level (30s on Vercel), often silently.

**3. Circuit Breaker** — after N consecutive failures (or X% error rate over a window), "open" the circuit: stop calling the failing service and return a predefined fallback immediately. Re-probe on a timer (half-open state). Prevents cascading failures where one slow dependency takes the whole app down.

**4. Retry with exponential backoff + jitter** — only for transient errors (timeouts, HTTP 429, HTTP 503). Never retry 4xx client errors (400, 401, 403, 404) — the request won't succeed on retry. Jitter (adding random delay to the backoff) prevents retry thundering herds after a provider outage.

```ts
// Exponential backoff + jitter pattern:
async function callWithRetry<T>(fn: () => Promise<T>, maxAttempts = 3): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts || !isTransientError(err)) throw err;
      const baseDelay = Math.min(100 * 2 ** attempt, 10_000); // cap at 10s
      const jitter = Math.random() * baseDelay * 0.25;
      await new Promise(res => setTimeout(res, baseDelay + jitter));
    }
  }
  throw new Error('unreachable');
}

function isTransientError(err: unknown): boolean {
  // Retry on network errors, timeouts, 429, 503 — not on 4xx client errors
  if (err instanceof TimeoutError) return true;
  if (err instanceof HttpError) return [429, 503, 502, 504].includes(err.status);
  return false;
}
```

**Implementation shortcut:** [`cockatiel`](https://github.com/connor4312/cockatiel) is a TypeScript-native library providing Bulkhead, Circuit Breaker, Timeout, and Retry policies in one package. Prefer it over writing these patterns from scratch — the edge cases (half-open state, concurrent retry limits) are non-trivial.

### Email Deliverability — DNS + reputation

> **Why this matters:** transactional email ending up in spam is one of the most user-visible failure modes (password resets, magic links, receipts). It is almost always a DNS/reputation problem, not a code problem. Set this up before launch.

The four DNS records every product needs:

1. **SPF** (TXT record on sending domain) — lists which mail servers are authorized to send for the domain. Resend/SendGrid/SES each provide the exact value.
2. **DKIM** (CNAME records) — cryptographically signs outgoing mail. Provider generates the keys; you publish the CNAME.
3. **DMARC** (TXT at `_dmarc.yourdomain.com`) — tells receiving servers what to do with mail failing SPF/DKIM, plus where to send aggregate reports.
4. **MX** records only if you also receive mail at that domain.

**DMARC rollout — staged, never jump to `p=reject`:**
```
_dmarc IN TXT "v=DMARC1; p=none; rua=mailto:dmarc@yourdomain.com"        # monitor 2 weeks
_dmarc IN TXT "v=DMARC1; p=quarantine; pct=25; rua=mailto:..."           # gradual enforce
_dmarc IN TXT "v=DMARC1; p=reject; rua=mailto:..."                       # full enforce
```

**Separate transactional and marketing domains.** Send password resets from `mail.yourdomain.com`, send newsletters from `marketing.yourdomain.com`. Reputation is per-domain; a marketing spam complaint can poison transactional delivery if they share a domain.

**Monitor:** Google Postmaster Tools (free) gives you domain reputation, spam rate, and authentication pass rate for Gmail recipients. Set up day 1 — Gmail is ~30% of consumer email.

---

## 20. Payment & Subscription Patterns

These patterns apply whether you use Stripe, Paddle, Lemon Squeezy, or any other gateway.

**Payment state machine — never just a boolean:**
```ts
type PaymentState = "idle" | "pending" | "processing" | "completed" | "failed" | "refunded";
```
Every transition is logged with timestamp and reason. Never skip states. A `completed → refunded` transition must create an audit row, not just update the `status` field.

**Idempotency keys on every payment API call.** Network retries must never double-charge. The key must be **stable across retries** — generate it once, persist it with the order record, reuse it if the request needs to retry:
```ts
// Generate once when the order/checkout session is created and store in DB:
const idempotencyKey = crypto.randomUUID(); // stable; NOT Date.now()
await db.orders.create({ ...orderData, idempotencyKey });

// On every attempt (including retries), use the stored key:
const { idempotencyKey } = await db.orders.findById(orderId);
await stripe.paymentIntents.create({ amount, currency }, { idempotencyKey });
```
> ❌ `\`payment-${userId}-${Date.now()}\`` — **wrong**. `Date.now()` changes every millisecond so every network retry becomes a new key and the idempotency guard is never triggered.

**HMAC-validate every payment webhook before acting on it.** The webhook endpoint is public — anyone can POST to it. Check the provider's signature header before touching your database.

**Webhook replay protection — HMAC alone is not enough.** A captured signed webhook can be re-submitted unless you also reject stale or repeated deliveries:
- **Timestamp window** — reject if the webhook timestamp is more than 5 minutes off from server time. Stripe and similar providers include a timestamp in the signed payload.
- **Idempotency on the event ID** — every webhook event has a stable ID (Stripe: `evt_...`); record processed IDs in a `processed_webhooks` table and skip duplicates. Pair with the §11 at-least-once / idempotent-consumer pattern.

**Subscription access gate — one field, one check:**
```ts
// The ONLY field that gates access:
const hasAccess = user.premium_until !== null && user.premium_until > new Date();
```
- All other subscription fields (`tier`, `status`, `plan_id`) are informational for display and billing logic only.
- This check lives in one middleware function and one shared utility. It is never recalculated inline.
- Server-side on every protected request. Frontend display is derived from the same function — not reimplemented.
- Never trust the client to report whether it has access.

**Free trials — server-side always:**
- Trial state is backed by a database row, not a cookie or localStorage.
- Trial limits (message count, feature gates) enforced server-side. Client-side display is informational.
- IP ledger (stored in DB or Redis with appropriate TTL) prevents trivial abuse via account cycling.

---

