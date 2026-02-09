---
layout: default
title: "Production Readiness Gates"
parent: "Core Concepts"
nav_order: 5
permalink: /core-concepts/production-readiness/
---

# Production Readiness Gates
{: .no_toc }

Default contract rules that catch demo data, placeholder domains, and hardcoded IDs before they reach production.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why Production Readiness Gates?

Demo accounts, `localhost` URLs, and hardcoded UUIDs are the most common sources of production incidents in early-stage projects. These patterns are easy to introduce during development and easy to miss during review.

The production readiness contract (`production_readiness_defaults.yml`) catches these patterns with regex-based rules that run as part of your contract test suite. Like the [security and accessibility gates](/core-concepts/security-accessibility/), these are ready-to-use defaults that can be extended for your project.

---

## Rules

### PROD-001: No Demo or Mock Data in Production Code

Source code must not contain demo users, mock constants, or fake data references.

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| `/DEMO_USER/` | Demo user reference -- remove or gate behind feature flag |
| `/DEMO_PLAN/` | Demo plan reference -- remove or gate behind feature flag |
| `/MOCK_[A-Z_]{3,}/` | Mock constant -- use real values or environment variables |
| `/['"]demo[@.].*['"]/i` | Demo email or domain -- use environment-configured values |
| `/['"]test_user['"]/i` | Test user reference -- remove or gate behind feature flag |
| `/fake(User\|Data\|Account\|Email\|Name)/i` | Fake data reference -- use real values |
| `/seed(User\|Data\|Account)s?\s*=/i` | Seed data assignment -- move to seed scripts |

**Scope:** `src/**/*.{ts,js,tsx,jsx}`, excluding test files, fixtures, mocks, and seed directories.

**Example violation:**

```typescript
const DEFAULT_USER = DEMO_USER
const plan = DEMO_PLAN
const email = "demo@example.com"
```

**Example compliant:**

```typescript
const DEFAULT_USER = process.env.DEFAULT_USER_ID
const plan = await getPlanFromDatabase(userId)
const email = user.email
```

---

### PROD-002: Domain Allowlist Enforcement

Source code must not contain placeholder or development-only domain references.

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| `/https?:\/\/example\.com/` | Placeholder domain -- use real domain |
| `/https?:\/\/localhost[:\/"']/` | Localhost reference -- use environment variable |
| `/https?:\/\/127\.0\.0\.1/` | Loopback address -- use environment variable |
| `/https?:\/\/placeholder\./i` | Placeholder domain -- use real domain |
| `/https?:\/\/todo\./i` | TODO domain -- replace with actual domain |
| `/https?:\/\/changeme\./i` | Changeme domain -- replace with actual domain |

**Scope:** `src/**/*.{ts,js,tsx,jsx}`, excluding test files.

**Example violation:**

```typescript
const API_URL = "https://example.com/api"
const BASE_URL = "http://localhost:3000"
```

**Example compliant:**

```typescript
const API_URL = process.env.API_URL
const BASE_URL = process.env.NEXT_PUBLIC_BASE_URL
```

---

### PROD-003: No Hardcoded IDs in Production Code

Source code must not contain hardcoded UUIDs, user IDs, tenant IDs, or org IDs.

**Forbidden patterns:**

| Pattern | Message |
|---------|---------|
| `/['"][0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}['"]/` | Hardcoded UUID -- use configuration or database lookup |
| `/user_id\s*[:=]\s*['"][a-zA-Z0-9_-]{10,}['"]/` | Hardcoded user ID -- use authenticated session or lookup |
| `/tenant_id\s*[:=]\s*['"][a-zA-Z0-9_-]{10,}['"]/` | Hardcoded tenant ID -- use session context or lookup |
| `/org_id\s*[:=]\s*['"][a-zA-Z0-9_-]{10,}['"]/` | Hardcoded org ID -- use session context or lookup |

**Scope:** `src/**/*.{ts,js,tsx,jsx}`, excluding test files, fixtures, and migration directories.

**Example violation:**

```typescript
const adminUser = "550e8400-e29b-41d4-a716-446655440000"
const tenant_id = "acme_corp_12345"
```

