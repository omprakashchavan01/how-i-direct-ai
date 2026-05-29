# Testing Strategy Design

## Overview

Behavior-based testing strategy for the QAVetted platform. Tests verify what the system does, not how it does it. Every test maps to observable outcomes: DB state changes, return values, HTTP responses, or page content.

Tests run against real services — local Supabase, Stripe test mode, Anthropic production API. No mocking, with one narrow exception for fail-closed security invariants (see Principle 6).

---

## Test Suites

Eight suites organized by boundary, speed, and trigger:

| Suite | Runner | Trigger | Environments | Hits Anthropic? |
|-------|--------|---------|--------------|-----------------|
| **Domain** | Vitest | Every push (CI) | Local + staging | No |
| **Component** | Vitest + React Testing Library | Every push (CI) | Local + staging | No |
| **Server Actions + API Routes** | Vitest | Every push (CI) | Local + staging | No |
| **AI Integration** | Vitest | Every push (CI) | Local + staging | No (recorded fixtures) |
| **E2E** | Playwright | Every push (CI) | Local + staging | No (pre-injected data) |
| **Smoke** | Playwright | On deploy | Production only | No |
| **Container** | Playwright | Manual (local), on deploy (staging) | Local + staging | Yes (proxy flow) |
| **AI Suite** | Vitest | Manual / scheduled | Local or staging | Yes (real API) |

### Suite Details

**Domain tests** exercise pure functions with no DB or network: score calculators, status normalization, progress calculation, generation validators, rubric weight arithmetic, difficulty bounds checks.

**Component tests** use React Testing Library to test React components that contain meaningful client-side logic — conditional rendering, interactive state (multi-step wizards, dynamic form fields), computed display values, or complex user interactions beyond simple form submission. These tests render components in a jsdom environment and assert on user-visible behavior: what appears on screen, what happens when users click/type/navigate, and how the component responds to different props or states. Component tests do NOT test thin form components that simply delegate to server actions — those are covered by E2E tests. Component tests do NOT mock server actions or API calls; they test the component's rendering and interaction logic in isolation from the server.

**Server action + API route tests** call server actions and HTTP endpoints against real local Supabase. Tests set up data via factories (direct DB inserts using admin client), invoke the action/route, and assert on DB state and return values. RLS policies are exercised because the system-under-test uses the regular authenticated client, not the admin client.

**SSR client limitation:** Server actions that use the SSR Supabase client (via `createServerClient` + `cookies()` from `next/headers`) cannot be tested in Vitest — they require real browser cookies managed by the Next.js runtime. These actions are tested exclusively at the E2E layer with Playwright. Only server actions and library functions that use the admin client (`createAdminClient`) or are pure functions can be tested in Vitest. When planning tests for a feature, check whether each action imports from `@/shared/lib/supabase/server` (SSR — needs E2E) or `@/shared/lib/supabase/admin` (admin — works in Vitest).

**AI integration tests** verify the platform's AI pipeline code handles responses correctly: prompt construction, structured output parsing, retry logic, error handling, score computation from AI responses. These use recorded Anthropic API response fixtures — captured once from real API calls, replayed in tests. No API cost at CI time.

**E2E tests** drive a browser via Playwright against a running local app with local Supabase and Stripe test mode. Cover critical user journeys: auth, assessment creation, invitation sending, MCQ submission, billing checkout, dashboard views. Use pre-injected challenge data (factories) to skip AI generation.

**Smoke tests** are lightweight read-only checks against production: health endpoint, auth login works, critical pages load. No test data created in production.

**Container tests** require Docker Desktop (local) or ECS (staging). Cover the candidate experience for component challenges: container provisioning, IDE loading, VNC interaction, workspace submission, and AI proxy flow (extension → proxy → Anthropic → response).

**AI suite** has two tiers — Tier 1 (live integration) and Tier 2 (quality). See "AI Testing Approach" for details on each.

### Same Tests, Any Environment

See Principle 5 for the rule. Per-environment configuration values are listed in Environment Configuration below. Smoke tests are the one exception (production-only).

---

## File Structure

Domain and server action tests co-locate with source code. E2E, container, AI, and smoke suites live in a top-level `tests/` directory.

