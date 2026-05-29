# Testability Playbook

Guidelines for writing testable code during development and auditing existing code before
writing tests. Applies to all features.

For overall testing strategy, suite structure, the mocking policy, and the recorded-AI-fixture
workflow, [`testing-strategy.md`](testing-strategy.md) is the authority. This
playbook covers development-time testability and the per-feature pre-test audit.

---

## Part 1: Development-Time Rules

Follow these when writing any new code. Code that violates these rules will need refactoring
before tests can be written.

### Rule 1: Separate Business Logic from Framework and DB Concerns

Business logic (validation, transformation, computation) must live in pure functions that can be
tested with zero setup — no DB, no Next.js runtime, no mocking. By convention, pure logic lives in
a `domain/` subfolder of the feature's `lib/` (e.g. `lib/submission/domain/`,
`lib/ai-evaluation/domain/`); IO-bound code lives in sibling `helpers/`, `handlers/`,
`orchestrator/`, and the `*-actions.ts` entry points.

```
// ✅ CORRECT: Pure function extracted, testable in isolation
function extractEmailDomain(email: string): string {
  const parts = email.split('@')
  if (parts.length !== 2 || !parts[0] || !parts[1]) {
    throw ErrorTypes.dataIntegrity(ERROR_MESSAGES.INVALID_EMAIL)
  }
  return parts[1].toLowerCase()
}

// The async function uses the pure function
export async function rejectDisposableEmail(email: string): Promise<void> {
  const domain = extractEmailDomain(email)  // Pure logic
  const supabase = createAdminClient()       // DB concern
  // ... DB query
}
```

```
// ❌ BAD: Business logic buried inside the DB function
export async function rejectDisposableEmail(email: string): Promise<void> {
  const supabase = createAdminClient()
  const parts = email.split('@')         // Logic mixed with DB
  if (parts.length !== 2) { ... }        // Can't test without mocking DB
  const domain = parts[1].toLowerCase()
  // ... DB query
}
```

### Rule 2: Push Next.js Runtime to the Boundary

`redirect()`, `cookies()`, `headers()` from Next.js must only appear at the route/page/layout
boundary — never inside business logic or utility functions.

```
// ✅ CORRECT: redirect() at the boundary (layout/page)
// authentication-server-utilities.ts
export async function requireClient(): Promise<ClientWithoutStripeIds> {
  const data = await getAuthenticatedClientData()
  if (!data) {
    redirect(ROUTES.LOGIN)  // Boundary function, called from pages/layouts
  }
  return removeStripeFields(data.client)
}

// Pure logic function for server actions (no redirect)
export async function requireAuthentication(): Promise<AuthData> {
  const data = await getAuthenticatedClientData()
  if (!data) {
    throw ErrorTypes.unauthorized(ERROR_MESSAGES.UNAUTHORIZED)
  }
  return { user: data.user, supabase: data.supabase }
}
```

```
// ❌ BAD: redirect() deep inside business logic
export async function calculateAssessmentScore(assessmentId: string) {
  const session = await getSession()
  if (!session) {
    redirect('/login')  // Framework concern buried in business logic
  }
  // ... scoring logic that's now untestable without Next.js
}
```

### Rule 3: Choose the Right Supabase Client

There are three clients, distinguished by their import path (not by distinct names — both the SSR
and browser clients are named `createClient`):

- **Admin client** — `createAdminClient()` from `@/shared/lib/supabase/admin`. Synchronous, no
  session. For operations that don't need the user's session (blocklist lookups, system queries,
  background jobs). Vitest-testable against local Supabase.
- **SSR / session client** — `createClient()` from `@/shared/lib/supabase/server`. Async, wrapped
  in React `cache()`, reads `cookies()`. Use only when the operation genuinely needs the
  authenticated user's session and RLS context. Tested via Playwright/E2E — never in Vitest with
  mocked cookies.
- **Browser client** — `createClient()` from `@/shared/lib/supabase/client`. Client components only.

The audit signal is the **import path**. If a function imports the SSR client but doesn't actually
need the user's identity or RLS filtering, switch it to the admin client to make it Vitest-testable.

### Rule 4: Server Actions Are Thin Wrappers

Server actions should follow: validate → delegate → return. Business logic lives in separate
functions, not inside the action.

```
// ✅ CORRECT: Action validates and delegates
export async function registerClientAction(data: ClientRegistrationData) {
  return safeServerAction(async () => {
    const validated = clientRegistrationSchema.parse(data)   // Validate
    await rejectDisposableEmail(validated.email)              // Delegate
    const supabase = await createClient()                     // SSR client (server)
    const { error } = await supabase.auth.signUp({ ... })
    if (error) {
      throw ErrorTypes.badRequest(error.message, ERROR_MESSAGES.REGISTRATION_FAILED)
    }
    return {}
  }, { action: 'registerClient' })
}
```