**Example compliant:**

```typescript
const adminUser = await getAdminUser(orgId)
const tenant_id = session.tenantId
```

---

## Quick Reference

| ID | Rule | What It Catches |
|----|------|-----------------|
| PROD-001 | No demo/mock data | `DEMO_USER`, `MOCK_DATA`, `fakeAccount`, seed assignments |
| PROD-002 | Domain allowlist | `localhost`, `example.com`, `placeholder.dev` |
| PROD-003 | No hardcoded IDs | UUID literals, `user_id = "abc..."`, `tenant_id = "..."` |

---

## Installation

Copy the default contract template to your project:

```bash
cp Specflow/templates/contracts/production_readiness_defaults.yml docs/contracts/
```

Then run contract tests to verify:

```bash
npm test -- contracts
```

The contract expects a test file at `src/__tests__/contracts/production_readiness_defaults.test.ts`. You can generate this from the contract YAML using the `contract-test-generator` agent, or write it manually following the patterns in the [contract schema reference](/reference/contract-schema/).

---

## Compliance Checklist

Before editing production source files, check:

| Question | If Yes |
|----------|--------|
| Are you referencing a specific user, tenant, or org ID? | Use environment variables, database lookups, or session context. Never hardcode IDs. |
| Are you adding a URL or domain reference? | Use environment variables for base URLs. Verify the domain is real and not a placeholder. |
| Are you adding demo or test data? | Move to seed scripts, fixtures, or test files. Production code must not contain mock data. |

---

## Extending the Defaults

Add project-specific rules alongside the defaults using the same `PROD-xxx` prefix:

```yaml
# docs/contracts/production_readiness_custom.yml
contract_meta:
  id: production_readiness_custom
  version: 1
  covers_reqs:
    - PROD-004
    - PROD-005
  owner: "platform-team"

rules:
  non_negotiable:
    - id: PROD-004
      title: "No test Stripe keys in production code"
      scope:
        - "src/**/*.{ts,js}"
        - "!src/**/*.test.*"
      behavior:
        forbidden_patterns:
          - pattern: /sk_test_[A-Za-z0-9]{24,}/
            message: "Test Stripe secret key in production code"
          - pattern: /pk_test_[A-Za-z0-9]{24,}/
            message: "Test Stripe publishable key in production code"

    - id: PROD-005
      title: "No TODO_REPLACE markers"
      scope:
        - "src/**/*.{ts,js,tsx,jsx}"
      behavior:
        forbidden_patterns:
          - pattern: /TODO_REPLACE/
            message: "Unreplaced TODO marker in production code"
```

---

## Overriding Rules

Disable specific rules in `.specflow/config.json`:

```json
{
  "disabled_rules": ["PROD-003"],
  "rule_overrides": {
    "PROD-002": {
      "severity": "important",
      "note": "We allow localhost in development-only config files"
    }
  }
}
```

The override protocol requires explicit human approval. An LLM agent cannot override non-negotiable rules unless the user says:

```
override_contract: production_readiness_defaults
```

---

## Relationship to Other Gates

Production readiness gates complement the other default contract templates:

| Gate | Focus | Contract File |
|------|-------|---------------|
| **Security** (SEC-xxx) | Secrets, XSS, SQL injection, eval | `security_defaults.yml` |
| **Accessibility** (A11Y-xxx) | Alt text, aria-labels, tab order | `accessibility_defaults.yml` |
| **Production Readiness** (PROD-xxx) | Demo data, placeholder domains, hardcoded IDs | `production_readiness_defaults.yml` |

All three can be installed independently. Together they provide baseline coverage for the most common issues that reach production.

---

## Related Pages

- [Security & Accessibility Gates](/core-concepts/security-accessibility/) -- SEC and A11Y contract rules
- [What Are Contracts?](/core-concepts/contracts/) -- How contracts enforce architecture
- [Contract Schema](/reference/contract-schema/) -- Full YAML schema reference
- [Self-Healing Fix Loops](/advanced/self-healing/) -- Auto-fix for PROD violations
- [Post-Mortem Learning System](/advanced/learning-system/) -- How violations feed into agent learning

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
