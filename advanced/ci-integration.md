---
layout: default
title: CI/CD Integration
parent: Advanced
nav_order: 5
---

# CI/CD Integration
{: .no_toc }

Run Specflow contracts and journey tests in your CI/CD pipeline.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Fail-Fast Pipeline Pattern

The most important CI pattern for Specflow is **fail-fast**:

```
contracts → unit-tests → build → e2e-tests → journey-tests
    ↓           ↓         ↓          ↓            ↓
  FAIL?      SKIP      SKIP       SKIP         SKIP
```

**Why fail fast?**
- Contract tests are FAST (pattern matching, no browser)
- E2E tests are SLOW (browser automation)
- If contracts fail, skip expensive tests (save CI minutes)

---

## The Key: `needs: contract-tests`

This single line creates the fail-fast behavior:

```yaml
e2e-tests:
  needs: contract-tests  # ← E2E waits for contracts
```

**If contract-tests fails:**
- e2e-tests is **SKIPPED**
- journey-tests is **SKIPPED**
- You save 10-20 minutes of CI time

---

## Recommended GitHub Actions Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'

jobs:
  # ============================================
  # STEP 1: Contract Tests (FAIL FAST)
  # ============================================
  contract-tests:
    name: Contract Tests (Pattern Enforcement)
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v3
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run contract tests
        run: pnpm test -- contracts --passWithNoTests

  # ============================================
  # STEP 2: Unit Tests (parallel with contracts)
  # ============================================
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm test:coverage

  # ============================================
  # STEP 3: E2E Tests (WAITS FOR CONTRACTS)
  # ============================================
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: contract-tests  # ← KEY: Wait for contracts!

    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm test:e2e

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/

  # ============================================
  # STEP 4: Journey Tests (RELEASE GATING)
  # ============================================
  journey-tests:
    name: Journey Tests (RELEASE GATING)
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: contract-tests  # ← Wait for contracts

    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm test:e2e tests/e2e/journey_*.spec.ts

  # ============================================
  # STEP 5: Build
  # ============================================
  build:
    name: Build Application
    runs-on: ubuntu-latest
    timeout-minutes: 10
    needs: [unit-tests, contract-tests]

    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

---

## Hooks vs CI: Two Enforcement Layers

Specflow enforces contracts in **two places**:

| Layer | Where | When | Speed |
|-------|-------|------|-------|
| **Hooks** | Local (your machine) | Build, commit | Seconds |
| **CI** | Remote (GitHub) | PR, merge | Minutes |

### Why Both?

**Hooks** catch problems before you push:
- Instant feedback
- No wasted CI minutes
- You see issues immediately

**CI** catches problems after you push:
- Authoritative source of truth
- Clean environment
- Required for branch protection

### The Combined Workflow

```
Your Machine                           GitHub
────────────                           ──────

pnpm build
    ↓
[post-build HOOK]
    ↓
Runs journey tests locally ← Fast feedback
    ↓
git commit (#375)
    ↓
[post-commit HOOK]
    ↓
Extracts #375, runs journey
    ↓
git push
    └──────────────────────────────→ [contract-tests]
                                          ↓ needs:
                                     [e2e-tests]
                                          ↓ needs:
                                     [journey-tests]
                                          ↓
                                     Branch protection ✓
```

---

## Journey Test Criticality

Not all journeys are equally important:

| Criticality | Example | CI Behavior |
|-------------|---------|-------------|
| **Critical** | `J-USER-LOGIN` | BLOCK merge |
| **Important** | `J-EXPORT-REPORT` | WARN only |
| **Future** | `J-AI-ASSISTANT` | Skip |

### Mark Criticality in Contracts

```yaml
# docs/contracts/journey_user_login.yml
name: User Login Journey
type: journey
criticality: critical  # ← Affects CI gating

scenarios:
  - name: User logs in with email
    # ...
```

### CI Job for Criticality

```yaml
journey-tests:
  steps:
    - name: Run critical journeys
      run: pnpm test:e2e -- --grep "@critical"

    - name: Run important journeys (warn only)
      continue-on-error: true  # ← Doesn't block
      run: pnpm test:e2e -- --grep "@important"
```

---

## Branch Protection Setup

Configure GitHub to require passing checks:

```
Settings → Branches → main → Branch protection rules

Required status checks:
✅ contract-tests     ← Must pass
✅ journey-tests      ← Must pass
◻️ e2e-tests         ← Optional

✅ Require branches to be up to date
```

---

## Debugging CI Failures

### Contract Test Failure

```
❌ contract-tests failed

Error: CONTRACT VIOLATION: ARCH-001
File: src/routes/AdminPage.tsx
Issue: Protected route missing ProtectedRoute wrapper
```

**Fix:** Read the contract, add the wrapper.

### Journey Test Failure

```
❌ journey-tests failed

Error: J-STAFF-REQUEST-LEAVE scenario 2
File: tests/e2e/journey_staff_request_leave.spec.ts:45
Issue: Expected "Pending" but got "Error"
```

**Debug locally:**
```bash
pnpm test:e2e:ui tests/e2e/journey_staff_request_leave.spec.ts
```

---

## GitLab CI Example

```yaml
# .gitlab-ci.yml
stages:
  - contracts
  - test
  - build

contract-tests:
  stage: contracts
  script:
    - pnpm install --frozen-lockfile
    - pnpm test -- contracts

e2e-tests:
  stage: test
  needs: [contract-tests]  # ← Wait for contracts
  script:
    - pnpm install --frozen-lockfile
    - pnpm exec playwright install --with-deps
    - pnpm test:e2e

build:
  stage: build
  needs: [contract-tests]
  script:
    - pnpm build
```

---

## npm Scripts Integration

Add these to your `package.json`:

```json
{
  "scripts": {
    "test": "jest",
    "test:contracts": "jest src/__tests__/contracts/",
    "test:e2e": "playwright test",
    "test:e2e:journeys": "playwright test tests/e2e/journey_*.spec.ts",
    "ci:contracts": "npm run test:contracts -- --passWithNoTests",
    "ci:full": "npm run ci:contracts && npm test && npm run build"
  }
}
```

---

## Summary

| Principle | Implementation |
|-----------|----------------|
| Fail fast | `needs: contract-tests` |
| Local enforcement | Journey verification hooks |
| Remote enforcement | CI pipeline with branch protection |
| Criticality gating | `--grep "@critical"` |
| Save CI minutes | Skip E2E if contracts fail |

**The key insight:** Contracts are cheap to check. Check them first, skip expensive tests on failure.
