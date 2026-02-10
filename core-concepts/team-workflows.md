---
layout: default
title: Team Workflows
parent: Core Concepts
nav_order: 4
permalink: /core-concepts/team-workflows/
---

# Team Workflows
{: .no_toc }

How product designers, tech leads, and developers collaborate using Specflow.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Team Problem

Specflow works great for solo developers. But teams have different roles:

| Role | Writes | Cares About |
|------|--------|-------------|
| **Tech Lead** | YAML contracts | Architectural invariants |
| **Product Designer** | CSV journeys | User workflows, acceptance criteria |
| **Developer** | Code | Passing contracts + journey tests |
| **CI** | Nothing | Catching what local hooks miss |

**The challenge:** Product designers shouldn't need to write YAML. Developers need contracts enforced automatically. CI needs to catch violations that slip through.

---

## CSV Journeys for Product Designers

Product designers define user journeys in a simple CSV format (works in Google Sheets, Excel, or any text editor):

```csv
journey_id,journey_name,step,user_does,system_shows,critical,owner,notes
J-SIGNUP-FLOW,User Signup,1,Clicks "Sign Up",Shows registration form,yes,@alice,
J-SIGNUP-FLOW,User Signup,2,Fills email + password,Validates in real-time,yes,@alice,
J-SIGNUP-FLOW,User Signup,3,Clicks submit,Shows success + redirect to dashboard,yes,@alice,Must receive welcome email
```

This is then compiled into YAML contracts and Playwright test stubs automatically.

See the [CSV Journey Schema Reference](/reference/csv-journey-schema/) for the full column specification.

---

## The Compile Pipeline

```
Designer authors journeys.csv
  |
  v
npm run compile:journeys -- journeys.csv
  |
  +---> docs/contracts/journey_*.yml    (one per journey_id)
  +---> tests/e2e/journey_*.spec.ts     (Playwright stubs per journey_id)
```

### Running the Compiler

```bash
# Compile a CSV file into contracts + test stubs
npm run compile:journeys -- path/to/journeys.csv

# Example output:
# Compiled 4 journey(s) from path/to/journeys.csv
#   Contract: docs/contracts/journey_signup_flow.yml
#   Contract: docs/contracts/journey_login_flow.yml
#   Test:     tests/e2e/journey_signup_flow.spec.ts
#   Test:     tests/e2e/journey_login_flow.spec.ts
```

The compiler is **idempotent** — running it twice on the same CSV produces identical output (overwrites, doesn't append).

### What Gets Generated

**YAML Contract** (`docs/contracts/journey_signup_flow.yml`):

```yaml
journey_meta:
  id: J-SIGNUP-FLOW
  from_spec: "journeys.csv"
  covers_reqs: []
  type: "e2e"
  dod_criticality: critical
  status: not_tested
  last_verified: "2026-02-10"
  owner: "@alice"

preconditions:
  - description: "None - journey starts from blank state"
    setup_hint: null

steps:
  - step: 1
    name: "Clicks \"Sign Up\""
    expected:
      - type: "element_visible"
        description: Shows registration form
  - step: 2
    name: "Fills email + password"
    expected:
      - type: "element_visible"
        description: Validates in real-time
  - step: 3
    name: "Clicks submit"
    expected:
      - type: "element_visible"
        description: "Shows success + redirect to dashboard"

acceptance_criteria:
  - Must receive welcome email

test_hooks:
  e2e_test_file: "tests/e2e/journey_signup_flow.spec.ts"
```

**Playwright Stub** (`tests/e2e/journey_signup_flow.spec.ts`):

```typescript
import { test, expect } from '@playwright/test';

test.describe('J-SIGNUP-FLOW: User Signup', () => {
  test('Step 1: Clicks "Sign Up"', async ({ page }) => {
    // TODO: Implement
    // User does: Clicks "Sign Up"
    // System shows: Shows registration form
  });

  test('Step 2: Fills email + password', async ({ page }) => {
    // TODO: Implement
    // User does: Fills email + password
    // System shows: Validates in real-time
  });

  test('Step 3: Clicks submit', async ({ page }) => {
    // TODO: Implement
    // User does: Clicks submit
    // System shows: Shows success + redirect to dashboard
  });
});
```

---

## Local Enforcement Flow

```
Designer authors journeys.csv (Google Sheets / Excel / text editor)
  --> Runs: npm run compile:journeys -- journeys.csv
  --> Commits: journeys.csv + docs/contracts/journey_*.yml + tests/e2e/journey_*.spec.ts
  --> Pushes to GitHub

Developer pulls from GitHub
  --> git pull (gets new/updated contracts + journey tests)
  --> Makes code changes
  --> Runs: npm run build (triggers post-build hook)
  --> Hook runs contract tests automatically
  --> If contracts fail --> build blocked, dev must fix
  --> If contracts pass --> commit with issue reference (#123)
  --> Push --> PR created --> CI runs specflow-compliance.yml
  --> PR blocked if critical violations found

Bob pushes directly to main without running Specflow
  --> Post-merge audit fires on push to main
  --> Scans changed files against contract patterns
  --> Opens GitHub issue: "Contract violation by @bob in commit abc123"
  --> Team sees violation, Bob fixes it
```

---

## CI Enforcement

Specflow provides two GitHub Action templates for CI enforcement.

### PR Compliance Reporter

Runs on every pull request. Posts a comment with pass/fail status and owner attribution per contract.

```yaml
# .github/workflows/specflow-compliance.yml
# Triggers on: pull_request → main
# Does: Runs contract tests → Posts PR comment → Sets commit status
```

If a **critical** contract fails, the PR is blocked.

### Post-Merge Audit

Runs on every push to main. The safety net for direct pushes that bypass PRs.

```yaml
# .github/workflows/specflow-audit.yml
# Triggers on: push → main
# Does: Diffs changed files → Runs contract tests → Opens issue on violations
```

If violations are found, a GitHub issue is created and tagged with the contract owner.

Both templates are available at `templates/ci/` in the Specflow repository.

---

## Role-Based Workflow Summary

### Product Designer

1. Define journeys in CSV (Google Sheets, Excel, text editor)
2. Run `npm run compile:journeys -- journeys.csv`
3. Commit the CSV + generated contracts + test stubs
4. Push to GitHub

### Tech Lead

1. Write YAML contracts manually for architectural invariants
2. Review generated journey contracts from CSV compilations
3. Set `dod_criticality` levels for each contract
4. Monitor CI compliance reports

### Developer

1. Pull latest contracts from git
2. Write code that satisfies contracts
3. Run `npm test` to verify locally
4. Commit with issue reference (`#123`)
5. Push — CI handles the rest

---

## Next Steps

- **[CSV Journey Schema](/reference/csv-journey-schema/)** — Full column reference and validation rules
- **[What Are Journeys?](/core-concepts/journeys/)** — Deep dive into journey concepts
- **[Contract Schema Reference](/reference/contract-schema/)** — Full YAML format
- **[Getting Started](/getting-started/)** — Set up Specflow in your project

---

**Specflow makes team workflows enforceable, not just documented.**
