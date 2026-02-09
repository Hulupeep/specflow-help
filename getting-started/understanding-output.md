---
layout: default
title: Understanding Output
parent: Getting Started
nav_order: 6
permalink: /getting-started/understanding-output/
---

# Understanding Output
{: .no_toc }

Learn to read contracts, test results, and agent reports.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What Specflow Generates

When waves-controller executes, you get four types of output:

1. **Contracts** (YAML files) — Architectural rules
2. **Contract Tests** (TypeScript/JavaScript) — Enforce contracts via static analysis
3. **E2E Tests** (Playwright) — Enforce journeys via browser automation
4. **Implementation Code** (Your stack) — Feature code that satisfies contracts

**Think of it like TypeScript:**
- Contracts = Type definitions
- Contract tests = Type checker
- E2E tests = Runtime assertions
- Implementation = The actual code

---

## 1. Understanding Contracts

Contracts live in `docs/contracts/*.yml`.

### Feature Contract Example

**File:** `docs/contracts/feature_authentication.yml`

```yaml
contract_type: feature
feature_name: authentication
description: User authentication with email/password and JWT tokens

invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed using bcrypt before storage"
    severity: critical
    enforcement: contract_test
    test_file: src/__tests__/contracts/auth.test.ts

  - id: AUTH-002
    rule: "JWT tokens MUST expire after 24 hours"
    severity: critical
    enforcement: e2e_test
    test_file: tests/e2e/journey_auth.spec.ts

  - id: AUTH-003
    rule: "Failed login attempts MUST be rate limited (5 per minute)"
    severity: important
    enforcement: contract_test

protected_files:
  - path: src/features/auth/signup.ts
    reason: "Password hashing logic"
  - path: src/features/auth/login.ts
    reason: "JWT token generation"

compliance_checklist:
  - item: "All password fields use bcrypt.hash()"
    required: true
  - item: "JWT_EXPIRY set to 24h in config"
    required: true
  - item: "Rate limiting middleware on /auth/* routes"
    required: true
```

**How to read this:**

| Field | Meaning | Example |
|-------|---------|---------|
| `contract_type` | Feature or Journey | `feature` (architectural rule) |
| `feature_name` | What area this covers | `authentication` |
| `invariants[].id` | Unique rule identifier | `AUTH-001` |
| `invariants[].rule` | What MUST hold true | "Passwords MUST be hashed" |
| `invariants[].severity` | How critical | `critical` (blocks release) |
| `invariants[].enforcement` | How verified | `contract_test` or `e2e_test` |
| `protected_files` | Files this contract watches | `src/features/auth/signup.ts` |

**Critical vs Important vs Future:**
- **Critical**: MUST pass before release (blocks merge if failing)
- **Important**: SHOULD pass before release (warning if failing)
- **Future**: CAN fail without blocking (aspirational)

### Journey Contract Example

**File:** `docs/contracts/journey_user_signup.yml`

```yaml
contract_type: journey
journey_name: user_signup
description: User can create account and access dashboard

preconditions:
  - "Database is empty (no existing user with test email)"
  - "User is logged out"

steps:
  - step: 1
    action: "Navigate to /signup"
    expected: "Signup form visible"

  - step: 2
    action: "Fill email and password, submit form"
    expected: "Loading state shown"

  - step: 3
    action: "Wait for redirect"
    expected: "Redirected to /dashboard"

  - step: 4
    action: "Check welcome message"
    expected: "User email displayed in welcome message"

postconditions:
  - "users table contains new row with hashed password"
  - "User session exists in database"
  - "JWT token stored in localStorage"

dod_criticality: critical

test_file: tests/e2e/journey_user_signup.spec.ts
```

**How to read this:**

| Field | Meaning |
|-------|---------|
| `journey_name` | Unique identifier for this workflow |
| `preconditions` | State that MUST exist before test runs |
| `steps[]` | Ordered user actions and expectations |
| `postconditions` | State that MUST exist after test completes |
| `dod_criticality` | Release gating: critical, important, future |
| `test_file` | Where the Playwright test lives |

**Definition of Done (DoD):**
- **Critical journeys** MUST pass before release
- **Important journeys** SHOULD pass (warning if failing)
- **Future journeys** are aspirational (can fail without blocking)

---

## 2. Understanding Contract Tests

Contract tests **scan your code** for patterns. They run on every commit.

### Example Contract Test

**File:** `src/__tests__/contracts/auth.test.ts`

```typescript
import { describe, it, expect } from 'vitest'
import { glob } from 'glob'
import fs from 'fs'

describe('Contract: AUTH-001 - Password hashing', () => {
  it('All password storage uses bcrypt.hash()', async () => {
    // Find all files in auth feature
    const files = await glob('src/features/auth/**/*.ts')

    for (const file of files) {
      const content = fs.readFileSync(file, 'utf-8')

      // Check for password storage without hashing
      if (content.includes('password:') && !content.includes('bcrypt.hash')) {
        throw new Error(`
