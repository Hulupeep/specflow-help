---
layout: default
title: Journey Testing Guide
parent: Advanced
nav_order: 2
permalink: /advanced/journey-testing/
---

# Journey Testing Guide
{: .no_toc }

Advanced patterns for testing end-to-end user workflows.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What Are Journey Tests?

**Journey tests** verify complete user workflows from start to finish. Unlike unit tests (test functions) or integration tests (test modules), journey tests simulate real user behavior.

**Example journeys:**
- User signs up → verifies email → logs in → sees dashboard
- Staff requests leave → manager approves → staff sees updated balance
- Customer adds item → checks out → receives confirmation

**Key trait:** Journeys cross feature boundaries. They test the **system as a whole**.

---

## Journey vs E2E Tests

| Aspect | E2E Tests | Journey Tests |
|--------|-----------|---------------|
| **Scope** | Single feature flow | Cross-feature workflow |
| **Purpose** | Verify feature works | Verify business value delivered |
| **Assertion** | Element visible, API returns 200 | User can accomplish goal |
| **Example** | Login form submits successfully | User can access protected dashboard after signup |

**Journey tests ARE E2E tests**, but with broader scope and business focus.

---

## Journey Contract Structure

Journeys are defined in YAML:

```yaml
contract_type: journey
journey_name: staff_request_leave
description: Staff member can request time off and see pending status

# Definition of Done criticality
dod_criticality: critical  # Must pass before release

# Gherkin scenario
scenario: |
  Scenario: Staff requests annual leave
    Given a staff member is logged in
    And they have 10 days annual leave remaining
    When they submit a leave request for 3 days
    Then they see "Pending Approval" status
    And their balance shows 10 days (not debited yet)

# Test execution
test_file: tests/e2e/journey_staff_request_leave.spec.ts
test_command: pnpm test:e2e -- journey_staff_request_leave

# Dependencies
depends_on:
  - J-AUTH-LOGIN  # Must be able to log in first
  - F-LEAVE-REQUEST  # Leave request feature must exist

# Preconditions (test setup)
preconditions:
  - description: Demo user exists with role=staff
    setup_command: pnpm db:seed --user=demo_staff
  - description: Leave balance seeded
    setup_sql: |
      INSERT INTO leave_entitlements (user_id, leave_type, balance)
      VALUES ('demo-staff-id', 'annual', 10)

# Steps (for test generation)
steps:
  - step: 1
    action: Navigate to /leave-requests
    expected: Page loads, "Request Leave" button visible
    data_testid: request-leave-btn

  - step: 2
    action: Click "Request Leave" button
    expected: Leave request form appears
    data_testid: leave-request-form

  - step: 3
    action: Fill form (start_date, end_date, leave_type=annual)
    expected: Form accepts input
    data_testid: [start-date-input, end-date-input, leave-type-select]

  - step: 4
    action: Submit form
    expected: Success toast appears
    data_testid: toast-success

  - step: 5
    action: Verify request in list
    expected: Request shows "Pending Approval" status
    data_testid: leave-request-status
    assertion: text === "Pending Approval"

  - step: 6
    action: Check balance unchanged
    expected: Balance still shows 10 days
    data_testid: annual-leave-balance
    assertion: text === "10"
```

---

## Writing Journey Tests (Playwright)

### Manual Approach

**File:** `tests/e2e/journey_staff_request_leave.spec.ts`

