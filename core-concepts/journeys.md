---
layout: default
title: What Are Journeys?
parent: Core Concepts
nav_order: 3
permalink: /core-concepts/journeys/
---

# What Are Journeys?
{: .no_toc }

End-to-end workflows that define "done".
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Definition of Done Problem

**Traditional workflow:**
```
Feature "complete" → Code review approved → Ship → Hope nothing breaks
```

**Problem:** What does "complete" mean?
- UI works in isolation?
- Backend works in isolation?
- They work together end-to-end?
- Edge cases handled?

**Specflow journeys solve this:** A feature is **done** when its critical journeys pass.

---

## What is a Journey?

A **journey** is an end-to-end user workflow that must work for a feature to be considered complete.

**Example journey:** "Staff can request leave and see pending status"

```yaml
contract_type: journey
journey_name: staff_request_leave
dod_criticality: critical

preconditions:
  - "Staff user is logged in"
  - "Staff has leave entitlement (>0 days available)"

steps:
  - step: 1
    action: "Navigate to /leave-requests"
    expected: "Leave request form visible"

  - step: 2
    action: "Select dates and leave type"
    expected: "Form accepts input"

  - step: 3
    action: "Submit leave request"
    expected: "Success message shown"

  - step: 4
    action: "Check request list"
    expected: "New request shows status: pending"

postconditions:
  - "leave_requests table contains new row with status='pending'"
  - "leave_entitlements.balance unchanged (approval debits, not request)"
  - "Manager receives notification (email or WhatsApp)"
```

**This journey maps to a Playwright E2E test.**

---

## Journey vs Feature Contracts

| Aspect | Feature Contract | Journey Contract |
|--------|------------------|------------------|
| **What** | Architectural rules | User workflows |
| **Examples** | "Passwords MUST be hashed" | "User can sign up and log in" |
| **Enforced by** | Contract tests (static) or E2E | E2E tests (Playwright) |
| **Scope** | Single feature area | Cross-feature integration |
| **When used** | Always (for every feature) | When workflow spans multiple features |

**Both are needed:**
- **Feature contracts** ensure architecture is correct
- **Journey contracts** ensure workflows actually work

---

## Journey Criticality Levels

Not all journeys are equal. Specflow uses **DoD (Definition of Done) criticality:**

| Criticality | Meaning | Release Behavior |
|-------------|---------|------------------|
| **critical** | MUST pass to ship | ❌ Blocks release if failing |
| **important** | SHOULD pass to ship | ⚠️ Warns but allows release |
| **future** | Nice to have | ℹ️ No enforcement, aspirational |

**Example:**

```yaml
# Critical journey (blocks release)
journey_name: staff_request_leave
dod_criticality: critical

# Important journey (warns if failing)
journey_name: staff_check_balance
dod_criticality: important

# Future journey (aspirational)
journey_name: whatsapp_leave_request
dod_criticality: future
```

---

## Journey Lifecycle

### 1. Define Journey (Contract)

Created by **specflow-writer** agent from GitHub issue:

```yaml
# docs/contracts/journey_staff_request_leave.yml
contract_type: journey
journey_name: staff_request_leave
dod_criticality: critical

steps:
  - step: 1
    action: "Navigate to /leave-requests"
    expected: "Form visible"
  # ... more steps ...

test_file: tests/e2e/journey_staff_request_leave.spec.ts
```

### 2. Generate E2E Test

Created by **playwright-from-specflow** agent:

```typescript
// tests/e2e/journey_staff_request_leave.spec.ts
import { test, expect } from '@playwright/test'

test.describe('J-STAFF-REQUEST-LEAVE', () => {

  test.beforeEach(async ({ page }) => {
    // Precondition: Staff user logged in
    await loginAsStaff(page, 'staff@example.com')

    // Precondition: Staff has leave entitlement
    await seedLeaveEntitlement({ user_id: '1', annual_leave_balance: 10 })
  })

  test('Staff can request leave and see pending status', async ({ page }) => {
    // Step 1: Navigate to leave requests
    await page.goto('/leave-requests')
    await expect(page.locator('[data-testid="leave-form"]')).toBeVisible()

    // Step 2: Select dates
    await page.fill('[data-testid="start-date"]', '2026-03-01')
    await page.fill('[data-testid="end-date"]', '2026-03-03')
    await page.selectOption('[data-testid="leave-type"]', 'annual')

    // Step 3: Submit request
    await page.click('[data-testid="submit-btn"]')
    await expect(page.locator('[data-testid="success-msg"]')).toBeVisible()

    // Step 4: Check request list
    await expect(page.locator('[data-testid="status-pending"]')).toBeVisible()
  })

  test.afterEach(async () => {
    // Postcondition: Verify database state
    const request = await getLeaveRequest('1')
    expect(request.status).toBe('pending')

    const balance = await getLeaveBalance('1')
    expect(balance).toBe(10)  // Unchanged (approval debits, not request)
  })
})
```