❌ CONTRACT VIOLATION: AUTH-001
File: ${file}
Rule: Passwords MUST be hashed using bcrypt before storage
Found: password field without bcrypt.hash()
See: docs/contracts/feature_authentication.yml
        `)
      }
    }
  })
})
```

**What it does:**
1. Scans all files in `src/features/auth/`
2. Looks for pattern: `password:` without `bcrypt.hash`
3. Fails with `CONTRACT VIOLATION` if found

**Think of it like ESLint, but for architecture.**

### Running Contract Tests

```bash
# Run all contract tests
npm test -- contracts

# Output (passing):
✓ Contract: AUTH-001 - Password hashing (23ms)
✓ Contract: AUTH-002 - JWT expiry (12ms)
✓ Contract: AUTH-003 - Rate limiting (18ms)

3 passed, 0 failed

# Output (failing):
❌ Contract: AUTH-001 - Password hashing (45ms)

  CONTRACT VIOLATION: AUTH-001
  File: src/features/auth/signup.ts:23
  Rule: Passwords MUST be hashed using bcrypt before storage
  Found: password field without bcrypt.hash()

  Expected:
    password_hash: await bcrypt.hash(password, 10)

  Actual:
    password: password

  See: docs/contracts/feature_authentication.yml

FAIL: 1 failed, 2 passed
```

**The build is now blocked** until you fix the violation.

---

## 3. Understanding E2E Tests

E2E tests **run your app** in a real browser. They verify journeys work end-to-end.

### Example E2E Test (Generated by Playwright-from-Specflow)

**File:** `tests/e2e/journey_user_signup.spec.ts`

```typescript
import { test, expect } from '@playwright/test'

test.describe('J-USER-SIGNUP: User can sign up and access dashboard', () => {

  test.beforeEach(async ({ page }) => {
    // Precondition: Ensure test user doesn't exist
    await page.goto('/api/test/cleanup')
  })

  test('User can sign up with valid credentials', async ({ page }) => {
    // Step 1: Navigate to signup
    await page.goto('/signup')
    await expect(page.locator('[data-testid="signup-form"]')).toBeVisible()

    // Step 2: Fill form and submit
    await page.fill('[data-testid="email-input"]', 'test@example.com')
    await page.fill('[data-testid="password-input"]', 'SecurePass123!')
    await page.click('[data-testid="signup-btn"]')

    // Step 3: Wait for redirect
    await page.waitForURL('/dashboard', { timeout: 5000 })

    // Step 4: Verify welcome message
    const welcome = page.locator('[data-testid="welcome-msg"]')
    await expect(welcome).toContainText('test@example.com')
  })

  test.afterEach(async ({ page, browserName }) => {
    // Postcondition: Verify database state
    const response = await page.goto('/api/test/verify-user')
    const data = await response.json()

    expect(data.user_exists).toBe(true)
    expect(data.password_is_hashed).toBe(true) // bcrypt hash detected
    expect(data.session_exists).toBe(true)
  })
})
```

**How to read this:**

| Section | Purpose | Maps to Contract |
|---------|---------|------------------|
| `beforeEach` | Set up preconditions | `journey.preconditions` |
| `test('...')` | Execute journey steps | `journey.steps[]` |
| `data-testid` selectors | Find UI elements reliably | Specified in GitHub issue |
| `afterEach` | Verify postconditions | `journey.postconditions` |

### Running E2E Tests

```bash
# Run all E2E tests
npm run test:e2e

# Output (passing):
Running 3 tests in journey_user_signup.spec.ts
  ✓ J-USER-SIGNUP: User can sign up with valid credentials (2.3s)
  ✓ J-USER-LOGIN: User can log in with credentials (1.8s)
  ✓ J-USER-LOGOUT: User can log out (1.2s)

3 passed (5.3s)

# Output (failing):
Running 1 test in journey_user_signup.spec.ts
  ✗ J-USER-SIGNUP: User can sign up with valid credentials (5.1s)

  Error: Timeout 5000ms exceeded waiting for URL '/dashboard'

  Expected: /dashboard
  Actual: /signup (still on same page)

  Possible causes:
  - Form submission not triggering redirect
  - Backend signup endpoint returning error
  - Frontend not handling success response

  Screenshot: test-results/signup-failure.png
  Video: test-results/signup-failure.webm

FAIL: 1 failed, 2 passed
```

**The build is blocked** until the journey works end-to-end.

---

## 4. Understanding Agent Reports

When waves-controller finishes, each agent reports what it did.

### Specflow-Writer Report

```
📝 SPECFLOW-WRITER REPORT

Issue: #42 "Add user authentication"

Generated Contracts:
  ✓ docs/contracts/feature_authentication.yml
    - 4 invariants defined (AUTH-001 to AUTH-004)
    - 2 protected files
    - 3 compliance checklist items

  ✓ docs/contracts/journey_user_signup.yml
    - 4 steps defined
    - 3 preconditions
    - 3 postconditions
    - DoD: critical

GitHub Issue Updates:
  ✓ Added "Contracts Created" comment to #42
  ✓ Labeled issue with "specflow-ready"

Next Agent: migration-builder (if database changes needed)
```

