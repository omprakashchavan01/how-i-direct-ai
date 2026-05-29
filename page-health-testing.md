# Page Health Testing Design

## Overview

Automated page health testing that catches the same issues a manual QA tester would find:
console errors, accessibility violations, broken UI elements, missing loading states, layout
problems, and slow pages. Every page in the app gets a health check, and any issue at any
severity blocks the deploy.

The health suite is separate from feature E2E tests. Feature tests verify user journeys ("can I
register and log in?"). Health tests scan pages for defects ("does this page have JS errors,
broken images, or accessibility violations?").

---

## Check Categories

Seven categories of checks, covering the full issue taxonomy from manual QA. Each category
defines *what* it looks for and *why*.

### 1. Console / Errors

Browser console output and network failures captured during page load.

- JavaScript exceptions and `console.error` calls
- Failed network requests (4xx and 5xx)
- CORS errors, CSP violations, mixed-content warnings, deprecation warnings

Listeners attach before navigation so nothing is missed, and events are collected until the page
settles. React development-mode warnings (hydration mismatches, missing keys) are treated as real
issues.

### 2. Accessibility

Automated WCAG 2.1 A/AA compliance scan.

- Missing alt text, unlabeled form inputs
- Keyboard navigation issues and focus traps
- Missing or incorrect ARIA attributes
- Insufficient color contrast
- Content unreachable by screen reader, invalid heading hierarchy

Each violation reports the failing element, its impact level, and a remediation link.

### 3. Functional

Whether interactive elements on the page actually work.

- Broken images (failed to load)
- Broken internal links (point to internal routes that 404; external links are skipped as slow and flaky)
- Empty interactive elements (buttons or links with no accessible label — visible text, ARIA label, title, or icon)

**Excluded — "dead buttons" (no event listeners):** React attaches handlers to the root via event
delegation, so per-element listener detection is unreliable and would false-positive on every
React-rendered button.

### 4. UX

Common UX anti-patterns that cause user confusion or data bugs.

- Missing loading indicators: a submit action should show a loading state quickly (disable, spinner, or text change)
- Double-submit protection: a second click before the first action completes should be prevented
- Destructive action confirmation: destructive buttons ("delete", "remove", "cancel subscription") should require a confirmation step before any request fires

These require interaction rather than just scanning, so they target the relevant interactive
elements on the page; a page with no such elements has nothing to check.

### 5. Visual / UI

Layout problems visible to users.

- Horizontal scrollbar (content overflowing the viewport)
- Overlapping interactive elements (significant bounding-box overlap)

**Excluded — clipped text (`overflow: hidden` truncation):** too many components legitimately use
`overflow: hidden` (cards, avatars, constrained layouts), so intentional truncation can't be
distinguished from a bug without per-element configuration, and the check would produce excessive
false positives.

### 6. Performance

Whether pages load fast and render smoothly. Targets follow Google's "good" Core Web Vitals where
applicable:

- Page load time — target 3s
- Largest Contentful Paint (LCP) — target 2.5s
- Cumulative Layout Shift (CLS) — target 0.1
- Large unoptimized images (natural dimensions far exceeding the rendered size)
- Excessive network requests — target under 50

**Production-build only:** the timing-sensitive checks (page load time and LCP) are meaningful only
against a production build — dev servers are too slow to judge them — so they run as a separate
production pass. CLS, image sizing, and request count apply in any environment.

### 7. Content

Placeholder or unfinished content.

- Placeholder text (e.g. "lorem ipsum", "TODO", "TBD", "FIXME")
- Empty states (opt-in per page): when a page has a data-dependent area and the test inserts no data, an empty-state message should appear rather than blank space

---

## Health Check Engine

A single orchestrator runs every category against a page and returns all issues found. A page check
supplies the route to scan and, optionally, an empty-state selector to enable the empty-state check
for that page. The check fails if any issue exists at any severity, and the failure output lists
every issue grouped by category and severity, so the fix list is immediate.

A companion verification suite injects known defects to confirm each category actually catches the
problems it targets — the checks themselves are tested.

---

## Authenticated Pages

Pages behind authentication are health-checked using a pre-established logged-in session: the suite
logs in once and reuses that session across checks, rather than logging in through the UI for every
page. This keeps the suite fast while exercising authenticated routes the way a real user reaches
them.

---

## Reporting and Gating

After a run, results are reported grouped by page, category, and severity, and the report is
produced whether or not the run passes — so developers see exactly what needs fixing. Any issue at
any severity fails the suite and blocks the deploy.

---

## Severity Classification

Issues are classified by severity. All severities currently block the deploy; the classification
communicates priority and leaves room to tune gating later.

| Severity | Examples |
|----------|----------|
| **Critical** | JS exceptions, failed API requests (5xx), page doesn't load, CLS > 0.25 |
| **High** | Accessibility violations with "critical" or "serious" impact, broken links, broken images, LCP > 4s, page load > 5s |
| **Medium** | Accessibility "moderate" impact, missing loading indicators, missing destructive-action confirmation, horizontal scrollbar, LCP 2.5–4s, page load 3–5s, large unoptimized images, excessive network requests |
| **Low** | Accessibility "minor" impact, placeholder text, deprecation warnings, CLS 0.1–0.25 |

---

## CI Integration

The health suite runs in CI on every push and gates deploys — a failing health check blocks the
merge and deploy. The timing-sensitive performance checks run as a separate production-build pass,
since they need a production build to be meaningful (see Performance).

---

## What This Does NOT Cover

Deliberate non-goals:

- **Spell checking / grammar** — requires a dictionary and has a high false-positive rate.
- **Visual regression screenshots** — comparing screenshots across builds needs a dedicated service (Percy, Chromatic) or careful baseline management. Can be added later.
- **Cross-browser testing** — health checks run in Chromium only. Other browsers can be added later as separate runs.
- **Mobile responsiveness** — viewport-specific checks (mobile layout, touch targets) can be added later by running at multiple viewport sizes.
