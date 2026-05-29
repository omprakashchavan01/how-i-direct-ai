# Component Testing Strategy

This document is the single source of truth for testing **challenge components** (the
applications in `challenge-components/`). For platform testing, see
[`testing-strategy.md`](testing-strategy.md) — this document follows
the same philosophy, adapted to components.

---

## Overview

Behavior-based testing for challenge components. Tests verify **what a component does**,
observed through its HTTP API and its UI — never how it does it.

A component is a small, self-contained application with two distinguishing properties:

1. **It is specified.** `specs/behaviors.md` is a catalog of every testable behavior,
   each with a stable ID. `specs/variants/*.md` document every injected bug, each with a
   deterministic detection. These specs are the **test charter** — tests are derived
   from them, not from reading `server.js`/`app.js`.
2. **It runs in variants.** The same component runs under a `VARIANT_TYPE` (STANDARD
   plus bug variants). Testing therefore has an extra dimension: every behavior is
   verified under STANDARD, and every variant's documented bugs are verified to
   reproduce.

Tests run against the **real running component** — its real Node process, real HTTP
endpoints, real browser UI. No mocking.

The component test suite is the **executable form of the Component Acceptance Procedure**
(`component-design-guide.md` section 11). A component passes acceptance when its suite passes.

---

## Test Layers

Both layers use **Playwright** — one framework, one runner, handling browser automation,
HTTP requests, and per-variant process lifecycle.

| Layer | Verifies | Mechanism |
|-------|----------|-----------|
| **API tests** | API behaviors and calculation/validation rules observable over HTTP (`CALC-*`, `VAL-*`, `API-*`) | Playwright `request` fixture against the component's endpoints |
| **UI tests** | UI behaviors observable in the browser (`UI-*`), and bugs that surface visually | Playwright browser automation against the component's UI |

A behavior is tested at the layer where it is observable. Many behaviors surface on both
layers (e.g. a calculation appears in `/api/calculate` and in the order summary) — each
observable surface gets its own test.

Playwright tests must follow [`playwright-best-practices.md`](playwright-best-practices.md) (no hardcoded
waits, resilient locators, proper assertions, cleanup).

---

## The Variant Dimension

Every component is tested under **every variant it implements**. This is the part unique
to component testing.

| Variant | What the suite asserts |
|---------|------------------------|
| **STANDARD** | Every behavior in `behaviors.md` holds exactly as specified |
| **Each bug variant** | (a) every bug in that variant's spec reproduces via its documented `Detection`; (b) every behavior **not** listed in any of that variant's bugs' `Affects` fields still holds as in STANDARD (no collateral breakage) |

This automates `component-design-guide.md` section 11 steps 2–4 and 7–8.

Bug-variant tests assert the **documented bug is present** — this verifies the component
was *built correctly* (the variant injects exactly its documented bugs). This is distinct
from a candidate's tests, which assert the *spec* and therefore fail against a buggy
variant. The component suite tests the component; the candidate tests the candidate.

---

## Coverage Bar

Full coverage is well-defined because the specs enumerate it:

- **100% of `behaviors.md` behavior IDs** verified under STANDARD.
- **100% of variant-spec bug IDs** verified to reproduce under their variant.
- Every test references the behavior ID or bug ID it covers (in its title or a comment),
  so coverage is auditable against the specs.

A behavior with both a UI and an API surface is covered on **both** surfaces.

---

## File Structure

Test tooling is shared and installed once; test specs live with each component.

```
challenge-components/
├── shared/                          # shared server module (runtime)
├── testing/                         # shared TEST tooling — installed once
│   ├── package.json                 # @playwright/test
│   ├── playwright.config.ts         # base config
│   ├── fixtures.ts                  # per-variant component launcher
│   └── helpers.ts                   # cross-component assertions (response envelope)
├── run-component-tests.sh           # run one component's suite locally
└── {slug}/
    ├── src/  specs/  manifest.json
    └── tests/
        ├── helpers.ts               # this component's spec-derived constants & fixtures
        ├── api/                     # API-layer specs
        │   └── *.spec.ts
        └── ui/                      # UI-layer specs
            └── *.spec.ts
```

- **Tests live in the component folder** (`{slug}/tests/`) alongside `src/` and `specs/`,
  and travel with `challenge-components/` if it is extracted to its own repository. The
  component's published version lives in `manifest.json`; an older version's tests live in
  git history at the corresponding `{slug}@{version}` tag.
- **Test tooling is shared** in `challenge-components/testing/` — one install, one
  config. Adding a component means adding a `tests/` folder: no new install, no new
  config, no new workflow.
- **`testing/` holds only cross-component tooling** — the launcher fixture and platform-wide
  contract assertions (the `{ success, data }` response envelope every component's API uses,
  per the "API response pattern" section of `component-design-guide.md`). Per-component
  spec-derived helpers live in `{slug}/tests/helpers.ts` — see Test Infrastructure > Spec-derived helpers.

---

## Test Infrastructure

### Per-variant component launcher

A Playwright fixture starts the component under a given `VARIANT_TYPE` on a free port,
waits for `/health`, yields the base URL and an API request context, and tears the
process down afterward. The component resolves its shared server module via
`require('@qavetted/component-shared')` (a `file:../shared` dependency); the runner
(`run-component-tests.sh`) runs `npm install` in the component before the suite so that
package is linked into `node_modules`, identically locally and in CI. The fixture itself
only launches the already-installed component.