```typescript
import { test, expect } from '@playwright/test'
import { setupTestUser, seedLeaveBalance } from '../helpers/test-setup'

test.describe('J-STAFF-REQUEST-LEAVE: Staff can request leave', () => {
  test.beforeEach(async ({ page }) => {
    // Precondition 1: Create test user
    await setupTestUser({
      id: 'demo-staff',
      role: 'staff',
      email: 'staff@test.com'
    })

    // Precondition 2: Seed leave balance
    await seedLeaveBalance('demo-staff', 'annual', 10)

    // Given: Staff member is logged in
    await page.goto('/login')
    await page.fill('[data-testid="email-input"]', 'staff@test.com')
    await page.fill('[data-testid="password-input"]', 'password123')
    await page.click('[data-testid="login-btn"]')
    await expect(page).toHaveURL('/dashboard')
  })

  test('Staff can submit leave request and see pending status', async ({ page }) => {
    // Step 1: Navigate to leave requests
    await page.goto('/leave-requests')
    await expect(page.getByTestId('request-leave-btn')).toBeVisible()

    // Step 2: Click request leave
    await page.click('[data-testid="request-leave-btn"]')
    await expect(page.getByTestId('leave-request-form')).toBeVisible()

    // Step 3: Fill form
    await page.fill('[data-testid="start-date-input"]', '2025-03-01')
    await page.fill('[data-testid="end-date-input"]', '2025-03-03')
    await page.selectOption('[data-testid="leave-type-select"]', 'annual')

    // Step 4: Submit
    await page.click('[data-testid="submit-btn"]')
    await expect(page.getByTestId('toast-success')).toBeVisible()

    // Step 5: Verify request in list
    const statusElement = page.getByTestId('leave-request-status').first()
    await expect(statusElement).toHaveText('Pending Approval')

    // Step 6: Balance unchanged (not debited until approved)
    const balanceElement = page.getByTestId('annual-leave-balance')
    await expect(balanceElement).toHaveText('10')
  })
})
```

---

### Agent-Generated Approach

Use `playwright-from-specflow` agent to generate tests from journey contracts:

**Process:**
1. Write journey contract YAML (as shown above)
2. Invoke agent:
   ```
   Task("Generate journey test",
        "{playwright-from-specflow prompt}\n\nContract: docs/contracts/journey_staff_request_leave.yml",
        "general-purpose")
   ```
3. Agent reads contract and generates Playwright test
4. Verify test passes

**Advantage:** Consistent test structure, faster than manual writing.

---

## Journey Testing Patterns

### Pattern 1: Happy Path Only

**When:** Early development, establishing baseline

```yaml
journey_name: happy_path_checkout
scenario: |
  Given items in cart
  When user completes checkout
  Then order confirmation shown
```

**Test:** Single test case, no error handling

---

### Pattern 2: Happy + Critical Error Paths

**When:** Production-ready journeys

```yaml
journey_name: checkout_with_errors
scenarios:
  - name: successful_checkout
    description: Standard flow
  - name: payment_declined
    description: Payment fails, user sees error
  - name: out_of_stock
    description: Item unavailable, user redirected
```

**Tests:** Multiple test cases in one file

---

### Pattern 3: Cross-Role Journeys

**When:** Workflow spans multiple user types

```yaml
journey_name: leave_approval_cycle
steps:
  - actor: staff
    action: Submit leave request
  - actor: manager
    action: Approve request
  - actor: staff
    action: Verify balance debited
```

**Test:** Switch between user contexts

```typescript
test('Full leave approval cycle', async ({ page, context }) => {
  // Act as staff
  await loginAs(page, 'staff@test.com')
  await submitLeaveRequest(page)
  await page.close()

  // Act as manager
  const managerPage = await context.newPage()
  await loginAs(managerPage, 'manager@test.com')
  await approveLeaveRequest(managerPage)
  await managerPage.close()

  // Act as staff again
  const staffPage = await context.newPage()
  await loginAs(staffPage, 'staff@test.com')
  await verifyBalanceDebited(staffPage)
})
```

---

## Handling Complex State

### Database Seeding

**Option 1: SQL Scripts**
```typescript
import { db } from '../helpers/db'

test.beforeEach(async () => {
  await db.query(`
    INSERT INTO users (id, role) VALUES ('test-user', 'staff');
    INSERT INTO leave_entitlements (user_id, balance) VALUES ('test-user', 10);
  `)
})
```

**Option 2: Seed Functions**
```typescript
import { seedUser, seedLeaveBalance } from '../helpers/seed'

test.beforeEach(async () => {
  await seedUser({ id: 'test-user', role: 'staff' })
  await seedLeaveBalance('test-user', 10)
})
```

---

### Time-Dependent Tests

**Problem:** Journey depends on current date

**Solution:** Mock system time