### 3. Run Journey Test

Run by **test-runner** agent or manually:

```bash
npm run test:e2e -- journey_staff_request_leave

# Output (passing):
✓ J-STAFF-REQUEST-LEAVE: Staff can request leave (2.3s)

# Output (failing):
✗ J-STAFF-REQUEST-LEAVE: Staff can request leave (5.2s)

  Error: Timeout waiting for [data-testid="success-msg"]

  Step failed: "Submit request"
  Expected: Success message shown
  Actual: Form still visible, no message

  Screenshot: test-results/request-leave-failure.png
```

### 4. Verify Journey Before Release

Run by **journey-enforcer** agent:

```
journey-enforcer:
  Checking critical journeys...

  ✓ J-STAFF-REQUEST-LEAVE: Passing
  ✓ J-MANAGER-APPROVE-LEAVE: Passing
  ✗ J-STAFF-VIEW-SCHEDULE: FAILING (timeout on /schedule)

  Critical journeys: 2/3 passing

  ❌ RELEASE BLOCKED
  Reason: 1 critical journey failing

  Fix J-STAFF-VIEW-SCHEDULE before release.
```

---

## Journey Patterns

### Single-Feature Journey

**Example:** User signup (all within auth feature)

```yaml
journey_name: user_signup
steps:
  - Navigate to /signup
  - Fill form
  - Submit
  - Redirected to /dashboard
```

**Scope:** One feature area (authentication)

### Cross-Feature Journey

**Example:** Staff request leave (touches leave + notifications)

```yaml
journey_name: staff_request_leave
steps:
  - Navigate to /leave-requests (leave feature)
  - Submit request (leave feature)
  - Manager receives notification (notifications feature)
```

**Scope:** Multiple features working together

### Multi-Step Journey

**Example:** Manager approves leave (complex workflow)

```yaml
journey_name: manager_approve_leave
steps:
  - Manager logs in
  - Navigates to pending requests
  - Approves 3-day leave request
  - Staff balance decremented by 3
  - Staff sees updated balance
  - Roster shows leave dates
```

**Scope:** Cross-feature + database + integrations

---

## Preconditions and Postconditions

### Preconditions (Setup State)

**What:** State that MUST exist before the journey can run

**Examples:**
- "User is logged in"
- "Database has test data (10 leave days available)"
- "Manager role exists in roles table"

**Implemented in:**
```typescript
test.beforeEach(async ({ page }) => {
  // Set up preconditions
  await loginAsStaff(page)
  await seedLeaveEntitlement({ balance: 10 })
})
```

### Postconditions (Verify Outcomes)

**What:** State that MUST exist after the journey completes

**Examples:**
- "leave_requests table contains new row"
- "leave_entitlements.balance = 7 (was 10, approved 3)"
- "Audit log contains approval entry"

**Implemented in:**
```typescript
test.afterEach(async () => {
  // Verify postconditions
  const request = await getLeaveRequest('1')
  expect(request.status).toBe('approved')

  const balance = await getLeaveBalance('1')
  expect(balance).toBe(7)
})
```

**Postconditions enforce invariants at the database level.**

---

## data-testid Selectors

Journeys rely on **stable selectors** to find UI elements. Specflow uses `data-testid`:

**Why data-testid?**
- ✅ Doesn't break when CSS classes change
- ✅ Doesn't break when text content changes
- ✅ Clear intent: "This element is for testing"

**Example:**

