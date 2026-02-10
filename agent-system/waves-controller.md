---
layout: default
title: waves-controller
parent: Agent System
nav_order: 1
permalink: /agent-system/waves-controller/
---

# waves-controller: The Orchestrator
{: .no_toc }

8-phase methodology for automated issue execution.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What is waves-controller?

**waves-controller** is the **orchestration agent** that coordinates all other Specflow agents. It implements the **DPAO methodology** (Discovery → Parallel → Analysis → Orchestration) to execute GitHub issues automatically.

**Think of it like a conductor:**
- TypeScript has `tsc` to orchestrate compilation
- Webpack has `webpack-cli` to orchestrate bundling
- Specflow has `waves-controller` to orchestrate issue execution

**One command executes everything:**
```
Execute waves
```

---

## The 8 Phases

waves-controller executes in 8 sequential phases:

| Phase | Name | Purpose | Duration |
|-------|------|---------|----------|
| 1 | **Discovery** | Analyze issues, calculate dependencies | 30s - 1m |
| 2 | **Wave Planning** | Group issues into waves | 10s |
| 3 | **Agent Spawning** | Launch specialized agents in parallel | 5s |
| 4 | **Parallel Execution** | Agents work simultaneously | 2-4m |
| 5 | **Validation** | Run contract + E2E tests | 1-3m |
| 6 | **Quality Gates** | Check if work meets standards | 30s |
| 7 | **Issue Closure** | Update and close GitHub issues | 30s |
| 8 | **Reporting** | Output summary and next wave | 10s |

**Total per wave:** 5-10 minutes (vs 4-6 hours manual)

---

## Mandatory Visualizations

waves-controller renders 5 ASCII visualizations at specific phases. These are the trust layer — they make every phase visible, every dependency explicit, every test mapped.

| Phase | Visualization | Purpose |
|-------|--------------|---------|
| Phase 1 (first wave) | EXECUTION TIMELINE | "Here's what's about to happen" |
| Phase 1 (every wave) | DEPENDENCY TREE | "Here's the execution order" |
| Phase 1 (every wave) | ENFORCEMENT MAP | "Here's what gets tested and how" |
| Phase 4 (during work) | PARALLEL AGENT MODEL | "Here's who's working on what now" |
| Phase 8 (wave end) | SPRINT SUMMARY TABLE | "Here's what this wave delivered" |
| Phase 8 (wave end) | EXECUTION TIMELINE (updated) | "Here's progress so far" |
| `/specflow status` | ALL five visualizations | On-demand full dashboard |

The **ENFORCEMENT MAP** is the key visualization — it answers "what exactly will be tested, by what mechanism, and when?" for every issue:

```
ENFORCEMENT MAP — Wave 3
═══════════════════════════════════════════════════════════════

 Issue #52: Billing Integration
 ├─ CONTRACT TESTS (build-time, pattern scan):
 │   ├─ SEC-001: No hardcoded Stripe keys     → src/billing/**
 │   ├─ SEC-002: No SQL concatenation          → src/billing/**
 │   ├─ BILL-001: Must use paymentMiddleware   → src/routes/billing*
 │   └─ BILL-002: Amounts must use Decimal     → src/billing/**
 │
 └─ PLAYWRIGHT TESTS (post-build, E2E):
     ├─ J-BILLING-CHECKOUT: User completes checkout flow
     ├─ J-BILLING-CANCEL: User cancels subscription
     └─ J-BILLING-INVOICE: User views invoice history

 TOTALS: 4 contract rules enforced | 3 journey tests | 1 issue
═══════════════════════════════════════════════════════════════
```

See `agents/waves-controller.md` in the Specflow repo for all 5 visualization templates.

---

## Phase 1: Discovery

**Goal:** Understand all open issues and their dependencies.

### What Happens

```bash
# Fetch all open issues
gh issue list -R your-org/your-repo --json number,title,body,labels

# For each issue:
1. Extract acceptance criteria (Gherkin if present)
2. Identify blockers (mentions "blocked by #X" or "depends on #Y")
3. Extract tags (labels like "critical", "backend", "frontend")
4. Extract data-testid requirements (from issue body)
```

