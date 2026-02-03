---
layout: default
title: What Are Contracts?
parent: Core Concepts
nav_order: 1
permalink: /core-concepts/contracts/
---

# What Are Contracts?
{: .no_toc }

Architectural invariants that must hold, enforced like types.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Compiler Analogy

> **TypeScript rejects type errors at compile time.**
>
> **Specflow rejects architecture errors at test time.**

### TypeScript Example

```typescript
function add(a: number, b: number): number {
  return a + b
}

add("hello", 5)  // ❌ Type 'string' is not assignable to 'number'
```

**TypeScript blocks this at compile time.** You cannot run the code until you fix it.

### Specflow Example

```yaml
# Contract: AUTH-001
rule: "Passwords MUST be hashed using bcrypt before storage"
enforcement: contract_test
```

```typescript
// Implementation
const user = {
  password: password  // ❌ CONTRACT VIOLATION: AUTH-001
}
```

**Specflow blocks this at test time.** You cannot merge the PR until you fix it.

**Same enforcement model. Different boundary.**

---

## What Problem Do Contracts Solve?

### Before LLMs: Slow Generation, Manageable Review

When humans write code slowly:
- ✅ Review scales (you review what you generate)
- ✅ Violations are rare bugs
- ✅ Architecture drift is visible

**Old tools worked:** Code review, TDD, linters

### After LLMs: Infinite Generation, Review Bottleneck

When LLMs generate code infinitely:
- ❌ Review doesn't scale (you review 10x what you generate)
- ❌ Violations are normal behavior
- ❌ Architecture drift is invisible until too late

**Old tools are insufficient.** Specflow fills the gap.

---

## How Contracts Work

### 1. Define What Matters

Contracts are **YAML files** that define architectural rules:

```yaml
contract_type: feature
feature_name: leave_management
description: Time-off request and approval system

invariants:
  - id: LEAVE-001
    rule: "Leave approval MUST debit from leave_entitlements ledger"
    severity: critical
    enforcement: e2e_test

  - id: LEAVE-002
    rule: "Leave balance CANNOT go negative without explicit override"
    severity: critical
    enforcement: contract_test
```

**Think of these like TypeScript type definitions.**

### 2. Generate Tests Automatically

Each invariant maps to a test:

**Contract test (LEAVE-002):**
```typescript
it('LEAVE-002: Balance cannot go negative', async () => {
  const files = await glob('src/features/leave/**/*.ts')

  for (const file of files) {
    const content = fs.readFileSync(file, 'utf-8')

    // Check for balance updates without validation
    if (content.includes('balance -') && !content.includes('if (balance < 0)')) {
      throw new Error(`CONTRACT VIOLATION: LEAVE-002 in ${file}`)
    }
  }
})
```

**E2E test (LEAVE-001):**
```typescript
test('LEAVE-001: Approval debits balance', async ({ page }) => {
  // Given: Staff has 10 days annual leave
  await seedUser({ id: '1', annual_leave_balance: 10 })

  // When: Manager approves 3-day leave request
  await approveLeaverequest('1', { days: 3 })

  // Then: Balance is now 7 days
  const balance = await getLeaveBalance('1')
  expect(balance).toBe(7)  // 10 - 3 = 7
})
```

**Think of these like TypeScript's type checker (`tsc`).**

### 3. Enforce on Every Commit

Contracts run in CI:

```yaml
# .github/workflows/contracts.yml
name: Verify Contracts

on: [push, pull_request]

jobs:
  contracts:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- contracts  # Contract tests
      - run: npm run test:e2e       # E2E journey tests
```

**If a contract fails, the PR is blocked.**

**Think of this like how TypeScript errors block compilation.**

---

## Types of Contracts

### Feature Contracts (Architectural Invariants)

**What:** Rules about how the system MUST work

**Examples:**
- "All database tables MUST have RLS policies"
- "Passwords MUST be hashed before storage"
- "Leave approval MUST debit from balance ledger"

**Enforced by:** Contract tests (static analysis) or E2E tests

**File:** `docs/contracts/feature_*.yml`

```yaml
contract_type: feature
feature_name: authentication
invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed using bcrypt"
    severity: critical
    enforcement: contract_test
```

### Journey Contracts (End-to-End Workflows)

**What:** User flows that define "done"

**Examples:**
- "Staff can request leave and see pending status"
- "Manager can approve leave and staff sees updated balance"
- "User can sign up, log in, and access dashboard"

**Enforced by:** E2E tests (Playwright)

**File:** `docs/contracts/journey_*.yml`

```yaml
contract_type: journey
journey_name: staff_request_leave
dod_criticality: critical
steps:
  - step: 1
    action: "Navigate to /leave-requests"
    expected: "Request form visible"
  - step: 2
    action: "Submit leave request for 3 days"
    expected: "Status shows 'pending'"
```

---

## Contract Severity Levels

| Severity | Meaning | Build Behavior |
|----------|---------|----------------|
| **critical** | MUST pass before release | ❌ Blocks merge if failing |
| **important** | SHOULD pass before release | ⚠️ Warning if failing, but allows merge |
| **future** | CAN fail without blocking | ℹ️ Info only, no block |

**Example:**

```yaml
invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed"
    severity: critical  # Blocks merge if failing

  - id: AUTH-002
    rule: "Password reset emails SHOULD use templates"
    severity: important  # Warns but allows merge

  - id: AUTH-003
    rule: "2FA SHOULD be available"
    severity: future  # Aspirational, no enforcement yet
```

---

## Enforcement Methods

### Contract Tests (Static Analysis)

**Best for:** Checking code patterns, configuration, file structure

**How it works:** Scans code for patterns using regex/AST