**Config note:** The existing `vitest.config.ts` includes only `src/**/*.test.ts`. It must be extended to include `tests/ai-quality/**/*.test.ts` for the AI suite (or use a separate vitest config for that suite). Playwright has its own config for E2E, smoke, and container suites.

```
src/
  features/
    [feature]/
      components/
        __tests__/                          # Component tests (React Testing Library)
          [component].test.tsx
      lib/
        __tests__/                          # Server action tests
          [feature]-actions.test.ts
        [subdomain]/
          __tests__/                        # Domain tests (pure functions)
            [module].test.ts
  app/
    api/
      [route]/
        __tests__/                          # API route tests
          route.test.ts

tests/
  e2e/                                      # Playwright E2E (local + staging)
    auth.spec.ts
    assessment-creation.spec.ts
    mcq-submission.spec.ts
    billing-checkout.spec.ts
    invitation-management.spec.ts
    dashboard-results.spec.ts
  smoke/                                    # Playwright smoke (production)
    health.spec.ts
    auth-login.spec.ts
    pages-load.spec.ts
  container/                                # Playwright container (on-demand)
    component-challenge.spec.ts
    api-automation-challenge.spec.ts
    ui-automation-challenge.spec.ts
    ai-proxy-flow.spec.ts
  ai-quality/                               # Vitest AI suite (on-demand)
    live-integration/
      generation-pipeline.test.ts
      evaluation-pipeline.test.ts
      proxy-endpoint.test.ts
      summary-generation.test.ts
    quality/
      scoring-accuracy.test.ts
      scoring-consistency.test.ts
      generation-quality.test.ts
      summary-alignment.test.ts
  support/
    config/
      test-environment.ts                   # Environment-aware configuration
    factories/
      client-factory.ts                     # Test data: clients
      assessment-factory.ts                 # Test data: assessments + skills + challenges
      invitation-factory.ts                 # Test data: invitations + results
      component-factory.ts                  # Test data: components + variants + keywords
      billing-factory.ts                    # Test data: Stripe-related test data
    setup/
      supabase-clients.ts                   # Admin + authenticated test clients
      auth-helpers.ts                       # Create authenticated sessions for tests
    fixtures/
      ai-responses/                         # Recorded Anthropic responses for CI
        challenge-generation-response.json
        evaluation-response.json
        summary-response.json
```

---

## Test Infrastructure

### Environment Configuration

Single config module resolves all environment-specific values:

```
tests/support/config/test-environment.ts
```

Reads `TEST_ENV` (defaults to `local`) and provides:
- Supabase URL and keys (local Docker or staging project)
- Stripe keys (test mode in both environments)
- App base URL (localhost:3000 or staging.qavetted.com)
- Anthropic API key and model (same across environments)
- Feature flags (Docker available, AI suite enabled)

### Factories

Type-safe test data builders using `Database['public']['Tables'][T]['Insert']` types. Each factory:
- Provides sensible defaults for all required fields
- Allows overrides for test-specific values
- Returns the inserted row(s) for use in assertions
- Uses the Supabase admin client (bypasses RLS for setup)

Factory design principle: a factory creates the minimum complete graph of related data. `createTestAssessment()` creates a client, assessment, skills, and challenge slots — because an assessment without these is invalid. Individual table factories exist for edge case tests that need partial data.

### Test Data Isolation

Tests must never interfere with existing data. Each test manages its own data lifecycle:
- **`beforeAll`**: Insert test-specific data using factories (unique identifiers via UUID)
- **`afterAll`**: Delete only the specific data that was inserted — never truncate or wipe tables

No bulk reset functions (`TRUNCATE`, "delete all users", "clear all rows"). Tests use unique identifiers to avoid collisions with each other and with any existing data. This ensures tests are safe to run against local or staging environments without destroying seed data, developer data, or other test data.

Vitest runs test files in parallel by default but tests within a file run sequentially. With UUID-based data isolation, parallel file execution is safe against the shared Supabase instance.

### Environment Restrictions

Tests are restricted to `local` and `staging` environments only. The `TEST_ENV` config rejects any other value. Tests must never run against production.

### Recorded AI Fixtures