### Output

```
🌊 PHASE 1: DISCOVERY

Found 12 open issues:

Issue #42: Add user authentication [critical] [backend] [frontend]
  Acceptance criteria: 4 (Gherkin format)
  Blockers: None
  Tags: critical, backend, frontend

Issue #43: Add profile page [important] [frontend]
  Acceptance criteria: 3
  Blockers: #42 (needs auth)
  Tags: important, frontend

Issue #44: Export to CSV [future] [backend]
  Acceptance criteria: 2
  Blockers: None
  Tags: future, backend

... (9 more issues)

✓ Discovery complete (42s)
```

### Dependency Graph

waves-controller builds a dependency graph:

```
#42 (auth) ─┬─> #43 (profile)
            ├─> #45 (settings)
            └─> #47 (dashboard)

#44 (CSV export) → (no dependencies)

#46 (reporting) ─> #44 (needs CSV)
```

---

## Phase 2: Wave Planning

**Goal:** Group issues into waves based on dependencies and priority.

### Wave Calculation Algorithm

```typescript
function calculateWaves(issues) {
  const waves = []
  let currentWave = []

  // Wave 1: All issues with no blockers
  for (const issue of issues) {
    if (issue.blockers.length === 0) {
      currentWave.push(issue)
    }
  }
  waves.push(currentWave)

  // Wave 2+: Issues whose blockers are in previous waves
  while (hasRemainingIssues()) {
    currentWave = []
    for (const issue of remainingIssues) {
      if (allBlockersInPreviousWaves(issue)) {
        currentWave.push(issue)
      }
    }
    waves.push(currentWave)
  }

  return waves
}
```

### Priority Scoring

Within each wave, issues are prioritized:

| Factor | Weight | Example |
|--------|--------|---------|
| **Severity label** | 10x | `critical` > `important` > `future` |
| **Blocks count** | 5x | Blocks 3 issues > blocks 1 issue |
| **Age** | 2x | Older issues prioritized |
| **Complexity** | 1x | Simpler issues first (if wave is large) |

**Formula:**
```
priority = (severity_weight * 10) + (blocks_count * 5) + (age_days * 2) - complexity
```

### Output

```
🌊 PHASE 2: WAVE PLANNING

Wave 1 (Parallel execution possible):
  #42: Add user authentication [critical] (priority: 105)
  #44: Export to CSV [future] (priority: 22)

Wave 2 (Blocked until Wave 1 completes):
  #43: Add profile page [important] (blocked by #42)
  #45: Add settings page [important] (blocked by #42)

Wave 3:
  #46: Add reporting dashboard [important] (blocked by #44)

Remaining: 7 issues (future waves)

✓ Wave planning complete (8s)
```

---

## Phase 2b: Contract Completeness Gate (MANDATORY)

**Goal:** Verify that Phase 2 actually produced ALL required artifacts — not just tickets.

```bash
node scripts/check-contract-completeness.mjs
```

This script cross-references `CONTRACT_INDEX.yml` against actual files on disk. If it finds gaps, it tells you exactly what to fix:

```
✗ Found 3 completeness issue(s):

  1. [ORPHAN_FILE] Journey file exists but is NOT in CONTRACT_INDEX: journey_user_login.yml

     HOW TO FIX:
     Add this entry to docs/contracts/CONTRACT_INDEX.yml under the contracts: section:

       - id: J-USER-LOGIN
         file: journey_user_login.yml
         status: active
         ...

  2. [MISSING_FILE] CONTRACT_INDEX lists J-SIGNUP → journey_signup.yml but file doesn't exist

     HOW TO FIX:
     Create the file: docs/contracts/journey_signup.yml
     Template: Copy an existing journey_*.yml and update journey_meta

  3. [COUNT_MISMATCH] metadata.total_journeys = 12 but found 14 journey entries

     HOW TO FIX:
     Open docs/contracts/CONTRACT_INDEX.yml
     Change: total_journeys: 14
```

**Quality Gate:** Exit code 0 = proceed. Exit code 1 = STOP, fix gaps, re-run.