### Migration-Builder Report

```
🗄️ MIGRATION-BUILDER REPORT

Issue: #42 "Add user authentication"

Database Changes Needed: YES

Generated Files:
  ✓ supabase/migrations/20260203_create_users_table.sql
    - CREATE TABLE users (id, email, password_hash, created_at)
    - RLS policies: users can read/update own row
    - Indexes: email (unique), created_at

Analysis:
  - No foreign key dependencies
  - Estimated migration time: <1s
  - RLS coverage: 100%

Contract Compliance:
  ✓ AUTH-001: password_hash column (bcrypt compatible)
  ✓ ARCH-004: RLS enabled on users table

Next Agent: edge-function-builder (check if RPC needed)
```

### Playwright-from-Specflow Report

```
🎭 PLAYWRIGHT-FROM-SPECFLOW REPORT

Issue: #42 "Add user authentication"
Source Contract: docs/contracts/journey_user_signup.yml

Generated Tests:
  ✓ tests/e2e/journey_user_signup.spec.ts
    - 4 test steps from contract
    - 3 preconditions in beforeEach
    - 3 postconditions in afterEach
    - data-testid selectors: 4 (from issue spec)

Test Status:
  ⚠️ Tests not yet run (implementation incomplete)

  To run: npm run test:e2e -- journey_user_signup

Estimated Coverage:
  - Signup flow: ✓
  - Login flow: ✓
  - Logout flow: ✓
  - Error cases: ⚠️ (add manually if needed)

Next Agent: test-runner (verify tests pass after implementation)
```

### Test-Runner Report

```
🧪 TEST-RUNNER REPORT

Execution Summary:
  Contract Tests: ✓ 12/12 passed
  E2E Tests: ✗ 1/3 failed

Failed Tests:
  ❌ J-USER-SIGNUP: User can sign up with valid credentials
     Error: Timeout waiting for [data-testid="dashboard"]
     Screenshot: test-results/signup-failure.png

     Root cause analysis:
     - Backend signup endpoint (/api/auth/signup) returns 500
     - Error: "bcrypt is not defined"
     - Fix: Install bcrypt package (npm install bcrypt)

Passing Tests:
  ✓ AUTH-001: Password hashing check
  ✓ AUTH-002: JWT expiry check
  ✓ AUTH-003: Rate limiting check
  ... (9 more)

Recommended Action:
  1. Install bcrypt: npm install bcrypt
  2. Re-run tests: npm run test:e2e
  3. If passing, close issue #42

Issue Status: BLOCKED (1 test failing)
```

---

## 5. Reading Build Failures

When Specflow blocks a build, you'll see clear output:

### Contract Violation Example

```bash
$ npm test -- contracts

❌ CONTRACT VIOLATION DETECTED

Contract: AUTH-001
File: src/features/auth/signup.ts
Line: 23
Rule: Passwords MUST be hashed using bcrypt before storage
Severity: CRITICAL (blocks release)

Current code:
  22 | const user = {
  23 |   password: password,  // ❌ Plaintext password
  24 | }

Expected:
  22 | const user = {
  23 |   password_hash: await bcrypt.hash(password, 10),  // ✅ Hashed
  24 | }

Why this matters:
  Storing passwords in plaintext violates security best practices.
  If the database is compromised, all user passwords are exposed.

To fix:
  1. Import bcrypt: import bcrypt from 'bcrypt'
  2. Hash password before storage
  3. Update column name to password_hash (matches migration)

Reference: docs/contracts/feature_authentication.yml

Build failed. Fix violation to continue.
```

### E2E Failure Example

```bash
$ npm run test:e2e

❌ E2E TEST FAILED

Journey: J-USER-SIGNUP
File: tests/e2e/journey_user_signup.spec.ts
Step: "Wait for redirect to /dashboard"

Error:
  Timeout 5000ms exceeded waiting for URL '/dashboard'

  Expected URL: /dashboard
  Actual URL: /signup

  Form submission did not trigger redirect.

Debug artifacts:
  Screenshot: test-results/signup-5000ms.png
  Video: test-results/signup.webm
  Trace: test-results/trace.zip

Possible causes:
  1. Backend /api/auth/signup endpoint failing (check logs)
  2. Frontend not handling success response (check network tab)
  3. Redirect logic not implemented (check SignupPage.tsx)

To debug:
  npm run test:e2e:ui  # Open Playwright UI
  # Or manually test: curl -X POST http://localhost:3000/api/auth/signup

Build failed. Fix journey to continue.
```

---

## Next Steps

Now that you understand output:

1. **[Explore Core Concepts](/core-concepts/)** -- Dive deeper into contracts, agents, journeys
2. **[Browse Agent System](/agent-system/)** -- Learn about the 23+ agent types
3. **[Read Background](/background/)** -- Understand the academic foundation

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