```typescript
import { test, expect } from '@playwright/test'

test('Request leave for next month', async ({ page }) => {
  // Mock date to ensure test is deterministic
  await page.addInitScript(() => {
    const mockDate = new Date('2025-02-15T10:00:00Z')
    global.Date = class extends Date {
      constructor(...args) {
        if (args.length === 0) {
          super(mockDate)
        } else {
          super(...args)
        }
      }
    }
  })

  // Now all date pickers will see Feb 15, 2025
  await page.goto('/leave-requests')
  // Test continues...
})
```

---

### External Service Mocking

**Problem:** Journey calls external API (payment, email, etc.)

**Solution:** Mock network requests

```typescript
test('Checkout with mocked payment', async ({ page }) => {
  // Intercept payment API
  await page.route('**/api/payments', route => {
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ status: 'success', transaction_id: 'mock-123' })
    })
  })

  // Now payment will use mocked response
  await page.goto('/checkout')
  // Test continues...
})
```

---

## Journey Coverage Analysis

### What to Measure

| Metric | Definition | Target |
|--------|------------|--------|
| **Critical journey coverage** | % of critical journeys with passing tests | 100% |
| **DOD gate coverage** | % of release-blocking journeys passing | 100% |
| **Cross-role coverage** | % of multi-actor workflows tested | 80%+ |
| **Error path coverage** | % of critical error scenarios tested | 60%+ |

---

### Coverage Report

**Generate:** `journey-enforcer` agent produces coverage report

**Example output:**
```
Journey Coverage Report
=======================

Critical Journeys (DOD): 4/5 passing (80%)
✅ J-STAFF-REQUEST-LEAVE
✅ J-MANAGER-APPROVE-LEAVE
✅ J-STAFF-VIEW-SCHEDULE
❌ J-MANAGER-BUILD-ROSTER (FAILING)
✅ J-STAFF-CHECK-BALANCE

Important Journeys: 3/3 passing (100%)
✅ J-EXPORT-PAYROLL
✅ J-BLACKOUT-OVERRIDE
✅ J-BULK-LEAVE-REQUEST

Future Journeys: 2/4 passing (50%)
✅ J-WHATSAPP-LEAVE-REQUEST
❌ J-WHATSAPP-NO-SHOW-ALERT (NOT IMPLEMENTED)
```

**Action:** Critical journey failing → Block release, fix immediately

---

## Debugging Journey Tests

### 1. Use Playwright UI Mode

```bash
pnpm test:e2e:ui -- journey_staff_request_leave
```

- See test execution in real-time
- Pause at any step
- Inspect DOM at failure point

---

### 2. Enable Tracing

```typescript
test.use({ trace: 'on' })
```

View trace:
```bash
pnpm exec playwright show-trace test-results/.../trace.zip
```

---

### 3. Save Context on Failure

```typescript
test.afterEach(async ({ page }, testInfo) => {
  if (testInfo.status !== 'passed') {
    await page.screenshot({ path: `error-${Date.now()}.png`, fullPage: true })
    const html = await page.content()
    await fs.writeFile(`error-${Date.now()}.html`, html)
  }
})
```

---

## Performance Optimization

### Parallel Test Execution

**Problem:** 20 journey tests × 60s each = 20 minutes

**Solution:** Run tests in parallel

```typescript
// playwright.config.ts
export default defineConfig({
  workers: 4,  // Run 4 tests simultaneously
  fullyParallel: true
})
```

**Result:** 20 minutes → 5 minutes

---

### Shared Browser Context

**Problem:** Each test launches new browser (slow)

**Solution:** Reuse browser between tests

```typescript
// global-setup.ts
export default async function globalSetup() {
  const browser = await chromium.launch()
  const context = await browser.newContext()
  await context.storageState({ path: 'state.json' })
  await browser.close()
}

// In tests
test.use({ storageState: 'state.json' })
```

---

### Database Reset Optimization

**Problem:** Full db reset before each test (slow)

**Solution:** Transaction-based rollback

```typescript
let transaction

test.beforeEach(async () => {
  transaction = await db.transaction()
})

test.afterEach(async () => {
  await transaction.rollback()
})
```

**Result:** 5s reset → 50ms rollback

---

## Journey Testing Checklist

Before marking a journey as "complete":

