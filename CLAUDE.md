# Development Guidelines

Standing instructions for this repository. When a doc in `/documents` covers a topic in
depth, that doc is the authority; this file is the index plus the rules that live nowhere else.

## Workflow

Understand before writing code:
- Create a TodoWrite plan for the discovery work.
- Read the relevant `/documents/**/*.md` specs and the DB schema in `supabase/migrations/*.sql`.
- Review the existing patterns in related files.
- List the bad patterns you'll need to fix, and fix them before building on them.

Implement:
- Make the change comprehensively: trace every file the change affects and update them all
  in the same pass. No partial updates that leave some call sites stale.
- After changing the DB schema, add a migration, then run `supabase db reset && npm run gen:types`.

Validate before claiming the task is done — all three must be clean:
```bash
npm run typecheck     # no type errors
npm run lint          # no warnings
npm run check-unused  # no unused exports
```

## Hard Rules

Each rule states what not to do and what to do instead.

- **Reverting changes:** never use `git checkout`, `git restore`, or `git reset` to undo
  work — they discard all uncommitted changes in the affected files. To undo your own edit,
  reverse just that edit with the editor.
- Don't use `console.log/debug/info`. Use `logger` (the sole exception is `src/shared/lib/errors/index.ts`).
- Don't use `_`-prefixed names or `eslint-disable` comments to silence warnings. Fix the
  underlying cause the warning points to.
- Don't use fallback values (`|| default`, `?? fallback`) to cover possibly-missing data.
  Validate the value first; a silent default hides a missing/invalid value instead of surfacing it.
- Don't swallow errors. Throw via `ErrorTypes` and let them surface (see `error-handling-guide.md`).
- Don't use barrel exports (`index.ts` re-exports). Import directly from the defining module.
- Don't leave TODOs, placeholders, commented-out code, or unused imports/exports. Implement it fully or remove it.
- Don't hardcode values. Use enums, config, constants, or `@/config/routes` for paths.
- Don't use vague names (`server.ts`, `utils.ts`, `data`, `process`) or abbreviations
  (`cfg`, `db`, `req`, `res`). Use full business-domain words (see [`naming-conventions.md`](naming-conventions.md)).
- Don't write backward-compatibility or legacy shims. This is pre-production — change the
  code directly and delete the old path.
- Don't edit mechanically from grep/search hits. Read the whole file first so the change fits its full context.
- Don't rely on the alphabetical execution order of DB triggers. If triggers must run in a
  set sequence, merge them into a single trigger function — name-based order is an
  implementation detail, not a contract.

## Core Approach

- Fix the root cause, not the symptom. If the DB schema is the real problem, change the
  schema rather than working around it.
- Understand the relevant DB schema fully before implementing against it.
- When sources disagree, the migrations are the source of truth.
- Before changing anything, trace the affected flows and the cascade impact across files.
- For any non-trivial decision, present the options and trade-offs and wait for the user's
  confirmation before changing code.
- Evaluate the design as the end user (recruiter, candidate) would, not only technically.
- Before writing a custom solution, check whether the SDK/library already provides it
  (e.g. Anthropic `zodOutputFormat()`, Stripe's built-in webhook verification, Supabase
  typed clients). Verify against the official docs; never assume API behavior.

## Type System

The Supabase-generated types are the single source of truth for data shapes. Use them
directly and centrally so each column's type is expressed in exactly one place: when the DB
changes, `gen:types` regenerates the types and the change surfaces as a compile error
everywhere it's used, rather than being silently swallowed or hand-patched field-by-field.

- Use the generated types directly: `database-tables.ts` for table rows, `database-enums.ts`
  for enum values (never hardcode enum string literals), `configuration-jsonb.ts` for JSONB columns.
- Type both function parameters and return values with the generated `Row` / `Insert` /
  `Update` types (or a `Pick<>` of them). Pass and return whole typed objects — never loose,
  individually-typed scalar fields.
- Reference a column's type only through its generated type. Never re-declare a field's type
  inline, so there is one place to change when the column changes.
- Don't create a wrapper type that mirrors a DB table — use `Row`/`Insert`/`Update` directly.
  Create a custom type only when it represents a genuine domain/business concept with no DB equivalent.
- Never type a required column as optional or nullable. The type must match the schema's
  NOT NULL constraints, so absent data is a compile error.
- If a needed type doesn't exist yet, create it (e.g. define the JSONB schema below), then use it.

Don't suppress the type system — use it. The next three all break the single-source-of-truth
guarantee by forcing a shape the compiler can't verify, so a schema change won't raise the error it should:
- Never use type assertions (`as Type`, `as any`, `as unknown`). Narrow with a type guard,
  or validate the value at the boundary so its shape is proven.
- Never use the `!` non-null assertion. Make the type non-nullable, or guard the value before using it.
- Never use `keyof typeof`. Define a proper enum or union (use `database-enums.ts` for DB enums) and reference that.

