<!-- Split from general-engineering-playbook.md — content verbatim. See README.md for reading order. -->

## 24. The AI + Testing Feedback Loop (how they work together)

This section explains how the architecture above creates a closed loop between AI development and automated verification.

```
AI writes code
     ↓
TypeScript (2s)  ←── inline in Cursor, fastest feedback
     ↓ pass
ESLint (flat config, 5s)  ←── catches import cycles, boundary breaks
     ↓ pass
Unit tests / Vitest (10-30s)  ←── pure functions, state machines, schemas
     ↓ pass
Integration tests (1-3min)  ←── real DB, auth flows, API routes
     ↓ pass
E2E / Playwright (5-15min)  ←── critical journeys only, against production build (staging or local `next build`)
     ↓ pass
PR approved → merge → deploy
```

### Why the order matters for AI

The AI gets feedback at each gate. If TypeScript fails, the AI corrects immediately. If unit tests fail, the AI corrects in the same session. If integration tests fail, the AI debugs with the test output. The AI never merges broken code — the CI pipeline acts as an always-on senior reviewer.

### What to write tests for when using AI

**High priority — the AI is prone to getting these wrong:**
- Auth rejection (endpoint works but is publicly accessible — the AI forgets the guard)
- Schema validation (the AI's Zod schema allows values the business rules shouldn't)
- Edge cases from AI extraction / AI-generated data (null fields, wrong types, missing required data)
- State machine transitions (the AI writes correct happy-path code but misses impossible-state prevention)

**Lower priority — the AI is usually correct here:**
- Simple CRUD operations with typed DB clients (the types prevent most mistakes)
- Pure UI rendering with no business logic

### Your test suite is training data

This is the most important insight from 2025-2026 AI coding research: **teams with comprehensive test suites extract dramatically more value from AI coding agents**, because tests are the reward signal — the thing the agent optimises toward. When an AI agent runs `pnpm test` and sees green, it knows it succeeded. When it sees red, it knows what to fix. No tests = the agent is flying blind and so are you.

This dynamic compounds: stronger test coverage → better AI output → faster development → more time to write tests. The inverse is also true.

### Layered verification — one layer is not enough

A single test suite as the sole verification layer is insufficient when an AI agent is actively optimising against it. Agents can pass unit tests while violating architectural rules. The defence is **multiple independent layers** that the agent cannot satisfy simultaneously through a single shortcut:

```
1. TypeScript (2s)          — type correctness
2. ESLint flat config (5s)  — import cycles, boundary violations
3. Unit tests (30s)         — function behaviour
4. Integration tests (2min) — real DB + auth, cross-layer contracts
5. E2E tests (10min)        — user-visible journeys
6. Architectural constraints — file size limits, no-restricted-imports, server-only boundaries
```

An agent that "passes the tests" by deleting assertions or adding bypasses will fail other layers. The more independent the layers, the harder gaming becomes.

### The "write test first" rule for AI

When using AI to implement a non-trivial feature, give it the test first:
1. Write a failing test that describes the expected behaviour in precise terms.
2. Ask the AI to make the test pass.
3. The AI now has a concrete, machine-verifiable success criteria — not a vague description.

This dramatically improves AI output quality compared to "implement feature X." The test is the spec.

### Cursor rules that reinforce the testing loop

The most effective testing-related rules to put in `.cursor/rules/007-testing-quality.mdc`:
```
- Write unit tests for every new pure function before marking the task complete.
- Every new API route must have an integration test with: happy path, auth rejection, and validation rejection.
- Never mock the DB in integration tests — use Testcontainers or a test Supabase project.
- Test files live next to the file they test (*.test.ts adjacent to *.ts).
- Never use `expect(true).toBe(true)` as a placeholder — write a real assertion or remove the test.
```

---