Do NOT proceed to Phase 3 until this gate passes.

---

## Phase 3: Agent Spawning

**Goal:** Launch specialized agents for Wave 1 issues.

### Agent Selection Logic

For each issue, waves-controller determines which agents to spawn:

```typescript
function selectAgents(issue) {
  const agents = []

  // Always spawn specflow-writer (contracts)
  agents.push('specflow-writer')

  // If issue mentions database/schema
  if (issue.body.includes('table') || issue.body.includes('migration')) {
    agents.push('migration-builder')
  }

  // If issue mentions RPC/backend logic
  if (issue.body.includes('endpoint') || issue.body.includes('API')) {
    agents.push('edge-function-builder')
  }

  // If issue mentions UI
  if (issue.body.includes('page') || issue.body.includes('component')) {
    agents.push('frontend-builder')
  }

  // Always spawn test generators
  agents.push('contract-test-generator')
  agents.push('playwright-from-specflow')

  return agents
}
```

### Parallel Spawning

All agents for all Wave 1 issues spawn **simultaneously**:

```
🌊 PHASE 3: AGENT SPAWNING

Wave 1 (2 issues, 12 agents total):

Issue #42: Add user authentication
  → specflow-writer
  → migration-builder
  → edge-function-builder
  → frontend-builder
  → contract-test-generator
  → playwright-from-specflow

Issue #44: Export to CSV
  → specflow-writer
  → edge-function-builder
  → contract-test-generator
  → playwright-from-specflow

✓ 12 agents spawned (5s)
```

**DPAO in action:** Discovery complete, now **parallel execution**.

---

## Phase 4: Parallel Execution

**Goal:** Agents work simultaneously, coordinating via shared memory.

### Agent Coordination

Agents run concurrently via Claude Code's Task tool:

```javascript
// Single message spawns all agents
Task("specflow-writer for #42", "Create contracts...", "agent-1")
Task("migration-builder for #42", "Generate migrations...", "agent-2")
Task("edge-function-builder for #42", "Build RPC...", "agent-3")
Task("frontend-builder for #42", "Create LoginPage...", "agent-4")
// ... 8 more agents in parallel
```

### Progress Tracking

```
🌊 PHASE 4: PARALLEL EXECUTION

Wave 1 Progress:

Issue #42: Add user authentication
  ✓ specflow-writer: Complete (1m 12s)
    - Created feature_authentication.yml (4 invariants)
    - Created journey_user_login.yml
  ⚙ migration-builder: Working... (1m 30s)
  ⚙ edge-function-builder: Working... (2m 10s)
  ✓ frontend-builder: Complete (1m 45s)
    - Created LoginPage.tsx
  ✓ contract-test-generator: Complete (0m 45s)
  ⚙ playwright-from-specflow: Working... (1m 20s)

Issue #44: Export to CSV
  ✓ specflow-writer: Complete (0m 58s)
  ✓ edge-function-builder: Complete (1m 22s)
  ✓ contract-test-generator: Complete (0m 38s)
  ✓ playwright-from-specflow: Complete (0m 52s)

Overall: ████████░░ 80% (8/12 agents complete)
```

### Agent Outputs

Each agent reports what it generated:

```
specflow-writer (#42):
  ✓ docs/contracts/feature_authentication.yml
  ✓ docs/contracts/journey_user_login.yml

migration-builder (#42):
  ✓ supabase/migrations/20260203_create_users.sql
  ✓ Added RLS policies (3)

edge-function-builder (#42):
  ✓ supabase/functions/login/index.ts
  ✓ JWT token generation logic

frontend-builder (#42):
  ✓ src/features/auth/LoginPage.tsx
  ✓ data-testid attributes added (4)

playwright-from-specflow (#42):
  ✓ tests/e2e/journey_user_login.spec.ts
  ✓ 3 test steps from contract
```

---

## Phase 5: Validation

**Goal:** Verify all generated code satisfies contracts.

### Contract Tests