For a JSONB column with no schema in `configuration-jsonb.ts`, define a Zod schema and its
inferred type there, then build payloads from a `Pick<>` of the Database `Update` type:
```typescript
export type ComponentUpdatePayload = Pick<
  Database['public']['Tables']['component_challenge_results']['Update'],
  'submission_data' | 'status' | 'submitted_at'
>
```

- Co-locate each Zod schema with its inferred type in `schemas.ts`; keep DB/domain types in
  `types.ts`. Exactly one `types.ts` and one `schemas.ts` per feature.

## Documentation

- Write concise, unambiguous docs; use the same terminology in the docs and the code.
- Document functional requirements only — not implementation details.
- When updating a doc section, rewrite the whole section; don't layer in incremental comments.
- Document only facts confirmed from the code or DB schema; remove redundant or outdated content.
- After any code change, update the docs it affects.
- Keep documentation in `/documents/**/*.md`.
- Hierarchy of truth when sources conflict: (1) released/production code, (2) `/documents/**/*.md`,
  (3) the DB schema in migrations, (4) never invent or silently override a spec — surface the conflict.

## Error Handling
Follow `error-handling-guide.md`.

## Naming
Follow [`naming-conventions.md`](naming-conventions.md).

## UI / UX

Follow `ux_playbook.md` for UX strategy and screen contracts.

- For any UI work — new components, layout changes, redesigns — generate the design with the
  `/frontend-design` skill before writing UI code.
- Use shadcn/ui and Radix components first; search `@/shared/components/ui/` before creating one.
- Use the CSS variables from `globals.css`; never hardcode colors.
- Use the `Heading` and `Text` components for all typography.
- Validate forms on both client and server.
- Disable the submit button while the operation is in flight (`disabled={isSubmitting}`).
- Show loading states with Suspense boundaries.
- Show field-level, actionable error messages under each field.
- Validate timing: on submit for new records, on blur for existing ones.
- Business logic lives in hooks; components hold UI only; server operations live in server actions.
- Group code by feature, not by technical layer; don't import across features.
- Use React Query for all remote server state, Zustand only for ephemeral UI state, one store per domain.

## Security & Data

- Validate all external input with Zod at the entry point (server action, API route, webhook)
  before any logic runs.
- Keep secrets server-side only.
- Throw on missing data — never return null to signal absence.
- Verify Stripe webhook signatures.
- Enforce RLS on every table.
- Don't add app-level ownership checks that the RLS policies already enforce.
- Prefer DB-level constraints (CHECK, conditional NOT NULL) over app-level validation, and
  drop integrity checks that are redundant with a constraint or with prior Zod validation.
- **No security fallbacks in any environment, including local development.**
- Never delete Docker images without explicit user permission.

## Supabase

Follow `supabase-database-guidelines.md`.

- Use the Supabase CLI locally.
- Create a migration for every schema change.
- Test with `supabase db reset` before pushing, and run `npm run gen:types` after schema changes.
- Never change the database directly.
- Use the typed client with the `Database` type. Import the client inside each function;
  never pass it in as a parameter.

## Code Quality

- Use explicit types for all parameters and return values.
- Use early returns and flat code; prefer a guard-and-return over nested conditionals.
- Functional TypeScript — no classes.
- Server Components by default.
- Get routes from `@/config/routes`; never hardcode paths.
- Call the Supabase client directly in each function; don't wrap a single query in its own
  function — that adds indirection without value.
- Inline a helper that's used once or wraps a single operation; don't add an abstraction you
  don't need yet.
- Improve working code by reorganizing it, not by rewriting it from scratch.

## Testability

Follow [`testability-playbook.md`](testability-playbook.md). In addition:
- Keep business logic in pure functions, separate from DB and framework concerns.
- Confine Next.js runtime calls (`redirect`, `cookies`, `headers`) to the route/page/layout boundary.
- Use the SSR client (`createServerClient`) only when you need the user's session/RLS;
  otherwise use the admin client.
- Keep server actions thin: validate → delegate → return.
- Keep components thin: logic in hooks, server work in server actions.

## Testing

- Before writing any test, read [`testing-strategy.md`](testing-strategy.md)
  — the single source of truth for testing strategy and structure. Re-read it each testing session.
- Follow [`playwright-best-practices.md`](playwright-best-practices.md) for all Playwright E2E tests.
- Run the Pre-Test Audit (in [`testability-playbook.md`](testability-playbook.md)) before writing tests:
  classify each function as Vitest-testable, E2E-only, or needs-refactoring, and fix
  testability issues before writing a single test.
- Keep the testing spec in sync: when the implementation diverges from it, update the spec
  before marking the work done.

## AI Prompts

Treat AI prompts as core product. Review every prompt change thoroughly for clarity and
intent. Never edit a prompt mechanically or in bulk.

## Commands

```bash
supabase migration new <name>
supabase db reset
npm run gen:types   # then check database-enums.ts
```

## Done means

- The hard rules hold, and `typecheck`, `lint`, and `check-unused` are all clean.
- Every file the change affects is updated; DB types match the schema.
- Affected docs are updated.
- The TodoWrite plan is complete.
