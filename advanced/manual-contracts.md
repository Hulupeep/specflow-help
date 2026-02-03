---
layout: default
title: Writing Contracts Manually
parent: Advanced
nav_order: 1
permalink: /advanced/manual-contracts/
---

# Writing Contracts Manually
{: .no_toc }

For when you want full control over contract definitions.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## When to Write Contracts Manually

**Most users should use agents** (3-4x faster). But write contracts manually when:

- **Learning Specflow** — Understanding the format helps
- **Complex invariants** — Agents might miss nuance
- **Legacy codebases** — Reverse-engineering existing patterns
- **Custom enforcement** — Non-standard verification methods

**Time trade-off:**
- Agent-generated: 1-2 min per contract
- Manual: 20-30 min per contract

---

## Step-by-Step Guide

### Step 1: Create the YAML File

**Location:** `docs/contracts/`

**Naming:**
- Feature contracts: `feature_<name>.yml`
- Journey contracts: `journey_<name>.yml`

**Example:**
```bash
touch docs/contracts/feature_authentication.yml
```

---

### Step 2: Define Contract Metadata

```yaml
contract_type: feature
feature_name: authentication
description: User authentication with email/password and JWT tokens

created_from_spec: "GitHub issue #42"
owner: "backend-team"
```

**Fields:**
- `contract_type`: `feature` or `journey`
- `feature_name`: Unique identifier (snake_case)
- `description`: Human-readable summary
- `created_from_spec`: Source (issue, spec doc, etc.)
- `owner`: Team responsible

---

### Step 3: Define Invariants (Feature Contracts)

Invariants are **rules that must always hold**.

```yaml
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
    test_file: tests/e2e/journey_user_login.spec.ts

  - id: AUTH-003
    rule: "Failed login attempts MUST be rate limited (5 per minute)"
    severity: important
    enforcement: contract_test
```

