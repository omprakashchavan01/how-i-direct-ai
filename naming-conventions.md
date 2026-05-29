# Naming Conventions Guidelines

## Purpose

This document establishes naming conventions that ensure code is readable, domain-aligned, and consistent across the entire codebase. Names should communicate business intent, not implementation details.

---

## 1. Ubiquitous Language (Domain-Driven Design)

### Principle
Use the **exact terms** that business stakeholders, product managers, and domain experts use. Never invent technical synonyms.

### Rules
- Maintain a glossary of approved domain terms
- All code must use glossary terms consistently
- When in doubt, ask: "Would a product manager understand this name?"

### Examples
```typescript
// ❌ Developer-invented terms
type UserTest = { ... }
function processUserData() {}
const testConfig = { ... }

// ✅ Business/domain terms
type CandidateAssessment = { ... }
function submitChallengeResponse() {}
const assessmentConfiguration = { ... }
```

---

## 2. Name Formation Framework

### Structure
```
[Context] + [Subject] + [Action/State] + [Clarifier]
```

### Components

| Component | Question | Examples |
|-----------|----------|----------|
| **Context** | Which feature/domain? | `candidate`, `assessment`, `invitation`, `challenge` |
| **Subject** | What entity/concept? | `result`, `session`, `configuration`, `token` |
| **Action** | What operation? | `create`, `validate`, `fetch`, `submit`, `update` |
| **Clarifier** | What distinguishes it? | `ById`, `WithDetails`, `FromConfiguration` |

### Application Examples
```typescript
fetchCandidateAssessmentById()
//    [Context]  [Subject]  [Clarifier]

validateInvitationToken()
// [Action] [Context] [Subject]

createAssessmentFromRoleConfiguration()
// [Action] [Subject]    [Clarifier]
```

---

## 3. Syntactic Conventions

### Files and Folders

| Type | Convention | Example |
|------|------------|---------|
| Components | kebab-case | `assessment-form.tsx`, `pricing-card.tsx` |
| Hooks | kebab-case with `use-` prefix | `use-assessment-timer.ts` |
| Server Actions | kebab-case with `-actions` suffix | `invitation-actions.ts` |
| Utilities | kebab-case, descriptive purpose | `container-url-utils.ts` |
| Types | kebab-case | `database-types.ts` |
| Schemas | kebab-case | `configuration-jsonb.ts` |
| Domain logic | kebab-case in `domain/` folder | `domain/assessment-scoring.ts` |

### Functions and Methods

| Type | Convention | Example |
|------|------------|---------|
| Actions | `verb` + `Domain` + `Subject` | `createAssessment`, `sendInvitation` |
| Fetchers | `fetch` + `Subject` + `Clarifier` | `fetchAssessmentById`, `fetchPendingInvitations` |
| Validators | `validate` + `Subject` | `validateInvitationToken`, `validateAssessmentInput` |
| Handlers | `handle` + `Event` | `handleSubmit`, `handleChallengeComplete` |
| Transformers | `transform` + `From` + `To` | `transformDatabaseRowToViewModel` |
| Predicates | `is`/`has`/`can`/`should` | `isAssessmentComplete`, `canSubmitResponse` |

### Variables

| Type | Convention | Example |
|------|------------|---------|
| Regular | camelCase, descriptive | `candidateEmail`, `assessmentConfiguration` |
| Booleans | assertion prefix | `isActive`, `hasPermission`, `canEdit` |
| Arrays | plural nouns | `assessments`, `challenges`, `invitations` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRY_ATTEMPTS`, `DEFAULT_TIMEOUT_MS` |

### Types and Interfaces

| Type | Convention | Example |
|------|------------|---------|
| Database rows | `Table` + `Row` pattern | `Database['public']['Tables']['assessments']['Row']` |
| Domain types | PascalCase + purpose suffix | `AssessmentResult`, `CandidateProfile` |
| Input types | PascalCase + `Input` suffix | `CreateRoleInput`, `UpdateAssessmentInput` |
| Props | PascalCase + `Props` suffix | `AssessmentFormProps`, `ChallengeCardProps` |

### Zod Schemas

| Type | Convention | Example |
|------|------------|---------|
| Schema | camelCase + `Schema` suffix | `createAssessmentSchema`, `loginSchema` |
| Inferred type | PascalCase + `Input` suffix | `CreateAssessmentInput` |

```typescript
// Schema and type MUST be co-located
export const createAssessmentSchema = z.object({ ... })
export type CreateAssessmentInput = z.infer<typeof createAssessmentSchema>
```

---

## 4. Readability Rules

### Rule 1: Names Must Read Like Prose
```typescript
// ❌ Cryptic
const asmtCfg = getAC(roleId)
if (usr.prm.canEdt) { ... }

