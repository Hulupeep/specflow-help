---
layout: default
title: How Agents Work
parent: Core Concepts
nav_order: 2
permalink: /core-concepts/agents/
---

# How Agents Work Together
{: .no_toc }

23+ specialized LLM agents orchestrated by waves-controller.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Agent-First Philosophy

**Traditional workflow:**
```
Human writes spec → Human writes code → Human writes tests → Human reviews → Ship
```

**Time:** 4-6 hours per feature

**Specflow workflow:**
```
Human defines what matters → Agents generate everything → Tests enforce → Ship or stop
```

**Time:** 20-30 minutes per feature

**3-4x faster. <5% contract violation rate.**

---

## What is an Agent?

An **agent** is a specialized LLM-powered worker with:

1. **Specific trigger** (when to run)
2. **Clear inputs** (what it reads)
3. **Clear outputs** (what it generates)
4. **Quality gates** (how success is measured)

**Think of agents like build tools:**
- `tsc` compiles TypeScript → JavaScript
- `specflow-writer` compiles GitHub issues → YAML contracts
- `playwright-from-specflow` compiles YAML contracts → E2E tests

**Each agent does ONE thing well.**

---

## The Agents

### Orchestration Agent

| Agent | Role | When Used |
|-------|------|-----------|
| **waves-controller** | Orchestrates all other agents | User says "Execute waves" |

### Contract Generation Agents

| Agent | Role | When Used |
|-------|------|-----------|
| **specflow-writer** | Creates contracts from GitHub issues | Feature needs acceptance criteria |
| **contract-generator** | Generates contract YAML from specs | Manual contract creation needed |
| **contract-test-generator** | Creates contract tests (TypeScript) | Contract needs enforcement test |

### Implementation Agents

| Agent | Role | When Used |
|-------|------|-----------|
| **migration-builder** | Generates database migrations | Feature needs schema changes |
| **edge-function-builder** | Creates Supabase Edge Functions | Feature needs backend RPC |
| **frontend-builder** | Generates React components | Feature needs UI |

### Testing Agents

| Agent | Role | When Used |
|-------|------|-----------|
| **playwright-from-specflow** | Creates E2E tests from Gherkin | Journey contract exists |
| **journey-tester** | Creates cross-feature journey tests | Multiple features integrated |
| **test-runner** | Runs tests and reports failures | After code changes |
| **e2e-test-auditor** | Verifies E2E coverage for UI issues | Before release |

### Validation Agents

| Agent | Role | When Used |
|-------|------|-----------|
| **contract-validator** | Verifies implementation matches contracts | After feature implementation |
| **journey-enforcer** | Ensures critical journeys have coverage | Before release |

### Workflow Agents

| Agent | Role | When Used |
|-------|------|-----------|
| **ticket-closer** | Updates and closes GitHub issues | After feature complete |
| **dependency-mapper** | Calculates issue dependencies | Before wave execution |
| **board-auditor** | Analyzes GitHub project board health | Weekly/on-demand |
| **sprint-executor** | Executes all issues in a sprint | Sprint start |
| **specflow-uplifter** | Adds Specflow to legacy codebases | Mid-project adoption |

---

## How Agents Work Together

### Example: Issue #50 "Add Login"

**User action:**
```
Execute waves
```

**waves-controller orchestrates 8 phases:**

#### Phase 1: Discovery (30s)

```
waves-controller:
  → Fetch issue #50 from GitHub
  → Extract acceptance criteria:
    - User can log in with email/password
    - Invalid credentials show error
    - Successful login redirects to dashboard
  → Calculate dependencies: No blockers
  → Priority: Critical (has "auth" tag)
  → Add to Wave 1
```

#### Phase 2: Spawn Agents (Parallel)

```
waves-controller:
  → Spawn specflow-writer for issue #50
  → Spawn migration-builder for issue #50
  → Spawn edge-function-builder for issue #50
```

#### Phase 3: Agents Work (2-3 minutes)

**specflow-writer:**
```
Input:
  - Issue #50 description
  - Acceptance criteria from issue body

Process:
  1. Extract Gherkin scenarios from acceptance criteria
  2. Identify invariants (e.g., "Passwords MUST be verified")
  3. Generate feature contract (feature_authentication.yml)
  4. Generate journey contract (journey_user_login.yml)

Output:
  ✓ docs/contracts/feature_authentication.yml (4 invariants)
  ✓ docs/contracts/journey_user_login.yml (3 steps)
```

**migration-builder:**
```
Input:
  - Issue #50 description
  - Existing database schema (from supabase/migrations/)

Process:
  1. Analyze if new tables needed
  2. Check for RLS policy requirements
  3. Generate migration SQL
  4. Add indexes for performance

Output:
  ✓ supabase/migrations/20260203_add_login_attempts.sql
  ✓ Migration adds rate limiting table
```

