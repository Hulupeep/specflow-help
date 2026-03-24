---
layout: default
title: "Drift Prevention"
parent: "Advanced"
nav_order: 7
permalink: /advanced/drift-prevention/
---

# Drift Prevention
{: .no_toc }

Three enforcement layers that catch contract violations at edit time, merge time, and deploy time — so drift never reaches production.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Problem: Specs Rot

Every project starts with good intentions. The spec says "all endpoints require auth." Six months later, three endpoints don't. Nobody noticed because:

1. **The spec is a document, not a check.** It describes what should be true but doesn't verify it.
2. **CI runs tests, but tests don't cover the spec.** Unit tests pass. Integration tests pass. The auth gap isn't tested because nobody wrote a test for it.
3. **The gap is invisible until production.** A customer reports they can access data without logging in. The spec said this shouldn't happen. But the spec couldn't enforce itself.

This is **drift** — the growing distance between what the spec says and what the code does.

---

## Three Layers Stop Drift

Specflow prevents drift with three enforcement layers. Each catches violations at a different stage, with different speed and coverage tradeoffs.

```
LAYER 1: Edit-time hooks (instant, narrow)
    ↓ catches pattern violations while agent is coding
LAYER 2: CI contract tests (minutes, broad)
    ↓ catches pattern violations + journey regressions on push
LAYER 3: Deploy gate (after deploy, production-verified)
    ↓ catches infrastructure failures + route availability
```

**Why three layers?**

| Layer | Speed | Catches | Misses |
|-------|-------|---------|--------|
| Edit-time hooks | Instant | Pattern violations in the file being edited | Violations in other files, cross-file dependencies |
| CI contract tests | 1-5 minutes | All pattern violations across the codebase | Runtime behavior, deployment failures |
| Deploy gate | After deploy | Route availability, both-planes-live, E2E regression | Nothing — this is the final check |

If you only have CI, violations land in PRs and get fixed reactively. If you only have hooks, you miss violations in files that weren't edited. If you only have deploy gates, you catch everything but only after the code is live.

**All three together:** the agent gets stopped before writing the violation, CI catches anything the hook missed, and the deploy gate verifies the live system works end-to-end.

---

## Layer 1: Edit-Time Hooks

Claude Code hooks fire during code editing. They read the file the agent just wrote or edited and check it against contract rules. Violations block the edit with an explanation.

### How It Works

```
Agent edits src/engagement/engagement.controller.ts
  ↓
PostToolUse hook fires
  ↓
Hook script reads the file
  ↓
Checks: does every @Get/@Post/@Patch/@Delete method have @RequireOrganization?
  ↓
VIOLATION: "getReactions" has no @RequireOrganization declaration
  ↓
Agent sees: "GUARD-001 violation: add @RequireOrganization() or @RequireOrganization(false)"
  ↓
Agent fixes the file before moving on
```

### Setting Up a Hook

Hooks live in `.claude/hooks/` or `.claude/settings.json` in your project.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "bash .claude/hooks/guard-check.sh \"$CLAUDE_FILE_PATH\"",
        "description": "GUARD-001: Check @RequireOrganization on controller endpoints"
      }
    ]
  }
}
```

The hook script is a simple bash file that greps the edited file for violations:

```bash
#!/bin/bash
FILE="$1"

# Only check controller files
[[ "$FILE" != *".controller.ts" ]] && exit 0

# Find route methods without @RequireOrganization
if grep -P '@(Get|Post|Patch|Delete|Put)\s*\(' "$FILE" | grep -v '@RequireOrganization' > /dev/null 2>&1; then
  echo "GUARD-001 VIOLATION: $FILE has route methods without @RequireOrganization"
  echo "Every endpoint must declare @RequireOrganization() or @RequireOrganization(false)"
  exit 1
fi

exit 0
```

### When to Use Hooks vs CI

| Use hooks for | Use CI for |
|--------------|------------|
| Single-file pattern checks (missing decorator, forbidden import) | Cross-file analysis (every API button has a matching E2E test) |
| Rules the agent can fix immediately | Rules that require running the full test suite |
| Violations that are embarrassing if they reach a PR | Violations that need broader context to detect |

---

## Layer 2: CI Contract Tests

Contract tests run as part of `npm test` in your CI pipeline. They scan source code for `forbidden_patterns` defined in contract YAML files. If a pattern is found, the test fails and the PR is blocked.

### How It Works

```
Developer pushes to main
  ↓
GitHub Actions runs npm test
  ↓
Contract tests scan all controller files
  ↓
guard_compatibility.test.ts finds: engagement.controller.ts line 42
  @Get('processed-emails/:id/reactions') has no @RequireOrganization
  ↓
TEST FAILS → PR blocked
  ↓
Developer reads: "GUARD-001: Every endpoint must declare @RequireOrganization"
  ↓
Developer adds the missing decorator → pushes → tests pass → PR merges
```

### Anatomy of a Contract Test

Contract tests are **static analysis**, not runtime tests. They read source files and pattern-match against rules.

```typescript
// src/__tests__/contracts/guard_compatibility.test.ts
import * as fs from 'fs';
import * as path from 'path';
import { glob } from 'glob';