```tsx
// src/features/leave-requests/LeaveRequestForm.tsx
export function LeaveRequestForm() {
  return (
    <form data-testid="leave-form">
      <input
        type="date"
        data-testid="start-date"
        {...register('startDate')}
      />
      <input
        type="date"
        data-testid="end-date"
        {...register('endDate')}
      />
      <button type="submit" data-testid="submit-btn">
        Submit Request
      </button>
    </form>
  )
}
```

**Playwright test:**
```typescript
await page.fill('[data-testid="start-date"]', '2026-03-01')
await page.fill('[data-testid="end-date"]', '2026-03-03')
await page.click('[data-testid="submit-btn"]')
```

**Contract:**
```yaml
# GitHub issue specifies required testids
data-testid Requirements:
  - leave-form: Form container
  - start-date: Start date input
  - end-date: End date input
  - submit-btn: Submit button
  - success-msg: Success message after submit
```

---

## Journey Testing Strategy

### Critical Journeys (MUST Pass)

Test **every time** before merge:

```bash
npm run test:e2e -- --grep @critical

# Runs only critical journeys
# Blocks PR if any fail
```

**Example critical journeys:**
- User signup/login
- Core business workflows (request leave, approve leave)
- Payment/billing flows

### Important Journeys (SHOULD Pass)

Test **before release**, warn if failing:

```bash
npm run test:e2e -- --grep @important

# Runs important journeys
# Warns if failing, but allows merge
```

**Example important journeys:**
- Profile updates
- Settings changes
- Reporting/analytics

### Future Journeys (Aspirational)

Test **occasionally**, no blocking:

```bash
npm run test:e2e -- --grep @future

# Runs future journeys
# Informational only
```

**Example future journeys:**
- Experimental features
- Nice-to-have integrations (WhatsApp notifications)

---

## The Compiler Analogy for Journeys

| TypeScript | Specflow Journeys |
|------------|-------------------|
| Type definitions | Journey contract YAML |
| `tsc --noEmit` | `npm run test:e2e` |
| "Type 'string' not assignable to 'number'" | "Timeout waiting for [data-testid="success-msg"]" |
| Build blocked until fixed | PR blocked until fixed |

**Same enforcement model. Different domain.**

---

## Time Comparison: Manual vs Specflow

### Writing E2E Tests Manually

```
1. Read feature spec (10 min)
2. Identify test scenarios (15 min)
3. Write Playwright test (30 min)
4. Debug selectors (20 min)
5. Add database assertions (15 min)
Total: 90 minutes
```

### Specflow Agent (playwright-from-specflow)

```
1. Read journey contract (auto-generated)
2. Generate Playwright test (1 min)
3. Map data-testid selectors (from issue)
4. Add preconditions/postconditions (30s)
Total: 2 minutes
```

**45x faster.**

---

## Common Questions

### "Do I need journeys for every feature?"

**For critical features: Yes.** Journeys define when the feature is "done."

**For minor features: Maybe.** If the feature is self-contained, a feature contract may be enough.

**Rule of thumb:**
- User-facing feature → Journey required
- Internal refactor → Feature contract sufficient

### "Can I have multiple journeys per feature?"

**Absolutely.** Most features have 2-3 journeys:

**Example: Leave management**
- J-STAFF-REQUEST-LEAVE (critical)
- J-MANAGER-APPROVE-LEAVE (critical)
- J-STAFF-CHECK-BALANCE (important)

### "What if a journey is flaky?"

**Fix it.** Flaky tests are worse than no tests.

**Common causes:**
- Missing `await` (asynchronous timing issues)
- No data-testid selectors (unstable selectors)
- Missing preconditions (test depends on previous state)

**Specflow guideline:**
- Journeys MUST be deterministic
- Flaky journeys downgraded to "important" or "future" until fixed

### "Can journeys span multiple repositories?"

**Not yet.** Specflow journeys are currently **intra-repo only**.

For multi-repo workflows, use API contract tests instead.

---

## Next Steps

- **[Agent System Deep Dive](/agent-system/)** — Learn how agents generate journeys
- **[Journey Testing Guide](/advanced/journey-testing/)** — Advanced E2E patterns
- **[Contract Schema Reference](/reference/contract-schema/)** — Full YAML format

---

## Compiler Analogy Reminder

> **TypeScript rejects `"hello" + 5` at compile time.**
>
> **Specflow rejects "Missing success message" at test time.**

Same principle. Different boundary.

**Journeys enforce end-to-end correctness.**