**edge-function-builder:**
```
Input:
  - Issue #50 description
  - Contracts from specflow-writer

Process:
  1. Determine if backend RPC needed
  2. Generate Edge Function (login RPC)
  3. Add password verification logic (bcrypt)
  4. Return JWT token on success

Output:
  ✓ supabase/functions/login/index.ts
  ✓ RPC handles login logic
```

#### Phase 4: Spawn Test Agents (Parallel)

```
waves-controller:
  → Spawn contract-test-generator
  → Spawn playwright-from-specflow
```

**contract-test-generator:**
```
Input:
  - docs/contracts/feature_authentication.yml

Process:
  1. Read invariants (AUTH-001 to AUTH-004)
  2. Generate contract tests (TypeScript)
  3. Add to test suite

Output:
  ✓ src/__tests__/contracts/auth.test.ts
  ✓ Tests scan for bcrypt usage, JWT expiry, rate limiting
```

**playwright-from-specflow:**
```
Input:
  - docs/contracts/journey_user_login.yml
  - data-testid selectors from issue #50

Process:
  1. Read journey steps
  2. Generate Playwright test
  3. Map preconditions → beforeEach
  4. Map postconditions → afterEach

Output:
  ✓ tests/e2e/journey_user_login.spec.ts
  ✓ Test covers all 3 acceptance criteria
```

#### Phase 5: Generate Frontend (If Needed)

```
waves-controller:
  → Spawn frontend-builder

frontend-builder:
  Input:
    - Contracts from specflow-writer
    - data-testid selectors from issue

  Process:
    1. Generate LoginPage component
    2. Add form with email/password inputs
    3. Integrate with Edge Function
    4. Handle success/error states

  Output:
    ✓ src/features/auth/LoginPage.tsx
    ✓ Component has data-testid attributes
```

#### Phase 6: Validation

```
waves-controller:
  → Spawn contract-validator
  → Spawn test-runner

contract-validator:
  ✓ All invariants satisfied:
    - AUTH-001: bcrypt.compare() used in login RPC
    - AUTH-002: JWT expires after 24h
    - AUTH-003: Rate limiting table exists
    - AUTH-004: RLS enabled on users table

test-runner:
  Running contract tests...
    ✓ AUTH-001: Password verification (15ms)
    ✓ AUTH-002: JWT expiry (12ms)
    ✓ AUTH-003: Rate limiting (18ms)
    ✓ AUTH-004: RLS policies (22ms)

  Running E2E tests...
    ✓ J-USER-LOGIN: Valid credentials (2.1s)
    ✓ J-USER-LOGIN: Invalid credentials show error (1.8s)
    ✓ J-USER-LOGIN: Redirect to dashboard on success (2.3s)

  All tests passed ✓
```

#### Phase 7: Close Issue

```
waves-controller:
  → Spawn ticket-closer

ticket-closer:
  ✓ Added comment to issue #50:
    - Contracts created: feature_authentication.yml, journey_user_login.yml
    - Tests created: 4 contract tests, 3 E2E tests
    - All tests passing
    - Implementation complete

  ✓ Closed issue #50
  ✓ Labeled with "specflow-verified"
```

#### Phase 8: Reporting

```
waves-controller:
  🎉 WAVE 1 COMPLETE

  Summary:
    Duration: 4m 12s
    Issues processed: 1
    Issues closed: 1
    Contracts created: 2
    Migrations generated: 1
    Edge Functions created: 1
    Frontend components: 1
    Contract tests: 4
    E2E tests: 3
    All tests passing: ✓

  Manual workflow estimate: 4-6 hours
  Time saved: ~5 hours
```

**Total time: 4 minutes vs 5 hours manual.**

---

## Agent Coordination Patterns

### Sequential Dependencies

Some agents must run in order:

```
specflow-writer → contract-test-generator
                → playwright-from-specflow

migration-builder → (waits for DB migration) → edge-function-builder
```

### Parallel Execution

Independent agents run simultaneously:

```
Wave 1 (Parallel):
  Issue #50: specflow-writer + migration-builder + edge-function-builder
  Issue #51: specflow-writer + frontend-builder
  Issue #52: specflow-writer + contract-test-generator
```

**DPAO Methodology:** Discovery → Parallel → Analysis → Orchestration

### Quality Gates

Every agent has success criteria:

| Agent | Quality Gate |
|-------|--------------|
| specflow-writer | Contracts have valid YAML, map to acceptance criteria |
| migration-builder | Migration has RLS policies, passes `supabase db lint` |
| playwright-from-specflow | Tests use data-testid selectors, cover all steps |
| test-runner | All tests pass (contract + E2E) |
| contract-validator | Implementation satisfies all invariants |