### Test isolation

A component holds in-memory state (orders, promo usage) for the life of its process.
Isolation is achieved by **a fresh component instance per test** — a new process means
clean state, so test order never matters. Process startup is cheap, and Playwright
parallelizes across workers.

This is the component equivalent of the platform's UUID-based data isolation: the
platform gives each test unique data; component tests give each test a fresh instance.

### Spec-derived helpers

Each component's `{slug}/tests/helpers.ts` holds that component's spec-derived values
(e.g. checkout-flow's promo codes, tax rates, shipping costs from `behaviors.md`) and its
request builders, so a spec change is updated in one place. Cross-component assertion
helpers (the response-envelope checks) live in the shared `testing/helpers.ts` — they
encode a platform-wide contract, not any one component's data.

---

## Testing Principles

### 1. Design-first — derive tests from the specs (NON-NEGOTIABLE)

Tests are derived from `behaviors.md` and `variants/*.md` — the component's design —
never from reading `server.js`/`app.js`. The process: read the specs, list the behaviors
and bugs, write a test for each. Auditing the implementation against the specs comes
*after*; gaps are component defects to fix, not tests to bend.

### 2. Fix component defects, never work around them (NON-NEGOTIABLE)

If a test reveals the component does not match its spec, the component (or the spec, when
the spec is wrong) is fixed. Tests never assert buggy STANDARD behavior, never skip a
behavior because it is unimplemented, never accommodate a defect.

### 3. Test behaviors, not implementation

Tests assert on observable outcomes — HTTP responses, status codes, rendered UI. They
never import component internals or assert on internal state.

### 4. Real component, no mocks

Tests run the real component process and exercise real endpoints and the real UI. There
is nothing to mock — a component has no external dependencies.

### 5. Every variant is covered

STANDARD and every bug variant are tested (see The Variant Dimension). A component is not
accepted until its suite passes for all variants it declares.

### 6. Tests are independent and deterministic

Tests are isolated via a fresh component instance each (see Test Infrastructure > Test
isolation), so test order never matters. No hardcoded waits — follow
[`playwright-best-practices.md`](playwright-best-practices.md). Bug injection is deterministic by design
(`component-design-guide.md` section 5), so detection tests are reproducible.

---

## CI/CD

### One workflow, change-scoped

A **single** GitHub Actions workflow — `.github/workflows/component-pipeline.yml` — holds
both the test phase (A) and the publish phase (B), the standard CI/CD pipeline shape. It is
not one workflow per component, and not one job that runs every component on every push. It
uses change-detection and a dynamic matrix:

1. **Detect job** — inspects the changed paths. If a file under `challenge-components/shared/`
   or `challenge-components/testing/` (or `run-component-tests.sh` or the workflow) changed,
   the affected set is **all components** (every component depends on them). Otherwise, the
   affected set is the components with changed files. Outputs the set as JSON.
2. **Test job** — `strategy.matrix` fans out over the affected set (`fail-fast: false`),
   running **one parallel job per affected component**. Each job self-provisions (Node,
   Playwright browser, dependencies) and runs `run-component-tests.sh {slug}`.

Triggers: `pull_request` and `push` to `main` filtered to `paths: challenge-components/**`,
plus `workflow_dispatch`.

This scales to any number of components: one file to maintain, execution scoped to what
changed, jobs run in parallel.

### Local execution

`./challenge-components/run-component-tests.sh {slug}` runs one component's full suite
locally — the same command each CI matrix job runs.

### Merge gate

Branch protection requires the component test job to pass before merge. A component whose
suite fails cannot be merged.

### CD — publish

The same workflow's Phase B `publish` job runs on push to `main`, gated on Phase A: after
the affected components' tests pass, `publish-component.sh` uploads tarballs and regenerates
`registry.json`. Tested components publish automatically. Publishing needs S3 credentials
provided as repository secrets; the `publish` job is skipped on `pull_request`.

### CI runner requirements

- Node.js 20+
- Playwright browsers (`npx playwright install --with-deps`, cached)
- The `challenge-components/testing/` dependencies

---

## How To Add Tests For A Component

### Step 0 — Read the specs

Read `specs/behaviors.md` (every behavior ID and its testable statement) and every
`specs/variants/*.md` (every bug ID, its `Affects`, and its `Detection`). These define
what the tests must verify. Do not read `server.js`/`app.js` for this step.

### Step 1 — Map behaviors to layers

For each behavior, decide its observable surface(s):
- API behaviors and calculation/validation rules → API tests.
- UI behaviors → UI tests.
- A behavior observable on both surfaces → a test on each.

### Step 2 — Audit the component against the specs

Run the component and compare its behavior to the specs (the section 11 procedure). A mismatch
is a component or spec defect — fix it before writing the test (Principle 2).

### Step 3 — Write the STANDARD suite

Write API and UI tests covering every behavior ID under STANDARD. Each test references
its behavior ID.

### Step 4 — Write the variant suites

For each bug variant, write tests that reproduce each documented bug via its `Detection`,
and spot-check that behaviors outside the variant's `Affects` set still hold.

### Step 5 — Verify coverage

Confirm every behavior ID and every bug ID is referenced by a test. Run
`run-component-tests.sh` for the component and confirm all variants pass.