Each captured fixture includes the exact Anthropic response body plus metadata (model, usage tokens, stop reason). See "AI Testing Approach > Recorded Fixtures (CI)" for the recording workflow and re-record triggers.

---

## Testing Principles

### 1. Design-first test planning (NON-NEGOTIABLE)

Tests are derived from the **design, behavioral flows, and specifications** — never from reading the implementation. The process:

1. **Understand the design** — read specs, DB schema, and flow documentation to understand what the feature *should* do
2. **Map behavioral flows** — identify all user-facing and system behaviors as flows (e.g., "recruiter sends invitations", "candidate submits assessment")
3. **Derive test cases from flows** — each flow produces Given/When/Then test cases based on the expected behavior defined in the design
4. **Audit the implementation** — compare the code against expected behavior; gaps or deviations are implementation defects to fix, not test adjustments to make

Test planning for a feature is organized by **behavioral flows**, not by code structure. Pure functions, helpers, and utilities are tested as part of the flow they serve, not as standalone implementation concerns.

### 2. Fix implementation defects, never work around them (NON-NEGOTIABLE)

The purpose of writing tests is to find bugs. When test planning or implementation reveals that the code does not match the expected behavior:

- **Buggy behavior**: The code produces wrong results or side effects → fix the code, then write the test asserting correct behavior
- **Incomplete implementation**: A flow is missing validation, error handling, or a state transition the design requires → add the missing implementation, then test it
- **Improper patterns**: The code uses patterns that make testing impossible or unreliable (mixed concerns, buried framework imports, raw errors) → refactor first, then test

Tests must **never** assert on current buggy behavior, skip a scenario because the implementation doesn't handle it, or add workarounds to accommodate bad code. Every test asserts the **correct** (intended) behavior. Bugs found during test development are fixed immediately — not deferred, documented as "expected failures", or worked around.

### 3. Test behaviors, not implementation

Tests assert on observable outcomes. Never import internal helper functions, never assert on function call counts, never check internal state. If you can't observe it from outside the function boundary, it's not a test concern.

Behavior contracts are the test specifications. Each Given/When/Then maps directly to a test:
- **Given** → factory inserts (test setup)
- **When** → server action call, HTTP request, or Playwright action
- **Then** → DB state assertions, return value checks, page content verification

Invariants become test assertions. Forbidden behaviors become negative tests.

### 4. Vertical coverage per feature

When adding tests for a feature, go vertical through the layers for each behavioral flow:

```
Behavioral flow
  → Domain tests (pure business rules that serve this flow)
  → Component tests (client-side rendering and interaction logic)
  → Server action tests (DB flows, validation, error handling)
  → API route tests (HTTP layer, auth, webhook handling)
  → E2E tests (critical user journeys)
```

Not every flow needs all layers. Add tests where they provide confidence at that layer:
- Pure business logic with edge cases → domain tests
- Conditional rendering, interactive state, complex UI logic → component tests
- DB state transitions and RLS enforcement → server action tests
- Webhook signature verification and HTTP semantics → API route tests
- Multi-step user workflows → E2E tests

### 5. Same tests, any environment

Tests are environment-agnostic. The `TEST_ENV` config switches the target. No environment-specific test files. No conditional logic inside tests based on environment. The same test that passes locally must pass against staging.

### 6. Real services, not mocks

- **Supabase**: Real Postgres via Docker locally, hosted Supabase on staging. RLS policies, triggers, constraints all exercised.
- **Stripe**: Test mode (`sk_test_` / `pk_test_`) in all environments. Real API, test card numbers. For webhooks: server action tests can trigger events via Stripe SDK (create checkout session, complete it). E2E tests drive the Stripe checkout UI with test cards. API route tests for webhook endpoints construct webhook payloads with valid test-mode signatures using Stripe's `stripe.webhooks.generateTestHeaderString()`.
- **Anthropic**: Real production API for AI suite and container proxy tests. Recorded fixtures for CI-time AI integration tests.
- **No framework mocking**: No mocking of Next.js internals (navigation, headers, cache). Server actions that depend on the Next.js runtime (SSR Supabase client, cookies) are tested via E2E with Playwright, not in Vitest with mocks.
- **One exception**: Infrastructure failure security invariants may mock a service client to simulate unavailability (e.g., mock `createAdminClient` to verify fail-closed behavior). This is the only legitimate mock pattern — testing that the system blocks rather than allows when a dependency is unreachable.

