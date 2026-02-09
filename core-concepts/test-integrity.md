---
layout: default
title: "Test Integrity Gates"
parent: "Core Concepts"
nav_order: 6
permalink: /core-concepts/test-integrity/
---

# Test Integrity Gates
{: .no_toc }

Default contract rules that prevent mocking in E2E tests, catch silent test anti-patterns, and flag placeholder assertions.
{: .fs-6 .fw-300 }

> No-mock philosophy from [forge](https://github.com/ikennaokpala/forge) by [Ikenna N. Okpala](https://github.com/ikennaokpala), adapted from Ikenna's Continuous Behavioral Verification work. Silent test anti-patterns from Specflow's own `e2e-test-auditor` agent, production-tested across 280+ issues.

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why Test Integrity Gates?

Tests that pass when they should fail are worse than no tests at all. They create false confidence. The test integrity contract (`test_integrity_defaults.yml`) catches the most common ways tests silently become useless:

- **Mocking in E2E tests** hides integration bugs and tests implementation details instead of behavior
- **Swallowed errors** make tests pass on network failures
- **Length-only assertions** verify counts without checking content
- **Placeholder markers** indicate tests that were never finished

These defaults are:
- **Ready to use** -- copy the template and run tests immediately
- **Configurable** -- toggle mocking rules per project, add allowed mock patterns
- **Non-overridable** (TEST-003 through TEST-005) -- silent anti-patterns are never acceptable

---

## TEST-001: No Mocking in E2E Tests

**Rule:** E2E tests must hit real services. Mocking in E2E tests defeats the purpose of end-to-end verification.

**Enabled by default:** Yes (configurable via `no_mock_in_e2e`)

**Scope:** `tests/e2e/**/*.{ts,js,spec.ts}` (excludes helpers and fixtures)

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| `jest.mock()` | jest.mock() in E2E test -- E2E tests must hit real services |
| `vi.mock()` | vi.mock() in E2E test -- E2E tests must hit real services |
| `sinon.stub()` / `sinon.mock()` / `sinon.fake()` | sinon mocking in E2E test -- E2E tests must hit real services |
| `nock()` | nock() in E2E test -- E2E tests must hit real services |
| `.mockImplementation()` | mockImplementation in E2E test -- E2E tests must hit real services |
| `.mockReturnValue()` | mockReturnValue in E2E test -- E2E tests must hit real services |
| `.mockResolvedValue()` | mockResolvedValue in E2E test -- E2E tests must hit real services |

**Example violation:**

```typescript
// tests/e2e/checkout.spec.ts
jest.mock('../services/stripe')
test('checkout completes', async () => { ... })
```

**Example compliant:**

```typescript
// tests/e2e/checkout.spec.ts
test('checkout completes against real API', async ({ page }) => {
  await page.goto('/checkout')
  // Test hits real Stripe test mode
})
```

---

## TEST-002: No Mocking in Journey Tests

**Rule:** Journey tests verify real user behavior across features. Mocking breaks the integration guarantee that journeys provide.

**Enabled by default:** Yes (configurable via `no_mock_in_journey`)

**Scope:** `tests/e2e/journey_*.spec.ts`, `tests/e2e/journeys/**/*.spec.ts`

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| Mock/stub/fake/spy imports | Mock import in journey test -- journeys verify real user behavior |
| `jest.mock()` / `vi.mock()` / `sinon.stub()` | Mocking framework used in journey test -- journeys must be real |

**Example violation:**

```typescript
// tests/e2e/journey_signup.spec.ts
vi.mock('@/lib/supabase')
```

**Example compliant:**

```typescript
// tests/e2e/journey_signup.spec.ts
test('J-SIGNUP: user completes registration', async ({ page }) => {
  // All calls hit real Supabase
})
```

---

## TEST-003: No Silent Test Anti-Patterns

**Rule:** Tests must not silently pass when they should fail. This rule is **not configurable** -- silent anti-patterns are never acceptable.

**Scope:** `tests/e2e/**/*.{ts,js,spec.ts}`, `src/__tests__/**/*.{ts,js}`

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| `.catch(() => false)` / `.catch(() => null)` | Swallowed error -- test will silently pass on failure |
| `test.skip(true)` | Unconditional skip -- use `test.fixme('Blocked by #XXX')` with tracking issue |
| `expect(x).toHaveLength(N)` alone | Suspicious test -- only checks array length, not content |
| `// placeholder` / `// will be enhanced` / `// todo...later` | Placeholder test -- implement real assertions or remove |

**Example violation:**

```typescript
test('loads data', async () => {
  const data = await fetch('/api').catch(() => false)
  expect(data).toBeTruthy() // passes even on network error
})
```

**Example compliant:**

```typescript
test('loads data', async () => {
  const response = await fetch('/api')
  expect(response.ok).toBe(true)
  const data = await response.json()
  expect(data.users).toEqual(expect.arrayContaining([
    expect.objectContaining({ name: expect.any(String) })
  ]))
})
```

---

## TEST-004: No Suspicious Test Patterns

**Rule:** Tests must make meaningful assertions that verify correctness, not just existence or count. This rule is **not configurable**.

**Scope:** `tests/e2e/**/*.{ts,js,spec.ts}`, `src/__tests__/**/*.{ts,js}`

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| `expect(x).toHaveLength(N)` (only assertion) | Suspicious test -- only checks array length, not content. Verify array contents with `toEqual` or `toContain`. |
| `expect(x).toBe(42)` (hardcoded multi-digit value) | Suspicious test -- hardcoded expected value with no visible setup. Ensure expected values come from test fixtures or documented constants. |
| `expect(x).toEqual([])` | Suspicious test -- asserts empty array. Verify this is intentional and not masking missing data. |
| `expect(x).toBeDefined()` (only assertion) | Weak assertion -- only checks existence, not correctness. Add specific value checks. |

**Example violation:**

```typescript
test('loads users', async () => {
  const users = await getUsers()
  expect(users).toHaveLength(3)  // Only checks count, not content
})
```

**Example compliant:**

```typescript
test('loads users', async () => {
  const users = await getUsers()
  expect(users).toEqual(expect.arrayContaining([
    expect.objectContaining({ name: 'Alice', role: 'admin' })
  ]))
})
```

---

## TEST-005: No Placeholder Test Markers

**Rule:** Tests must not contain comments or names indicating they are placeholders for real tests. This rule is **not configurable**.

**Scope:** `tests/e2e/**/*.{ts,js,spec.ts}`, `src/__tests__/**/*.{ts,js}`

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| `// placeholder` | Placeholder test -- implement real assertions or remove |
| `// will be enhanced` | Deferred test -- implement now or create a tracking issue |
| `// TODO: real test` | TODO marker for test -- implement the real test before merging |
| `// TODO: add assertions` / `// TODO: add more assertions` | Missing assertions -- add them before merging |
| `// FIXME: test` | FIXME in test -- resolve before merging |
| `test('...placeholder...')` | Test name indicates placeholder -- implement real test or remove |

**Example violation:**

```typescript
test('user registration', async () => {
  // placeholder -- will be enhanced later
  expect(true).toBe(true)
})
```

**Example compliant:**

```typescript
test('user registration', async () => {
  const response = await register({ email: 'test@example.com', password: 'secure123' })
  expect(response.status).toBe(201)
  expect(response.body.user.email).toBe('test@example.com')
})
```

---

## Quick Reference

| ID | Rule | Category | Configurable |
|----|------|----------|--------------|
| TEST-001 | No mocking in E2E tests | No-mock | Yes |
| TEST-002 | No mocking in journey tests | No-mock | Yes |
| TEST-003 | No silent test anti-patterns | Silent failures | No |
| TEST-004 | No suspicious test patterns | Weak assertions | No |
| TEST-005 | No placeholder test markers | Incomplete tests | No |

---

## Configuration

### Toggling Mock Rules

Override in `.specflow/config.json`:

```json
{
  "contract_defaults": {
    "test_integrity": {
      "no_mock_in_e2e": true,
      "no_mock_in_journey": true,
      "no_mock_in_unit": false,
      "allowed_mock_patterns": ["stripe", "twilio"]
    }
  }
}
```

| Setting | Default | Effect |
|---------|---------|--------|
| `no_mock_in_e2e` | `true` | TEST-001 enforced in E2E tests |
| `no_mock_in_journey` | `true` | TEST-002 enforced in journey tests |
| `no_mock_in_unit` | `false` | Unit tests may mock freely |
| `allowed_mock_patterns` | `[]` | Strings that exempt a mock match (e.g., `"stripe"` allows mocking Stripe in E2E if the match contains "stripe") |

### When to Use `allowed_mock_patterns`

Use for services where real calls have side effects that cannot be safely tested:
- Payment processors (Stripe, Paddle) -- real charges
- SMS/voice providers (Twilio) -- real messages sent
- Email providers -- real emails delivered

Add the service name to `allowed_mock_patterns` and the mock match will be exempted if it contains that string.

---

## Installation

Copy the default contract template to your project:

```bash
cp Specflow/templates/contracts/test_integrity_defaults.yml docs/contracts/
```

Then run contract tests to verify:

```bash
npm test -- contracts
```

The contract expects a test file at `src/__tests__/contracts/test_integrity_defaults.test.ts`. You can generate this from the contract YAML using the `contract-test-generator` agent, or write it manually following the patterns in the [contract schema reference](/reference/contract-schema/).

---

## Compliance Checklist

Before editing test files, check:

| Question | If Yes |
|----------|--------|
| Are you adding a mock/stub to an E2E or journey test? | STOP. E2E and journey tests must hit real services. Move the mock to unit tests. |
| Are you adding try/catch that swallows errors in tests? | Use `expect().rejects` or let the error propagate -- tests should fail visibly. |
| Are you skipping a test? | Use `test.fixme('Blocked by #XXX')` -- never `test.skip(true)`. |
| Does your test only check array length or existence? | Add content assertions -- verify values, not just counts. |
| Does your test contain placeholder comments or TODO markers? | Implement real assertions now. If not possible, create a tracking issue and use `test.fixme()`. |

---

## Related Pages

- [Security & Accessibility Gates](/core-concepts/security-accessibility/) -- SEC and A11Y contract rules
- [Production Readiness Gates](/core-concepts/production-readiness/) -- PROD-001 through PROD-003
- [What Are Contracts?](/core-concepts/contracts/) -- How contracts enforce architecture
- [Contract Schema](/reference/contract-schema/) -- Full YAML schema reference
- [Self-Healing Fix Loops](/advanced/self-healing/) -- Auto-fix for TEST violations

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