**Example:**
```typescript
// Check that all API routes have rate limiting
const routes = await glob('src/routes/**/*.ts')
for (const route of routes) {
  const content = fs.readFileSync(route, 'utf-8')
  if (!content.includes('rateLimit(')) {
    throw new Error(`Missing rate limiting in ${route}`)
  }
}
```

**Runs fast:** Scans files in milliseconds

### E2E Tests (Runtime Verification)

**Best for:** Checking behavior, user flows, database state

**How it works:** Runs app in browser, verifies outcomes

**Example:**
```typescript
// Check that leave approval debits balance
await page.goto('/leave-requests')
await page.click('[data-testid="approve-btn"]')
const balance = await getBalanceFromDB()
expect(balance).toBe(7)  // Was 10, approved 3 days
```

**Runs slower:** Each test takes seconds

---

## When Contracts Run

### 1. Pre-Commit Hook (Optional)

Catch violations before commit:

```bash
# .husky/pre-commit
npm test -- contracts --bail
```

**Blocks commit if contracts fail.**

### 2. Pre-Push Hook (Recommended)

Catch violations before push:

```bash
# .husky/pre-push
npm test -- contracts
npm run test:e2e -- --grep @critical
```

**Blocks push if critical contracts/journeys fail.**

### 3. CI (Required)

Catch violations in PR:

```yaml
# .github/workflows/contracts.yml
jobs:
  verify:
    steps:
      - run: npm test -- contracts
      - run: npm run test:e2e
```

**Blocks merge if any contract fails.**

### 4. Local (Anytime)

Check manually:

```bash
npm test -- contracts
npm run test:e2e
```

---

## Contract Tests vs Unit Tests

| Aspect | Contract Tests | Unit Tests |
|--------|----------------|------------|
| **Purpose** | Enforce architecture | Verify logic |
| **What they test** | Code patterns, structure | Function behavior |
| **How they run** | Scan files (static) | Execute code (runtime) |
| **When they fail** | "Missing RLS policy" | "Expected 5, got 3" |
| **Example** | "All tables have RLS" | "add(2, 3) === 5" |
| **Speed** | Very fast (ms) | Fast (ms-s) |

**Both are needed.** Contracts enforce architecture. Unit tests verify logic.

---

## Example: Full Contract Lifecycle

### 1. Define Contract (Specflow-Writer Agent)

```yaml
# docs/contracts/feature_authentication.yml
invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed using bcrypt"
    severity: critical
    enforcement: contract_test
```

### 2. Generate Contract Test

```typescript
// src/__tests__/contracts/auth.test.ts
it('AUTH-001: Passwords hashed with bcrypt', async () => {
  const files = await glob('src/features/auth/**/*.ts')
  for (const file of files) {
    const content = fs.readFileSync(file, 'utf-8')
    if (content.includes('password:') && !content.includes('bcrypt.hash')) {
      throw new Error(`CONTRACT VIOLATION: AUTH-001 in ${file}`)
    }
  }
})
```

### 3. Implement Feature

```typescript
// src/features/auth/signup.ts
export async function signup(email: string, password: string) {
  const user = {
    email,
    password_hash: await bcrypt.hash(password, 10)  // ✅ Satisfies AUTH-001
  }
  await db.insert(user)
}
```

### 4. Run Tests

```bash
$ npm test -- contracts

✓ AUTH-001: Passwords hashed with bcrypt (12ms)

1 passed
```

### 5. Merge PR

Contract passes → PR approved → Merge

---

## Why This Matters

### Traditional Workflow

```
Write code → Hope reviewer catches issues → Ship → Find bugs in production
```

**Problems:**
- Review doesn't scale
- Violations slip through
- No systematic enforcement

### Specflow Workflow

```
Define contracts → Generate code → Tests enforce → Violations blocked
```

**Benefits:**
- Review is automated
- Violations caught immediately
- Enforcement is continuous

**The compiler analogy holds:**
- TypeScript: `"hello" + 5` → Type error → Build fails
- Specflow: Missing RLS → Contract violation → Build fails

---

## Common Questions

### "Are contracts just fancy linting rules?"

**Partially.** Contracts are like ESLint rules, but:
- They're **domain-specific** (architecture, not syntax)
- They're **connected to journeys** (feature + journey contracts work together)
- They're **enforced in CI** (not just warnings)

Think: **ESLint + TypeScript + E2E tests + enforcement**

### "Can I have contracts without agents?"

**Yes.** Write contracts manually (see [Advanced](/advanced/manual-contracts/)).

But **agents are 3-4x faster**. They generate contracts from GitHub issues automatically.

### "What if a contract is wrong?"

**Update it.** Contracts are YAML files. Edit, commit, push.

```yaml
# Before
rule: "Passwords MUST be at least 8 characters"

# After
rule: "Passwords MUST be at least 12 characters"
```

Update the contract test to match.

### "Can contracts change over time?"

**Absolutely.** Contracts evolve with your system.

**Common pattern:**
1. Start with `severity: future` (aspirational)
2. Implement gradually
3. Upgrade to `severity: important` (warnings)
4. Finalize as `severity: critical` (enforcement)

---

## Next Steps

- **[How Agents Work](/core-concepts/agents/)** — Learn how 18 agents generate contracts
- **[What Are Journeys?](/core-concepts/journeys/)** — Understand end-to-end workflows
- **[Manual Contract Creation](/advanced/manual-contracts/)** — Write contracts by hand

---

## Compiler Analogy Reminder

> **The compiler doesn't trust your types.**
>
> **Specflow doesn't trust your architecture.**

Both enforce boundaries automatically. Both block the build when violated.

**That's the point.**
