# AI-Directed Engineering Standards

These are the engineering and testing standards I wrote to direct AI coding agents while building a B2B SaaS product (a platform for evaluating QA engineers on real work). The entire codebase was written by an AI agent (Claude Code) — I wrote none of the application code by hand. My role is to set the standards, plan the work, review what the agent produces, and hold the quality bar. These documents are how I hold that bar.

I'm publishing them because "I use AI to build and test software" doesn't mean much on its own. This is what it actually looks like in practice: the rules the agent has to follow before it writes a line of code, how testing is approached, and what gets rejected.

A note on authorship: the decisions, principles, and standards here are mine. The AI drafted and maintains the documents under my direction (which is why some revision histories list the agent as author), the same way the AI writes the code under my direction. The judgment is human; the typing is not.

## What's here

**Start with these two if you only read two:**

- **[`testability-playbook.md`](testability-playbook.md)** — How I keep code testable as a development-time discipline, not an afterthought. Includes a pre-test audit that classifies every function as unit-testable, end-to-end-only, or needs-refactoring *before* any test is written.
- **[`CLAUDE.md`](CLAUDE.md)** — The standing instructions the AI agent must follow on every task: the workflow, the prohibitions, the type and error-handling rules, and the testing gates. This is the clearest single view of how I direct the agent.

**Testing approach:**

- **[`testing-strategy.md`](testing-strategy.md)** — The overall testing philosophy: tests verify behavior against written specs, not implementation; real services over mocks; tests derived from design.
- **[`component-testing-strategy.md`](component-testing-strategy.md)** — Testing strategy for the platform's challenge components, including the variant dimension (testing both a correct reference build and deliberately bug-injected builds) and a change-scoped CI pipeline.
- **[`page-health-testing.md`](page-health-testing.md)** — An automated suite that catches what a manual QA tester would: console errors, accessibility (WCAG) violations, broken elements, missing loading states, layout problems, and performance budgets — and gates deploys on them. Includes the checks I deliberately *excluded* and why.
- **[`playwright-best-practices.md`](playwright-best-practices.md)** — Mandatory rules for every Playwright test: independence, no hardcoded waits, locator priority, web-first assertions, parallelism, cleanup.

**Code quality:**

- **[`naming-conventions.md`](naming-conventions.md)** — Domain-aligned naming so code reads like the business it models, not like implementation detail.

## Context

This is a snapshot of standards for one product, in active development. It's meant as a work sample showing how I direct AI — not a general-purpose framework to adopt wholesale. Some references to internal file paths and product specifics remain; they're harmless and left in for authenticity.