```bash
npm test -- contracts

# Output:
Running contract tests...

  ✓ AUTH-001: Passwords hashed with bcrypt (15ms)
  ✓ AUTH-002: JWT expires after 24h (12ms)
  ✓ AUTH-003: Rate limiting on /auth/* (18ms)
  ✓ AUTH-004: RLS enabled on users table (22ms)
  ✓ CSV-001: Export includes all required columns (10ms)

5 passed, 0 failed
```

### E2E Tests

```bash
npm run test:e2e

# Output:
Running E2E tests...

  ✓ J-USER-LOGIN: Valid credentials (2.1s)
  ✓ J-USER-LOGIN: Invalid credentials show error (1.8s)
  ✓ J-USER-LOGIN: Redirect to dashboard (2.3s)
  ✓ J-CSV-EXPORT: Export button generates file (3.2s)

4 passed, 0 failed
```

### Validation Report

```
🌊 PHASE 5: VALIDATION

Wave 1 Test Results:

Issue #42: Add user authentication
  Contract tests: ✓ 4/4 passed
  E2E tests: ✓ 3/3 passed
  Status: VERIFIED ✓

Issue #44: Export to CSV
  Contract tests: ✓ 1/1 passed
  E2E tests: ✓ 1/1 passed
  Status: VERIFIED ✓

Overall: ✓ All tests passing
```

---

## Phase 6: Quality Gates

**Goal:** Check if work meets release criteria.

### Quality Checks

| Check | Passing Criteria |
|-------|------------------|
| **Contract compliance** | All critical invariants satisfied |
| **Test coverage** | All acceptance criteria have tests |
| **Journey coverage** | All critical journeys passing |
| **Build success** | `npm run build` succeeds |
| **No regressions** | Existing tests still pass |

### Gate Evaluation

```
🌊 PHASE 6: QUALITY GATES

Wave 1 Quality Report:

Issue #42: Add user authentication
  ✓ Contract compliance: 4/4 invariants satisfied
  ✓ Test coverage: 100% (4/4 criteria tested)
  ✓ Journey coverage: 1/1 critical journey passing
  ✓ Build success: Production build OK
  ✓ No regressions: 127/127 existing tests pass

  Quality score: 100% (5/5 gates passed)
  APPROVED FOR RELEASE ✓

Issue #44: Export to CSV
  ✓ Contract compliance: 1/1 invariants satisfied
  ✓ Test coverage: 100% (2/2 criteria tested)
  ✓ Journey coverage: 0 (feature-only, no journey)
  ✓ Build success: Production build OK
  ✓ No regressions: 127/127 existing tests pass

  Quality score: 100% (5/5 gates passed)
  APPROVED FOR RELEASE ✓

Overall: ✓ All quality gates passed
```

---

## Phase 7: Issue Closure

**Goal:** Update GitHub issues with results and close completed work.

### GitHub Updates

For each verified issue:

1. **Add completion comment:**
```markdown
🎉 Issue completed via Specflow waves-controller

**Contracts Created:**
- docs/contracts/feature_authentication.yml (4 invariants)
- docs/contracts/journey_user_login.yml (3 steps)

**Tests Created:**
- Contract tests: 4 (all passing)
- E2E tests: 3 (all passing)

**Files Generated:**
- supabase/migrations/20260203_create_users.sql
- supabase/functions/login/index.ts
- src/features/auth/LoginPage.tsx
- tests/e2e/journey_user_login.spec.ts

**Quality Gates:** 5/5 passed
**Status:** VERIFIED ✓

All acceptance criteria met. Safe to merge.
```

2. **Add labels:**
- `specflow-verified`
- `ready-to-merge`

3. **Close issue:**
```bash
gh issue close #42 --comment "Verified by waves-controller"
```

### Closure Report

```
🌊 PHASE 7: ISSUE CLOSURE

Wave 1 Closures:

Issue #42: Add user authentication
  ✓ Comment added with completion summary
  ✓ Labeled: specflow-verified, ready-to-merge
  ✓ Closed

Issue #44: Export to CSV
  ✓ Comment added with completion summary
  ✓ Labeled: specflow-verified, ready-to-merge
  ✓ Closed

Overall: ✓ 2/2 issues closed
```

---

## Phase 8: Reporting