- [ ] Journey contract YAML exists in `docs/contracts/`
- [ ] Test file exists at path specified in contract
- [ ] Test passes consistently (3/3 runs)
- [ ] Preconditions automated (not manual setup)
- [ ] All `data-testid` attributes present in UI
- [ ] Cross-role transitions work (if applicable)
- [ ] Error paths tested (at least happy + 1 error)
- [ ] Test runs in <60 seconds
- [ ] No flakiness (no random failures)
- [ ] DOD criticality set correctly

---

## Common Pitfalls

### ❌ Flaky Assertions
**Bad:**
```typescript
await page.click('[data-testid="submit-btn"]')
await expect(page.getByTestId('success-toast')).toBeVisible()  // Race condition!
```

**Good:**
```typescript
await page.click('[data-testid="submit-btn"]')
await page.waitForSelector('[data-testid="success-toast"]', { state: 'visible', timeout: 5000 })
await expect(page.getByTestId('success-toast')).toBeVisible()
```

---

### ❌ Hardcoded Waits
**Bad:**
```typescript
await page.click('[data-testid="submit-btn"]')
await page.waitForTimeout(2000)  // Arbitrary wait
```

**Good:**
```typescript
await page.click('[data-testid="submit-btn"]')
await page.waitForLoadState('networkidle')
// Or wait for specific element
await page.waitForSelector('[data-testid="success-message"]')
```

---

### ❌ Missing Test Isolation
**Bad:**
```typescript
// Test 1 leaves data behind
test('Create user', async () => {
  await createUser('test@example.com')  // Not cleaned up
})

// Test 2 fails because user exists
test('Create user again', async () => {
  await createUser('test@example.com')  // Duplicate key error!
})
```

**Good:**
```typescript
test.afterEach(async () => {
  await db.query('DELETE FROM users WHERE email = $1', ['test@example.com'])
})
```

---

## Three-Tier Journey Gates (Agent Teams)

When using [Agent Teams mode](/agent-system/agent-teams/), journey enforcement is automated via `journey-gate` with three tiers:

| Tier | Scope | When | Blocks |
|------|-------|------|--------|
| **Tier 1** | Single issue | Before closing an issue | Issue closure |
| **Tier 2** | All wave issues | Before starting next wave | Next wave |
| **Tier 3** | Full regression | Before merging to main | Merge |

**Tier 1** runs the Playwright test for a specific issue's journey contract. If it fails, the issue cannot be closed.

**Tier 2** runs all journey tests for every issue in the current wave. If any critical journey fails, the next wave cannot start.

**Tier 3** compares the full test suite against `.specflow/baseline.json` — a known-good snapshot of all test results. Any regression (test that was passing but now fails) blocks the merge.

```bash
# Tier 1: Check a single issue
# journey-gate runs: pnpm test:e2e tests/e2e/journey_staff_request_leave.spec.ts

# Tier 2: Check all wave issues
# journey-gate runs all journey tests for wave issues

# Tier 3: Full regression
# journey-gate compares against .specflow/baseline.json
```

> **Note:** Three-tier gates require Agent Teams mode (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true`). In standard subagent mode, `journey-enforcer` provides coverage analysis without hard gating.

---

## Next Steps

- **[Agent Reference](/agent-system/agent-reference/)** — See `journey-tester`, `journey-enforcer`, and `journey-gate` agents
- **[Agent Teams](/agent-system/agent-teams/)** — Persistent teammate coordination with three-tier gates
- **[DPAO Methodology](/agent-system/dpao/)** — Test journeys in parallel waves
- **[Contract Schema](/reference/contract-schema/)** — Full journey contract YAML format

---

## Journey Testing in Production

**Real examples:**

**Timebreez (childcare scheduling):**
- 4 critical journeys (staff request leave, manager approve, view schedule, check balance)
- 100% coverage before release
- <60s per journey test
- Parallel execution: 4 minutes total

**HookTunnel (webhook infrastructure):**
- 3 critical journeys (create hook, receive webhook, replay request)
- E2E tests run on every commit
- Catches integration issues before production

**Journey tests are the Definition of Done. If they pass, you can ship.**
