# Playwright E2E Best Practices

Rules for writing Playwright tests in this project. These are mandatory and must be followed for every test.

## 1. Test Independence

- Every test must be completely isolated — own local storage, session, data, cookies.
- Never use `test.describe.serial()` — if tests seem to need ordering, redesign them to be independent.
- Each test sets up its own preconditions and cleans up after itself.
- Tests must be runnable in any order, in parallel, and individually.

## 2. No Hardcoded Waits

- **Never use `page.waitForTimeout()`** — it is always wrong.
- Playwright auto-waits before actions (checks visibility, stability, enabled state, hit target).
- Use web-first assertions that auto-retry: `await expect(element).toBeVisible()`.
- If waiting for a condition, use `page.waitForURL()`, `page.waitForResponse()`, or `expect().toBeVisible()` — never arbitrary milliseconds.
- For long-running operations (e.g., container readiness), use polling/retry patterns like `expect.poll()` or `page.waitForResponse()` with appropriate conditions — never hardcoded sleeps.

## 3. Locator Priority (in order)

1. `page.getByRole()` — buttons, headings, checkboxes (highest priority)
2. `page.getByLabel()` — form fields via their label
3. `page.getByText()` — non-interactive elements
4. `page.getByPlaceholder()` — inputs with placeholder
5. `page.getByAltText()` — images
6. `page.getByTitle()` — elements with title attribute
7. `page.getByTestId()` — last resort, explicit test contract

**Never use** CSS selectors (`page.locator('.my-class')`) or XPath — fragile, breaks with DOM/style changes.

## 4. Web-First Assertions

- **Always** `await expect(locator).toBeVisible()` — auto-waits and retries.
- **Never** `expect(await locator.isVisible()).toBe(true)` — evaluates immediately, no retry.
- Use `expect.soft()` for non-critical checks to collect all failures in one run.

## 5. Test User-Visible Behavior

- Test what users see and interact with, not implementation details.
- Don't assert on CSS classes, internal state, or DOM structure.
- Assert on visible text, URL changes, element visibility/enabled state.

## 6. Data & Cleanup

- Each test creates its own test data in `beforeEach` or at test start.
- Each test cleans up its own data in `afterEach` or at test end — not in `afterAll` for a group.
- Use unique identifiers (UUID) per test run to avoid collisions.
- Never depend on data left by a previous test.

## 7. Authentication

- Use Playwright's storage state pattern for authenticated tests — authenticate once in a setup project, reuse state.
- Store auth state in `playwright/.auth/` directory (gitignored).
- Don't repeat login UI flow in every test — use saved browser state.

## 8. Locator Chaining & Filtering

- Narrow scope by chaining: `page.getByRole('listitem').filter({ hasText: 'Product' }).getByRole('button')`.
- Prefer chaining over complex CSS selectors.

## 9. Debugging & Tracing

- Use `trace: 'on-first-retry'` in config for CI failures.
- Use `--debug` flag for local step-through debugging.
- Prefer trace viewer over videos/screenshots.

## 10. Parallelism

- Run tests in parallel by default (`fullyParallel: true`).
- Design tests to support parallel execution (no shared mutable state).

## 11. Test Organization

- Group by feature/page, not by test type.
- Keep test files focused — one feature area per file.
- Use `test.describe()` blocks for logical grouping within a file.
- Prefix test names with action verbs describing the behavior.

## 12. Timeouts

- Set all timeouts globally in `playwright.config.ts` — never per-assertion or per-test.
- No custom `{ timeout: 15_000 }` on individual assertions.
- If something takes longer than the global timeout, the solution is a proper wait mechanism (polling, `waitForResponse`, `expect.poll()`), not a bigger number.