**Goal:** Output summary and determine next wave.

### Wave Summary

```
🌊 PHASE 8: REPORTING

═══════════════════════════════════════════════════
             WAVE 1 EXECUTION COMPLETE
═══════════════════════════════════════════════════

Duration: 8m 42s
Issues processed: 2
Issues closed: 2
Contracts created: 3
Migrations generated: 1
Edge Functions created: 2
Frontend components: 1
Contract tests: 5
E2E tests: 4
Quality gates: 10/10 passed

Manual effort estimate: 8-12 hours
Time saved: ~10 hours

═══════════════════════════════════════════════════
                   NEXT WAVE
═══════════════════════════════════════════════════

Wave 2 is now unblocked (2 issues ready):
  #43: Add profile page [important]
  #45: Add settings page [important]

Remaining after Wave 2: 8 issues

Ready to execute Wave 2? (yes/no)
```

### Continuous Execution

If user says "yes", waves-controller continues to Wave 2:

```
🌊 STARTING WAVE 2

Proceeding to Phase 2 (Wave Planning) for next wave...
```

Otherwise, execution pauses. User can resume anytime:
```
Execute waves
```

---

## Agent Teams Mode (Default)

Agent Teams is the default execution model. waves-controller automatically detects TeammateTool availability and uses persistent peer-to-peer teammate coordination when available.

### Auto-Detection

waves-controller checks for TeammateTool capability at startup:

```
Phase 0: Capability Detection

TeammateTool available → Agent Teams mode (default)
  → Persistent teammates, peer-to-peer coordination
  → Three-tier journey gates
  → Mandatory execution visualizations

TeammateTool unavailable → Subagent mode (fallback)
  → Task tool spawns one-shot agents
  → Hub-and-spoke coordination
```

No environment variable needed. Detection is automatic.

### Phase Changes in Agent Teams Mode

Several phases are modified or extended when running in teams mode:

#### Phase 2: Team Spawning (replaces Wave Planning)

Instead of planning which subagents to spawn per issue, waves-controller uses `TeammateTool(spawnTeam)` to create persistent teammates:

```javascript
TeammateTool.spawnTeam({
  teammates: [
    { role: "issue-lifecycle", issue: "#42", persistent: true },
    { role: "issue-lifecycle", issue: "#43", persistent: true },
    { role: "issue-lifecycle", issue: "#44", persistent: true },
    { role: "db-coordinator", persistent: true },
    { role: "quality-gate", persistent: true },
    { role: "journey-gate", persistent: true }
  ]
})
```

Each issue-lifecycle teammate handles all phases (contracts, migration, implementation, testing) for its issue. The coordinator agents (db-coordinator, quality-gate, journey-gate) are shared across the wave.

#### Phases 3-5: Self-Coordination (replaces Agent Spawning + Parallel Execution + Validation)

In subagent mode, waves-controller explicitly spawns agents, monitors execution, and triggers validation. In teams mode, **teammates self-coordinate**:

- issue-lifecycle agents request migration numbers from db-coordinator
- issue-lifecycle agents broadcast file modifications to avoid conflicts
- issue-lifecycle agents request test execution from quality-gate
- quality-gate returns results directly to the requesting teammate
- Teammates fix their own test failures (persistent context)

waves-controller only observes and receives `ISSUE_COMPLETE` / `ISSUE_BLOCKED` messages.

#### Phase 6b: Wave Gate (new)

After all issues in a wave pass their individual Tier 1 gates, waves-controller triggers the Wave Gate:

```
waves-controller → quality-gate: RUN_WAVE_GATE
quality-gate:
  → Run all contract tests
  → Run all E2E tests
  → Compare against .specflow/baseline.json
  → Report: WAVE_APPROVED or WAVE_REJECTED
```

If the wave gate fails, waves-controller identifies which issue caused the regression and re-opens it for the responsible issue-lifecycle teammate to fix.

#### Phase 6c: Regression Gate (new)

After the final wave completes, before any merge to main:

```
waves-controller → quality-gate: RUN_REGRESSION
quality-gate:
  → Run full test suite (contracts + E2E + build)
  → Compare against .specflow/baseline.json
  → Check .claude/.defer-journal for deferred failures
  → Report: WAVE_APPROVED (safe to merge) or WAVE_REJECTED (regressions found)
```

This is the final checkpoint. No code merges to main unless the regression gate passes.

#### Phase 7b: Graceful Shutdown (new)

After all waves complete and the regression gate passes, waves-controller performs graceful shutdown:

```
waves-controller:
  → Broadcast SHUTDOWN to all teammates
  → Wait for teammates to persist any final state
  → Collect final metrics from each teammate
  → Update .specflow/baseline.json with new test counts
  → Generate final report (Phase 8)
```

### Benefits of Agent Teams Mode

| Benefit | Details |
|---------|---------|
| **3-5x faster** | Less coordination overhead, persistent context eliminates re-reading |
| **Persistent context** | Agents remember their own code and can self-repair failures |
| **Parallel with coordination** | Agents work in parallel but coordinate directly (no bottleneck) |
| **Migration safety** | db-coordinator prevents migration number collisions |
| **Three-tier testing** | Graduated quality gates match scope to phase |
| **Regression tracking** | Baseline comparison prevents silent regressions |
| **Graceful degradation** | Falls back to subagent mode without env var |

### Agent Teams Phase Summary

| Phase | Subagent Mode | Agent Teams Mode |
|-------|---------------|-----------------|
| 1 | Discovery | Discovery (same) |
| 2 | Wave Planning | **Team Spawning** |
| 3 | Agent Spawning | **Self-Coordination** |
| 4 | Parallel Execution | **Self-Coordination** |
| 5 | Validation | **Self-Coordination** |
| 6 | Quality Gates | Quality Gates + **6b: Wave Gate** + **6c: Regression Gate** |
| 7 | Issue Closure | Issue Closure + **7b: Graceful Shutdown** |
| 8 | Reporting | Reporting (same) |

**[Full Agent Teams Documentation -->](/agent-system/agent-teams/)**

---

## Error Handling

### When Tests Fail

```
🌊 PHASE 5: VALIDATION

Wave 1 Test Results:

Issue #42: Add user authentication
  Contract tests: ✗ 1/4 failed
    ❌ AUTH-001: Password hashing check FAILED
       File: src/features/auth/login.ts:23
       Expected: bcrypt.hash(password)
       Actual: password (plaintext)

  Status: BLOCKED ✗

⚠️ WAVE 1 BLOCKED

Reason: Contract violation detected
Failed issue: #42
Recommendation: Fix AUTH-001 violation, then re-run wave

Wave execution paused. Fix issues and run "Execute waves" to retry.
```

### When Agent Times Out

```
⚠️ AGENT TIMEOUT

Agent: edge-function-builder
Issue: #42
Duration: 10m (exceeded 5m limit)

Possible causes:
  - Issue too complex (split into smaller issues)
  - Agent stuck on dependency (check logs)
  - Infrastructure issue (retry)

Recommendation:
  - Simplify issue #42 (reduce scope)
  - Or manually implement and re-run validation
```

---

## Configuration

### Wave Size Limits

Control parallelism:

```yaml
# .specflow.yml
waves:
  max_issues_per_wave: 5  # Don't overwhelm agents
  max_agents_per_issue: 6  # Limit parallel spawns
```

### Timeout Settings

```yaml
timeouts:
  agent_execution: 5m  # Per agent
  test_suite: 10m      # All tests
  wave_total: 30m      # Entire wave
```

### Priority Overrides

```yaml
priority_weights:
  severity: 10  # critical > important > future
  blocks_count: 5  # Issues that unblock others
  age: 2  # Older issues prioritized
  complexity: -1  # Simpler first
```

---

## Next Steps

- **[Agent Reference](/agent-system/agent-reference/)** — Complete list of all 23+ agents
- **[Agent Teams](/agent-system/agent-teams/)** — Persistent teammate coordination
- **[DPAO Methodology](/agent-system/dpao/)** — Deep dive into parallel execution
- **[Customizing Agents](/agent-system/customizing/)** -- Edit agent prompts for your workflow

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