**Fields:**
- `id`: Unique identifier (FEATURE-###)
- `rule`: What MUST hold (imperative, specific)
- `severity`: `critical`, `important`, or `future`
- `enforcement`: `contract_test` (static) or `e2e_test` (runtime)
- `test_file`: Where the enforcement test lives

---

### Step 4: Define Protected Files

Files that this contract watches:

```yaml
protected_files:
  - path: src/features/auth/signup.ts
    reason: "Password hashing logic (AUTH-001)"
  - path: src/features/auth/login.ts
    reason: "JWT token generation (AUTH-002)"
  - path: src/middleware/rateLimit.ts
    reason: "Rate limiting (AUTH-003)"
```

**Purpose:** Contract tests scan these files for violations.

---

### Step 5: Define Compliance Checklist

Human-readable verification steps:

```yaml
compliance_checklist:
  - item: "All password fields use bcrypt.hash()"
    required: true
  - item: "JWT_EXPIRY set to 24h in environment config"
    required: true
  - item: "Rate limiting middleware applied to /auth/* routes"
    required: true
```

**Purpose:** Manual verification during code review.

---

### Step 6: Write Contract Test

**Location:** `src/__tests__/contracts/auth.test.ts`

```typescript
import { describe, it } from 'vitest'
import { glob } from 'glob'
import fs from 'fs'

describe('Contract: feature_authentication', () => {

  it('AUTH-001: Passwords hashed with bcrypt', async () => {
    const files = await glob('src/features/auth/**/*.ts')

    for (const file of files) {
      const content = fs.readFileSync(file, 'utf-8')

      // Check for password storage without bcrypt
      if (content.includes('password:') && !content.includes('bcrypt.hash')) {
        throw new Error(`
❌ CONTRACT VIOLATION: AUTH-001
File: ${file}
Rule: Passwords MUST be hashed using bcrypt before storage
Found: password field without bcrypt.hash()

Expected:
  password_hash: await bcrypt.hash(password, 10)

Actual:
  password: password

See: docs/contracts/feature_authentication.yml
        `)
      }
    }
  })

  it('AUTH-002: JWT expiry set to 24h', async () => {
    const envFile = fs.readFileSync('.env.example', 'utf-8')

    if (!envFile.includes('JWT_EXPIRY=24h')) {
      throw new Error(`
❌ CONTRACT VIOLATION: AUTH-002
File: .env.example
Rule: JWT tokens MUST expire after 24 hours
Expected: JWT_EXPIRY=24h
Actual: ${envFile.match(/JWT_EXPIRY=.*/)?.[0] || 'Not set'}

See: docs/contracts/feature_authentication.yml
      `)
    }
  })

  it('AUTH-003: Rate limiting on auth routes', async () => {
    const routeFile = fs.readFileSync('src/routes/auth.ts', 'utf-8')

    if (!routeFile.includes('rateLimit(')) {
      throw new Error(`
❌ CONTRACT VIOLATION: AUTH-003
File: src/routes/auth.ts
Rule: Failed login attempts MUST be rate limited
Expected: rateLimit() middleware on /auth/* routes
Actual: No rate limiting detected

See: docs/contracts/feature_authentication.yml
      `)
    }
  })
})
```

**Pattern:**
1. Scan relevant files
2. Check for violation patterns
3. Throw error with `CONTRACT VIOLATION: <ID>`
4. Include file, expected, actual, and contract reference

---

### Step 7: Write E2E Test (If Needed)

If `enforcement: e2e_test`, create Playwright test:

**Location:** `tests/e2e/journey_user_login.spec.ts`

```typescript
import { test, expect } from '@playwright/test'

test.describe('AUTH-002: JWT expiry', () => {

  test('JWT token expires after 24 hours', async ({ page }) => {
    // Login and get token
    await page.goto('/login')
    await page.fill('[data-testid="email"]', 'test@example.com')
    await page.fill('[data-testid="password"]', 'password123')
    await page.click('[data-testid="login-btn"]')

    // Get JWT from localStorage
    const token = await page.evaluate(() => localStorage.getItem('jwt'))
    const decoded = JSON.parse(atob(token.split('.')[1]))

    // Verify expiry is 24 hours from now
    const expiryTime = decoded.exp * 1000  // Convert to ms
    const now = Date.now()
    const diffHours = (expiryTime - now) / (1000 * 60 * 60)

    expect(diffHours).toBeCloseTo(24, 0)  // Within 1 hour of 24
  })
})
```

---

### Step 8: Run Tests

```bash
# Run contract tests
npm test -- contracts

# Output (passing):
✓ AUTH-001: Passwords hashed with bcrypt (15ms)
✓ AUTH-002: JWT expiry set to 24h (10ms)
✓ AUTH-003: Rate limiting on auth routes (12ms)

3 passed

# Run E2E tests
npm run test:e2e

# Output (passing):
✓ AUTH-002: JWT token expires after 24 hours (2.1s)

1 passed
```

---

## Journey Contracts (Manual)

### Step 1: Create Journey YAML

**Location:** `docs/contracts/journey_user_login.yml`

```yaml
contract_type: journey
journey_name: user_login
description: User can log in with email/password and access dashboard

dod_criticality: critical

preconditions:
  - "User account exists in database"
  - "User is logged out"
  - "Email is verified"

steps:
  - step: 1
    action: "Navigate to /login"
    expected: "Login form visible"

  - step: 2
    action: "Fill email and password, submit form"
    expected: "Loading state shown"

  - step: 3
    action: "Wait for redirect"
    expected: "Redirected to /dashboard"

  - step: 4
    action: "Check dashboard content"
    expected: "User email displayed in header"

postconditions:
  - "users table: last_login_at updated"
  - "JWT token stored in localStorage"
  - "Session exists in sessions table"

test_file: tests/e2e/journey_user_login.spec.ts
```

---

### Step 2: Write E2E Test from Journey

```typescript
import { test, expect } from '@playwright/test'

test.describe('J-USER-LOGIN: User can log in and access dashboard', () => {

  test.beforeEach(async ({ page }) => {
    // Precondition: User account exists
    await seedUser({
      email: 'test@example.com',
      password_hash: await bcrypt.hash('password123', 10),
      email_verified: true
    })

    // Precondition: User is logged out
    await page.goto('/logout')
  })

  test('User can log in with valid credentials', async ({ page }) => {
    // Step 1: Navigate to login
    await page.goto('/login')
    await expect(page.locator('[data-testid="login-form"]')).toBeVisible()

    // Step 2: Fill form and submit
    await page.fill('[data-testid="email"]', 'test@example.com')
    await page.fill('[data-testid="password"]', 'password123')
    await page.click('[data-testid="login-btn"]')

    // Verify loading state
    await expect(page.locator('[data-testid="loading"]')).toBeVisible()

    // Step 3: Wait for redirect
    await page.waitForURL('/dashboard', { timeout: 5000 })

    // Step 4: Check dashboard content
    const header = page.locator('[data-testid="user-email"]')
    await expect(header).toContainText('test@example.com')
  })

  test.afterEach(async () => {
    // Postcondition: Verify database state
    const user = await getUser('test@example.com')
    expect(user.last_login_at).toBeTruthy()  // Updated

    const session = await getSession(user.id)
    expect(session).toBeTruthy()  // Exists

    // Postcondition: Verify localStorage
    const token = await page.evaluate(() => localStorage.getItem('jwt'))
    expect(token).toBeTruthy()
  })
})
```

---

## Tips for Manual Contract Writing

### 1. Start Simple

Don't try to capture everything at once. Start with critical invariants:

```yaml
# ✅ Simple, enforceable
invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed using bcrypt"
    severity: critical

# ❌ Too vague
invariants:
  - id: AUTH-001
    rule: "Security must be good"
    severity: critical
```

---

### 2. Make Rules Specific

**Bad (vague):**
```yaml
rule: "Authentication should be secure"
```

**Good (specific):**
```yaml
rule: "Passwords MUST be hashed using bcrypt with salt rounds >= 10"
```

---

### 3. Use Imperative Language

**Bad (passive):**
```yaml
rule: "Passwords are hashed"
```

**Good (imperative):**
```yaml
rule: "Passwords MUST be hashed"
```

**Keywords:** MUST, MUST NOT, SHOULD, SHOULD NOT, MAY

---

### 4. Link to Tests

Every invariant needs a test:

```yaml
invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed"
    enforcement: contract_test
    test_file: src/__tests__/contracts/auth.test.ts  # ✅ Explicit link
```

---

### 5. Use data-testid for Journeys

**Bad (fragile selector):**
```typescript
await page.click('.btn-primary')  // Breaks when CSS changes
```

**Good (stable selector):**
```typescript
await page.click('[data-testid="login-btn"]')  // Stable
```

---

## Full Example: Feature Contract

**File:** `docs/contracts/feature_authentication.yml`

```yaml
contract_type: feature
feature_name: authentication
description: User authentication with email/password, JWT tokens, and rate limiting

created_from_spec: "GitHub issue #42"
owner: "backend-team"

invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed using bcrypt with salt rounds >= 10"
    severity: critical
    enforcement: contract_test
    test_file: src/__tests__/contracts/auth.test.ts

  - id: AUTH-002
    rule: "JWT tokens MUST expire after 24 hours"
    severity: critical
    enforcement: e2e_test
    test_file: tests/e2e/journey_user_login.spec.ts

  - id: AUTH-003
    rule: "Failed login attempts MUST be rate limited (5 per minute per IP)"
    severity: critical
    enforcement: contract_test
    test_file: src/__tests__/contracts/auth.test.ts

  - id: AUTH-004
    rule: "RLS policies MUST exist on users table"
    severity: critical
    enforcement: contract_test
    test_file: src/__tests__/contracts/database.test.ts

protected_files:
  - path: src/features/auth/signup.ts
    reason: "Password hashing (AUTH-001)"
  - path: src/features/auth/login.ts
    reason: "JWT generation (AUTH-002), rate limiting (AUTH-003)"
  - path: supabase/migrations/*_create_users.sql
    reason: "RLS policies (AUTH-004)"

compliance_checklist:
  - item: "All password fields use bcrypt.hash() with saltRounds >= 10"
    required: true
  - item: "JWT_EXPIRY environment variable set to '24h'"
    required: true
  - item: "Rate limiting middleware on /auth/* routes"
    required: true
  - item: "users table has RLS enabled"
    required: true
```

---

## Next Steps

- **[Contract Schema Reference](/reference/contract-schema/)** — Full YAML format
- **[Agent-First Approach](/getting-started/)** — Let agents generate contracts (faster)
- **[Core Concepts: Contracts](/core-concepts/contracts/)** — Deep dive into contract theory

---

## Compiler Analogy Reminder

> **TypeScript definitions (.d.ts) are written manually or generated.**
>
> **Specflow contracts (.yml) are the same.**

Manual = more control. Generated = faster.

**Choose based on your needs.**