**If any gate fails, the wave is blocked.**

---

## The Compiler Analogy for Agents

| Build Tool | Specflow Agent |
|------------|----------------|
| `tsc` (TypeScript → JavaScript) | `specflow-writer` (Issues → Contracts) |
| `webpack` (Modules → Bundle) | `migration-builder` (Schema → SQL) |
| `jest` (Code → Test results) | `test-runner` (Code → Pass/Fail) |
| `eslint` (Code → Lint errors) | `contract-validator` (Code → Violations) |

**Each agent is a specialized compiler for a specific domain.**

---

## Time Savings Breakdown

### Issue #50: Add Login (Real Example)

| Task | Manual Time | Agent Time | Agent Used |
|------|-------------|------------|------------|
| Write acceptance criteria | 15 min | 2 min (auto-extracted) | specflow-writer |
| Create contracts | 30 min | 1 min | specflow-writer |
| Design DB schema | 20 min | 30s | migration-builder |
| Write migration | 15 min | 30s | migration-builder |
| Implement login RPC | 60 min | 2 min | edge-function-builder |
| Create login UI | 45 min | 2 min | frontend-builder |
| Write E2E tests | 40 min | 1 min | playwright-from-specflow |
| Manual testing | 30 min | 0 min (automated) | test-runner |
| **Total** | **4h 15m** | **20-30m** | **waves-controller** |

**~85% time reduction.**

---

## How to Invoke Agents

### Method 1: Let waves-controller Orchestrate (Recommended)

```
Execute waves
```

**waves-controller decides which agents to spawn based on issue analysis.**

### Method 2: Invoke Individual Agents (Advanced)

```
Read scripts/agents/specflow-writer.md, then create contracts for issue #50
```

**Useful for:**
- Testing a single agent
- Debugging agent output
- Custom workflows

### Method 3: Sequential Agent Pipeline (Manual Control)

```
1. Read scripts/agents/specflow-writer.md, create contracts for #50
2. Read scripts/agents/migration-builder.md, generate migrations for #50
3. Read scripts/agents/playwright-from-specflow.md, create tests for #50
```

**Slower than waves-controller, but gives full control.**

---

## Agent Prompts (How They Work Internally)

Each agent is defined by a **prompt file** in `scripts/agents/*.md`.

**Example:** `scripts/agents/specflow-writer.md`

```markdown
# Specflow-Writer Agent

## Role
Create Specflow contracts from GitHub issues.

## Trigger
User says "create contracts" OR waves-controller spawns for issue analysis.

## Inputs
1. GitHub issue URL or number
2. Issue description and acceptance criteria
3. Existing contracts in docs/contracts/

## Process
1. Read issue body
2. Extract acceptance criteria (Gherkin preferred)
3. Identify invariants (rules that must hold)
4. Generate feature contract YAML
5. Generate journey contract YAML (if workflow described)
6. Validate YAML against schema

## Outputs
- docs/contracts/feature_<name>.yml
- docs/contracts/journey_<name>.yml (if applicable)
- GitHub comment on issue with contract summary

## Quality Gates
- YAML validates against CONTRACT-SCHEMA.md
- All acceptance criteria mapped to contract
- Invariants have enforcement method specified
```

**The prompt acts as the agent's instructions.**

---

## Common Questions

### "Can I customize agent behavior?"

**Yes.** Edit the prompt file in `scripts/agents/*.md`.

Example: Make specflow-writer always generate contract tests:
```markdown
## Process
...
6. Generate contract test (TypeScript) for each invariant
```

### "What if an agent fails?"

**waves-controller retries or reports failure:**

```
⚠️ Agent Failed: migration-builder

Issue: #50
Error: "No database changes detected in issue description"

Recommendation:
  - Update issue #50 with database requirements
  - OR skip migration-builder for this issue
```

### "Can I add new agents?"

**Yes.** Create a new prompt file following the template:

```markdown
# my-custom-agent

## Role
[What this agent does]

## Trigger
[When to run]

## Inputs
[What it reads]

## Process
[Step-by-step instructions]

## Outputs
[What it generates]

## Quality Gates
[How to verify success]
```

Then reference it in waves-controller or invoke manually.

---

## Next Steps

- **[waves-controller Deep Dive](/agent-system/waves-controller/)** — Understand the 8-phase orchestration
- **[Agent Reference](/agent-system/agent-reference/)** — Complete list of all 23+ agents
- **[DPAO Methodology](/agent-system/dpao/)** — How parallel execution works

---

## Compiler Analogy Reminder

> **TypeScript has `tsc`. Webpack has `webpack-cli`. Specflow has `waves-controller`.**

All orchestrate specialized tools to transform inputs into verified outputs.

**Agents are just build tools for architecture.**