describe('GUARD-001: @RequireOrganization on all endpoints', () => {
  const controllers = glob.sync('src/**/*.controller.ts');

  controllers.forEach(file => {
    it(`${file} — all route methods have @RequireOrganization`, () => {
      const content = fs.readFileSync(file, 'utf-8');
      const routePattern = /@(Get|Post|Patch|Delete|Put)\s*\(/g;
      let match;

      while ((match = routePattern.exec(content)) !== null) {
        const before = content.substring(
          Math.max(0, match.index - 200),
          match.index
        );
        const hasDeclaration = /@RequireOrganization\s*\(/.test(before);
        expect(hasDeclaration).toBe(true);
      }
    });
  });
});
```

### From Contract YAML to Test File

Every contract YAML has a `test_hooks.tests` field pointing to the test file:

```yaml
test_hooks:
  tests:
    - file: "src/__tests__/contracts/guard_compatibility.test.ts"
      description: "Verifies all controller endpoints have explicit @RequireOrganization"
```

The `test-generator` agent can create these test files from the YAML:

```
"Run test-generator for guard_compatibility_defaults.yml"
```

Or create them manually following the pattern above.

---

## Layer 3: Deploy Gate

The deploy gate verifies the live system after deployment. It catches failures that static analysis and local E2E cannot: failed deploys, unregistered routes, stale builds, infrastructure misconfigurations.

### How It Works

```
Backend deployed to Railway
  ↓
Deploy gate checks: do new endpoints return 401 (auth required)?
  curl https://api.example.com/new-endpoint → 401 ✓ (route exists)
  curl https://api.example.com/new-endpoint → 404 ✗ (deploy failed)
  ↓
Frontend deployed to Vercel
  ↓
Full Playwright E2E runs against production
  ↓
167 passed, 0 failed → deploy verified
```

### The Two-Plane Rule

When your architecture has separate frontend and backend deploys, the deploy sequence matters:

```
1. Deploy backend
2. Verify new endpoints return 401 (not 404)
3. Deploy frontend
4. Run E2E against production
5. Report results on tickets
```

**Never deploy frontend first** when features span both planes. If you do, users see broken UI calling endpoints that don't exist.

### Deploy Gate Contract

```yaml
# docs/contracts/deploy_gate_defaults.yml
rules:
  non_negotiable:
    - id: DEPLOY-002
      title: "Backend deploys before frontend"
      behavior:
        description: |
          When a feature spans both planes, deploy backend first.
          Verify new routes return 401 (auth required), not 404 (not found).
          Then deploy frontend.
```

---

## Real-World Example: The Hubduck Wave Execution

These three layers were designed after a real production incident during a 15-issue wave execution across two repos.

### What Happened

1. **15 issues executed in 5 waves** — security fixes, bug fixes, new features (engagement tracking, action classification, principal dashboard)
2. **All Playwright tests passed locally** — 167/167
3. **UAT revealed 3 bugs** that tests didn't catch:
   - `POST /acknowledge` returned **403** (OrganizationGuard default-deny)
   - `GET /reactions` returned **400** (same guard, different endpoint)
   - Backend hadn't deployed in **48 hours** (7 consecutive build failures, nobody noticed)

### What Each Layer Would Have Caught

| Bug | Layer 1 (hooks) | Layer 2 (CI) | Layer 3 (deploy gate) |
|-----|----------------|--------------|----------------------|
| 403 on acknowledge | ✅ Hook checks `@RequireOrganization` | ✅ Contract test scans controllers | ❌ Would still need to deploy first |
| 400 on reactions | ✅ Same hook | ✅ Same contract test | ❌ Same |
| Backend not deployed | ❌ Hooks don't check deploy status | ❌ CI doesn't check deploy status | ✅ Route availability check catches this |

### Contracts Created From This Incident

| Contract | ID | Layer | What it prevents |
|----------|-----|-------|-----------------|
| Guard Compatibility | GUARD-001..002 | Hook + CI | Missing `@RequireOrganization` on endpoints |
| Deploy Gate | DEPLOY-001..004 | Deploy gate | Frontend deployed before backend, stale builds |
| LLM Output | LLM-001..003 | CI | LLM type vocabulary mismatch, derived fields in prompts |
| E2E Action Coverage | E2E-ACT-001..002 | CI | Render-only tests that don't click buttons |

---

## How to Add a New Enforcement Rule

### Step 1: Write the contract YAML

```yaml
contract_meta:
  id: my_new_contract
  version: 1
  created_from_spec: "Post-mortem: [describe the incident that motivated this rule]"
  covers_reqs:
    - MY-001

rules:
  non_negotiable:
    - id: MY-001
      title: "Short description of the rule"
      scope:
        - "src/**/*.ts"
      behavior:
        forbidden_patterns:
          - pattern: /dangerous_pattern/
            message: "What's wrong and how to fix it"
```

### Step 2: Generate the test file

```
"Run test-generator for my_new_contract.yml"
```

Or manually create `src/__tests__/contracts/my_new_contract.test.ts` that scans source files for the forbidden patterns.

### Step 3: Add a hook (optional, recommended for critical rules)

Create `.claude/hooks/my-check.sh` that checks the pattern in the edited file. Add to `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "command": "bash .claude/hooks/my-check.sh \"$CLAUDE_FILE_PATH\""
    }]
  }
}
```

### Step 4: Update CONTRACT_INDEX.yml

```yaml
default_contracts:
  - id: MY-DEFAULTS
    file: my_new_contract.yml
    scope: backend
    description: "What this contract prevents"
```

### Step 5: Verify

```bash
npm test -- contracts              # CI test passes
# Edit a file with the violation
# Hook fires and blocks the edit
```

---

## The Drift Equation

```
Drift = (time since last check) × (number of unchecked changes)
```

- **No enforcement:** Drift accumulates silently until production breaks
- **CI only:** Drift is caught on push (hours to days of accumulation)
- **CI + hooks:** Drift is caught on edit (seconds of accumulation)
- **CI + hooks + deploy gate:** Drift is caught at every stage, including infrastructure

The goal isn't zero drift — it's **bounded drift**. Every check reduces the maximum distance between spec and code. Three layers bound it to seconds.