### 7. Pre-injected data for speed

E2E and server action tests use factories to insert known data directly into the database. This skips AI challenge generation, Stripe customer creation, and other slow setup paths. Tests are fast and deterministic.

For tests that need challenges: insert pre-built challenge data (instructions, rubric, variant) directly via factory. The challenge content matches what AI would generate but is known and stable.

AI live integration tests are the only tests that call Anthropic during setup.

### 8. Tests must be independent

No test depends on another test's side effects. Test execution order must not matter — any test must pass when run in isolation. See Test Data Isolation for the operational mechanics (factories, UUIDs, beforeAll/afterAll).

### 9. Container and AI tests are opt-in

Container tests require Docker Desktop (local) or ECS (staging). They are never part of the default CI pipeline. They run manually in local development and automatically on staging deploy.

AI quality tests require real Anthropic API calls and cost money. They run manually or on a schedule, never on every push.

---

## CI/CD Integration

### Pipeline Stages

```
Push to any branch:
  1. Domain tests (Vitest)
  2. Component tests (Vitest + React Testing Library)
  3. Server action + API route tests (Vitest, requires local Supabase)
  4. AI integration tests (Vitest, recorded fixtures)
  5. E2E tests (Playwright, requires running app + local Supabase + Stripe test mode)

Deploy to staging:
  5. All CI tests above against staging environment
  6. Container tests (Playwright, real ECS containers)

Deploy to production:
  7. Smoke tests (Playwright, health + auth + pages)

Manual / Scheduled:
  8. AI Suite Tier 1: Live integration (few API calls, fast)
  9. AI Suite Tier 2: Quality (more API calls, thorough)
  10. Container tests locally (requires Docker Desktop)
```

### npm Scripts

```
npm run test                 # Domain + component + server actions + API routes + AI integration
npm run test:e2e             # Playwright E2E (non-container)
npm run test:smoke           # Playwright smoke (production)
npm run test:container       # Playwright container tests
npm run test:ai              # AI live integration + quality (both tiers)
npm run test:ai:live         # AI live integration only (tier 1)
npm run test:ai:quality      # AI quality only (tier 2)
npm run test:ci              # Everything that runs on every push
npm run test:coverage        # Coverage report (domain + component + server actions + API routes)
```

### GitHub Actions Integration

Integrates with the existing workflow structure from the staging and production environment plan:

- **CI workflow** (every push): Starts local Supabase via `supabase start`, runs `test:ci`
- **Staging deploy workflow** (`platform-deploy.yml`): After deploy, runs full test suite against staging + container tests
- **Production deploy workflow**: After deploy, runs smoke tests against production
- **Scheduled workflow** (weekly or on-demand): Runs AI quality suite

### CI Requirements

The CI runner needs:
- Node.js 20+
- Supabase CLI (for `supabase start`)
- Docker (for local Supabase containers)
- Playwright browsers (installed via `npx playwright install`)
- `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom` (for component tests)
- Environment variables: Stripe test keys, Anthropic API key (for AI suite only)

---

## How To Add Tests For A Feature

### Step 0: Understand the Design

Before anything else, build a complete understanding of what the feature *should* do. Read the design sources in this order:

1. **Specs and flow documentation** — `docs/` and `/documents/*.md` for the feature's intended behaviors
2. **DB schema** — `supabase/migrations/*.sql` for tables, constraints, triggers, RLS policies, and enums that define the data model and its invariants
3. **Behavior contracts** — `docs/refactor/contracts/[feature]-contract.md` if one exists

From these sources, identify:
- **Behavioral flows** — the user-facing and system journeys (e.g., "recruiter sends invitations to candidates")
- **Core behaviors** (Given/When/Then) → become test cases
- **Invariants** — rules that must always hold (e.g., "cancelled invitations cannot be re-sent")
- **Forbidden behaviors** → become negative tests (assert errors/rejections)
- **Edge cases** → become specific test scenarios

Do NOT read the implementation code at this step. The design is the source of truth for what tests should verify.

### Step 1: Map Behavioral Flows