// ✅ Readable as English
const assessmentConfiguration = getAssessmentConfiguration(roleId)
if (user.permissions.canEditAssessment) { ... }
```

### Rule 2: Reveal Intent, Not Implementation
```typescript
// ❌ Implementation-focused
function queryDatabaseForUserRows() {}
function loopAndFilterArray() {}

// ✅ Intent-focused
function fetchActiveUsers() {}
function filterExpiredInvitations() {}
```

### Rule 3: Be Specific, Avoid Generic Names
```typescript
// ❌ Generic/vague
const data = await fetchData()
const result = process(input)
const items = getItems()

// ✅ Specific/meaningful
const assessmentResults = await fetchAssessmentResults()
const validatedInvitation = validateInvitationInput(rawInput)
const pendingChallenges = getPendingChallenges()
```

### Rule 4: Boolean Names Must Be Assertions
```typescript
// ❌ Ambiguous
const assessment = true
const permission = false

// ✅ Clear assertions
const isAssessmentComplete = true
const hasEditPermission = false
const shouldShowTimer = true
```

### Rule 5: Context-Aware Naming
Names should make sense at the **call site**, not just at definition.

```typescript
// ❌ Requires mental context switching
function send(email: string, id: string) {}

// ✅ Self-documenting at call site
function sendCandidateInvitation(
  candidateEmail: string,
  assessmentId: string
) {}

// Call site reads naturally:
await sendCandidateInvitation(candidateEmail, assessmentId)
```

---

## 5. Prohibited Patterns

### Never Use Abbreviations
```typescript
// ❌ Forbidden
cfg, config → configuration
db → database
req, res → request, response
auth → authentication
env → environment
params → parameters
props → properties (except React Props types)
utils → utilities (or better: specific purpose)

// ✅ Required
assessmentConfiguration
databaseConnection
authenticationToken
environmentVariables
```

### Never Use Vague Names
```typescript
// ❌ Forbidden file names
server.ts, utils.ts, helpers.ts, data.ts, types.ts (standalone)

// ✅ Required
assessment-server-actions.ts
invitation-validation-utilities.ts
challenge-scoring-helpers.ts
candidate-assessment-types.ts
```

### Never Use Generic Function Names
```typescript
// ❌ Forbidden
getData(), processData(), handleClick(), doSomething()
get(), set(), update(), process(), handle()

// ✅ Required
fetchCandidateAssessments()
processChallengeSubmission()
handleAssessmentFormSubmit()
```

---

## 6. Naming Decision Checklist

Before finalizing any name, verify:

| Question | If No, Action Required |
|----------|------------------------|
| Would a new team member understand this? | Add domain context |
| Does it use approved domain vocabulary? | Check glossary, align with business terms |
| Does it reveal intent (what), not implementation (how)? | Rename to describe purpose |
| Is it specific enough to distinguish from similar concepts? | Add qualifier |
| Can it be read aloud naturally as English? | Restructure for prose-like flow |
| Does it match related names in the codebase? | Ensure consistency |
| Does it avoid all prohibited patterns? | Fix abbreviations, vague terms |

---

## 7. Cross-Reference Consistency

### Database to Code Mapping
Use DB snake_case field names directly in TypeScript types — do NOT remap to camelCase. The `Database` types generated from Supabase preserve column names as-is, and consuming code reads those fields with their original snake_case names.

```typescript
// ✅ CORRECT — type fields preserve DB column names
type Assessment = Database['public']['Tables']['assessments']['Row']
const email = assessment.candidate_email
const completed = assessment.is_completed

// ❌ FORBIDDEN — wrapping in remapped types
type Assessment = { candidateEmail: string; isCompleted: boolean }
const email = assessment.candidateEmail
```

Variable names you create (locals, function parameters, props) still use camelCase per sections 3 and 4. This rule applies only to type field access on DB-derived types.

### Feature Consistency
Within a feature, use consistent terminology:

```typescript
// ✅ Consistent within invitation feature
sendInvitation()
validateInvitationToken()
fetchPendingInvitations()
InvitationFormProps
invitationSchema

// ❌ Inconsistent terminology
sendInvite()           // invite vs invitation
validateToken()        // missing context
fetchPending()         // missing subject
InviteFormProps        // invite vs invitation
```

---

## 8. Enforcement

### During Development
1. Reference this document when naming anything new
2. Use the naming decision checklist
3. Read names aloud to verify readability

### During Code Review
1. Verify all names follow these conventions
2. Check domain term consistency with glossary
3. Flag any abbreviated or vague names
4. Ensure cross-file consistency

### Refactoring
1. When touching a file, fix any naming violations found
2. Update all impacted files when renaming
3. Run full validation after changes