```
// ❌ BAD: Everything in one function
export async function registerClientAction(data: ClientRegistrationData) {
  return safeServerAction(async () => {
    // 50 lines of inline validation, email checks, DB queries,
    // transformation logic, error handling...
  }, { action: 'registerClient' })
}
```

### Rule 5: Components Stay Thin

Components render UI. Business logic goes in hooks. Server communication goes in server actions. A
component should never contain logic that needs its own test — if it does, extract it.

```
// ✅ CORRECT: All logic in hook, component renders
export function LoginForm() {
  const { form, loading, error, submitLogin } = useLogin()
  return (
    <Form onSubmit={handleSubmit(submitLogin)}>
      {error && <Alert>{error}</Alert>}
      <FormField label="Email" ... disabled={loading} />
      <Button disabled={loading}>Sign In</Button>
    </Form>
  )
}
```

### Rule 6: Use ErrorTypes, Not Raw Errors

`throw new Error('...')` produces untyped errors that tests can only assert on message strings
(fragile). `ErrorTypes` produces typed errors with status codes (testable).

```
// ✅ CORRECT: Typed, testable
throw ErrorTypes.badRequest(ERROR_MESSAGES.DISPOSABLE_EMAIL_NOT_ALLOWED)
// Test: expect(isAppError(error)).toBe(true); expect(error.statusCode).toBe(400)

// ❌ BAD: Untyped, fragile test assertions
throw new Error('Disposable email not allowed')
// Test: expect(error.message).toBe('Disposable email not allowed')  // Breaks on wording change
```

### Rule 7: Validation at the Entry Point

Zod validation must happen at the function entry point (server action, API route), not scattered
through the logic. This lets tests verify validation independently from business logic.

```
// ✅ CORRECT: Validate first, then execute
const validated = createAssessmentSchema.parse(data)  // Fails fast with typed error
await createAssessment(validated)                       // Works with clean data

// ❌ BAD: Validation mixed with logic
if (!data.name || data.name.length < 2) { ... }       // Scattered, untestable validation
if (!data.email) { ... }
```

### Rule 8: Isolate AI Calls; Keep Response Handling Pure

AI-backed features (evaluation, summaries, generation, conversation) split into a pure part and a
single network call:

- **Pure, Vitest-testable:** prompt construction, the Zod response schema, the Anthropic
  structured-output format, and response validation / score computation. These live in `domain/`
  (e.g. `ai-evaluation/domain/prompt-builder.ts`, `score-calculator.ts`) or as pure
  `validate*Content` functions in the feature's `ai-client.ts` helper. They take inputs or an
  AI-response object and return/throw — no API call.
- **The model call** goes through the shared structured client (`callClaudeStructured` in
  `@/shared/lib/ai/structured-ai-client.ts`). It is covered by **recorded Anthropic fixtures**, not
  mocks — see the testing-strategy spec for the recording workflow.

```
// ✅ CORRECT: pure prompt + pure validation around a thin model call
const prompt = buildEvaluationPrompt(rubric, submission)   // domain/ — pure, Vitest
const response = await callClaudeStructured(systemPrompt, prompt, {
  outputFormat: EVALUATION_OUTPUT_FORMAT,
  validate: validateEvaluationContent,                     // pure — Vitest (score bounds, proof, metrics)
})
```

---

## Part 2: Pre-Test Audit

Before writing tests for any feature, audit its code against the rules above. This is Step 0 — do
it before writing a single test.

### Audit Checklist

For each function in the feature's `lib/` (including its `domain/`, `helpers/`, `handlers/`,
`orchestrator/` subfolders):

| Check | What to look for | Action if found |
|-------|-----------------|-----------------|
| **Client type** | Which client module does it import — `supabase/server` (SSR, needs session) or `supabase/admin`? Does it actually need the user's session? | If it imports the SSR client but doesn't need session/RLS, switch to the admin client to make it Vitest-testable |
| **Mixed concerns** | Does one function validate + query DB + transform + handle errors? | Extract pure logic into a `domain/` function |
| **Next.js deep imports** | Does `redirect()`, `cookies()`, `headers()` appear inside business logic? | Move to the route/page boundary |
| **Inline AI calls** | Is prompt-building or response validation mixed into the model call? | Extract pure prompt + validation into `domain/`; keep the call thin (Rule 8) |
| **Raw errors** | Does it use `throw new Error()` instead of `ErrorTypes`? | Replace with the appropriate `ErrorTypes` method |
| **Missing validation** | Is input validated with Zod at the entry point? | Add schema validation |
| **Fat server actions** | Does the server action contain inline business logic (>15 lines of non-DB code)? | Extract into testable functions |
| **Cross-feature imports** | Does it import from another feature's `lib/`? | Extract shared logic to `@/shared/` or restructure |

### Classification

After auditing, classify each function:

| Classification | Criteria | Test with |
|---------------|----------|-----------|
| **Vitest-testable** | Pure/`domain` function, or admin-client only, no Next.js runtime | Vitest (domain, server action, or AI-response-validation tests) |
| **E2E only** | Uses the SSR/session client (`createClient` from `supabase/server`), reads cookies, or calls `redirect()` | Playwright |
| **Needs refactoring** | Mixed concerns, logic buried in framework/IO code, raw errors | Fix first, then classify again |

### Mocking

Follow the testing-strategy spec: real services (local Supabase, Stripe test mode), recorded
fixtures for Anthropic, and **no Next.js framework mocking** — SSR/cookie-dependent code is tested
via E2E, not in Vitest with mocked cookies. The single sanctioned mock is a service client mocked
to simulate unavailability, to verify **fail-closed** security behavior (e.g. `rejectDisposableEmail`
rejecting when the blocklist DB is unreachable). Anything else is a smell — refactor instead.

### Fix Before Testing

If any function is classified as "needs refactoring":

1. Extract business logic into pure `domain/` functions
2. Push framework imports to boundaries
3. Replace raw errors with `ErrorTypes`
4. Add Zod validation at entry points
5. Run `npm run typecheck && npm run lint` after refactoring
6. Then proceed to write tests

---

## Part 3: Testability by Layer

Quick reference for what's testable at each layer:

| Layer | What lives here | Testable in Vitest? | Notes |
|-------|----------------|--------------------:|-------|
| **Zod schemas** (`schemas.ts`) | Input validation, type inference | Yes | Pure, zero setup |
| **Domain logic** (`lib/[subdomain]/domain/`) | Business rules, calculations, prompt building, response/score validation | Yes | Pure functions, no DB |
| **Library functions** using admin client | DB operations with `createAdminClient` | Yes | Against local Supabase; seed with factories from `tests/support/` |
| **Library functions** using SSR client (`createClient` from `supabase/server`) | Session/RLS DB operations | No | Playwright (E2E); needs real cookies/session |
| **Server actions** (`'use server'`) | Thin entry points wrapping logic | Depends on client used | Admin client → Vitest, SSR client → E2E |
| **AI helpers** (`ai-client.ts`) | Model calls via `callClaudeStructured` | Pure parts yes | Response validation in Vitest; the call uses recorded fixtures |
| **API routes** (`app/api/`) | HTTP endpoints | Per spec | Proxy routes verified with recorded fixtures (auth, JWT, validation) — see spec |
| **Hooks** (`hooks/`) | Client state, server action calls | Via component tests | Test through the component that uses them |
| **Components** (`components/`) | UI rendering | Yes (RTL) | Only meaningful interaction logic; thin form components → E2E |
| **Orchestration** (`lib/orchestrator/`, container lifecycle/health) | Docker/Traefik IO | No | E2E; pure bits (`traefik-labels`, `session-configuration-builder`) are Vitest-testable |
| **Cached SSR access** (React `cache()`) | Request-scoped dedup; the SSR client is itself cached | E2E preferred | `cache()` + cookies make isolation hard |

Test files live in `__tests__/` next to the code they cover (`lib/__tests__/`,
`lib/[subdomain]/__tests__/`, feature-root `__tests__/`). Shared test support lives in
`tests/support/`: data factories, `auth-helpers.ts` (authenticated sessions for RLS tests), and
`fixtures/ai-responses/` (recorded Anthropic responses).

---

## Reference: Worked Examples

- **auth** — the canonical thin-feature example: pure `schemas.ts` (22 Vitest tests with zero
  setup); `extractEmailDomain` pure-extracted from `rejectDisposableEmail` (real local-Supabase
  tests via factories, plus a single fail-closed mock); thin `authentication-actions.ts`
  (validate → delegate → return); three accessor patterns in `authentication-server-utilities.ts`
  (`require` throws / `require` redirects / `fetch` returns null); thin `login-form.tsx` and
  `signup-form.tsx` with all logic in `useLogin` / `useSignup`.
- **billing** — `lib/domain/` holds pure billing rules and schemas, Vitest-tested
  (`billing-domain.test.ts`, `billing-schemas.test.ts`).
- **assessments** — `lib/ai-generation/` validators (`generation-validator`,
  `mcq-generation-validator`) and `assessment-policies`, Vitest-tested; pure validation split from
  the AI generation call.
- **invitations / ai-evaluation** — the AI pattern in full: pure `domain/prompt-builder.ts` and
  `score-calculator.ts`, pure `validate*Content` functions in `helpers/ai-client.ts`, and the model
  call through `callClaudeStructured` (recorded fixtures).
- **candidate-assessment** — a heavy orchestration feature: pure `domain/` logic (timer, MCQ,
  validation, event-mapping, Traefik labels) is Vitest-testable; container lifecycle and health
  (Docker/Traefik IO) are E2E.