Organize the feature into behavioral flows, not code modules. Each flow describes a complete user or system journey:

- Name each flow from the actor's perspective (e.g., "Recruiter sends invitations", not "sendInvitations action")
- For each flow, list the expected behaviors as Given/When/Then
- Identify which behaviors involve pure logic, DB state changes, HTTP handling, or UI interaction — this determines the test layer

### Step 2: Audit Implementation Against the Design

Now read the implementation code and compare it against the expected behaviors from Steps 0-1. Follow the Pre-Test Audit checklist in [`testability-playbook.md`](testability-playbook.md).

For each function, check:
- Does it implement the expected behavior correctly? If not → **fix the bug first**
- Is a required behavior missing entirely? → **add the missing implementation first**
- Is the code testable? (mixed concerns, SSR client usage, raw errors, missing validation) → **refactor first**

Classify each function as Vitest-testable, E2E-only, or needs-refactoring. Fix all testability issues and implementation defects before writing a single test.

### Step 3: Choose Test Layers Per Behavior

For each behavior in each flow, decide which test layer provides the most confidence:

| Behavior type | Best layer |
|---------------|------------|
| Pure calculation/transformation | Domain test |
| Conditional rendering, computed display values | Component test |
| Interactive state (wizards, dynamic forms, toggles) | Component test |
| DB read/write with validation (admin client) | Server action test |
| DB read/write with validation (SSR client) | E2E test (see SSR client limitation above) |
| Status transition with triggers | Server action test |
| RLS policy enforcement | Server action test (see RLS testing below) |
| Webhook handling | API route test |
| HTTP auth/parsing | API route test |
| Multi-step user workflow | E2E test |
| Visual feedback (toasts, loading) | E2E test |

### Step 4: Write Tests Vertically Per Flow

For each behavioral flow, write tests from domain (fastest feedback) up to E2E (highest confidence):

**Domain tests** (`src/features/[feature]/lib/[subdomain]/__tests__/`):
- Test pure functions that serve this flow
- No setup needed beyond function inputs
- Assert on return values

**Component tests** (`src/features/[feature]/components/__tests__/`):
- Render the component with `render()` from React Testing Library
- Query elements by accessible roles, labels, and text (`getByRole`, `getByLabelText`, `getByText`)
- Simulate user interactions with `userEvent` (click, type, select)
- Assert on what the user sees: visible text, enabled/disabled states, conditional elements
- Do NOT mock server actions — test rendering and interaction logic only

**Server action tests** (`src/features/[feature]/lib/__tests__/`):
- Set up data with factories
- Call the server action directly
- Assert on DB state changes and return values
- Test error cases (invalid input, unauthorized, not found)

**RLS testing**: To verify RLS policies, create data owned by two different clients via factories. Create authenticated sessions for each client using `auth-helpers.ts`. Call the server action as client A and assert it only sees client A's data. Call as client B and assert the same. This tests that RLS enforces tenant isolation without mocking anything.

**API route tests** (`src/app/api/[route]/__tests__/`):
- Construct HTTP Request objects
- Call the route handler function
- Assert on Response status, headers, body
- Test auth failures, invalid payloads, webhook signatures

**E2E tests** (`tests/e2e/`):
- Set up data with factories (direct DB inserts)
- Navigate browser to the page
- Interact with UI elements
- Assert on page content, navigation, toasts

### Step 5: Use Factories for Setup

Every test creates its own data. Use the factory that provides the minimum viable data graph:

```
createTestClient()                    → client row
createTestAssessment({ clientId })    → assessment + skills + challenge slots
createTestInvitation({ assessmentId })→ invitation + result records
createTestComponent()                 → component + keywords + variants
```

Override only the fields relevant to your test. Let defaults handle everything else.

### Step 6: Assert on Behaviors, Not Internals

```
// GOOD: Assert on observable outcome
// "Given an expired assessment, when candidate submits, then submission is rejected"
const result = await submitChallenge(challengeId, submissionData)
expect(result.error.type).toBe('OPERATION_NOT_ALLOWED')
await expectDatabaseRow('component_challenge_results', { id: resultId }, {
  status: 'not_started'  // unchanged
})

// BAD: Assert on implementation details
expect(validateExpiry).toHaveBeenCalledWith(assessmentId)
expect(supabase.from).toHaveBeenCalledWith('component_challenge_results')
```

---

## AI Testing Approach

### Recorded Fixtures (CI)

AI integration tests in CI use recorded API responses. This tests that platform code correctly:
- Constructs prompts from component specs, rubrics, and submissions
- Parses structured JSON output via Zod schemas
- Handles validation retries with error feedback
- Computes scores from AI criterion ratings
- Handles API errors (rate limits, timeouts)

Fixtures are stored in `tests/support/fixtures/ai-responses/`. Each fixture captures a real Anthropic API response for a specific scenario.

**Recording workflow:**
1. Set `RECORD_FIXTURES=true` environment variable
2. Run the AI integration test — it calls real API, saves response to fixture file
3. Commit the fixture file
4. Future runs replay the fixture (no API call)

**When to re-record:**
- Prompt templates change
- Output Zod schemas change
- New AI integration points added
- Model version changes (responses may differ structurally)

### AI Live Integration (Tier 1)

Verifies end-to-end AI plumbing works. Each test makes a real API call and asserts on structural success:

- **Generation pipeline**: Call with real component specs → assert valid challenge returned (schema passes, rubric weights sum to 100, duration in bounds)
- **Evaluation pipeline**: Submit known test data → assert evaluation completes (all criteria scored, proof items present, total score computed)
- **Proxy endpoint**: Send authenticated request to `/v1/messages` → assert streaming response received
- **Summary generation**: Provide evaluation results → assert summary generated with verdict

These tests do NOT assert on content quality — only that the pipeline completes successfully with structurally valid output.

**AI proxy** is tested at two layers:
- **API route tests (CI, fixtures)**: Verify proxy endpoint auth, JWT capability enforcement, request validation, tool filtering — using recorded fixtures, no real API call.
- **AI live integration (Tier 1)**: Verify end-to-end flow — authenticated request to `/v1/messages` → Anthropic responds → streaming response received. Real API call.
- **Container tests**: Verify the full network path from inside a container through the proxy to Anthropic and back.

### AI Quality (Tier 2)

Validates the quality of AI output. These tests define quality criteria and acceptable ranges:

**Scoring accuracy**: Create golden test cases — known submissions with expected quality levels. Example thresholds (calibrate empirically during implementation):
- A deliberately thorough bug report → expect score > 70
- A minimal/empty submission → expect score < 30
- A partially correct submission → expect score 30-70

Thresholds should be established by running baseline tests against current AI output, then set slightly above observed variance. Document chosen thresholds in test comments. Re-calibrate when prompts or model version change.

**Scoring consistency**: Run the same evaluation 3-5 times (evaluation uses temperature=0 which helps). Example thresholds (calibrate empirically):
- Assert standard deviation of total scores is below a threshold
- Flag if any individual criterion score deviates more than 15 points across runs

**Generation quality**: Generate challenges for known components and verify:
- Instructions reference actual component behaviors (not hallucinated)
- Rubric criteria are relevant to the skill being tested
- Difficulty level matches the requested constraints (duration, points)
- Paradigm assignment is appropriate for the execution environment

**Summary alignment**: Generate summaries from known evaluation results and verify:
- Verdict (strong_pass/pass/borderline/fail) aligns with score ranges
- Key observations reference actual criterion feedback
- No contradictions between summary and detailed scoring

---

## Approach: Direct DB Setup (Approach A)

Tests arrange data by inserting directly into local Supabase via the admin client. The system under test (server action, API route, Playwright) uses the regular authenticated client, which means RLS policies are exercised during the test.

```
Setup:     Factory inserts rows via admin client (bypasses RLS)
Action:    Server action / HTTP request / browser interaction (goes through RLS)
Assert:    Query DB via admin client + check return values
Cleanup:   Truncate tables between test suites
```

**Why this approach:**
- Fast and deterministic — direct inserts, no cascading failures
- Behavior contracts map directly — Given = inserts, When = action, Then = DB assertions
- Edge cases are easy — insert expired assessments, maxed-out quotas, any state directly
- Table coupling mitigated by TypeScript types — schema changes surface as compile errors in factories
- RLS coverage — production auth paths are exercised because the SUT uses the regular client
