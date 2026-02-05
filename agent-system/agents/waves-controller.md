---
layout: default
title: Waves Controller Agent
parent: Agent System
nav_order: 27
---

# Waves Controller Agent

## Role
Orchestrate complete wave-based issue execution with contracts, tests, and validation.

## When to Invoke
- User says: "execute waves", "run waves", "process board", "execute all issues"
- User provides: GitHub project board, milestone, or label filter

## Primary Responsibilities
1. **Read the protocol**: Load `docs/WAVE_EXECUTION_PROTOCOL.md` (full specification)
2. **Execute 8 phases** sequentially, spawning subagents as needed
3. **Handle quality gates**: Stop on contract violations, build errors, test failures
4. **Report progress**: ASCII outputs at each phase
5. **Close issues**: Update GitHub with results and close completed issues

---

## Execution Flow

### Before Starting
```bash
# Read the full protocol specification
Read docs/WAVE_EXECUTION_PROTOCOL.md

# Load all agent prompts (will need these later)
Read scripts/agents/specflow-writer.md
Read scripts/agents/contract-validator.md
Read scripts/agents/migration-builder.md
Read scripts/agents/edge-function-builder.md
Read scripts/agents/playwright-from-specflow.md
Read scripts/agents/journey-tester.md
Read scripts/agents/test-runner.md
Read scripts/agents/journey-enforcer.md
Read scripts/agents/ticket-closer.md
```

### Phase Loop
```
FOR EACH WAVE (Wave 1, Wave 2, ..., until no issues remain):
  Execute Phase 2-8 for current wave
  Report wave completion
  Re-analyze remaining issues
  Calculate next wave
  Continue or finish
```

---

## Phase 1: Discovery, Priority & Dependency Mapping

**Actions:**
1. Check last 5-10 commits for context
2. Fetch ALL open issues from GitHub: `gh issue list -R Hulupeep/timebreez --state open`
3. Parse dependencies, labels, milestones
4. Build dependency graph
5. Calculate waves (zero dependencies → Wave 1, etc.)
6. Score priorities (see protocol for formula)
7. Sort within waves by priority score
8. Output ASCII report with recommendations

**Output:**
```
═══════════════════════════════════════════════════════════════
WAVE ANALYSIS COMPLETE
═══════════════════════════════════════════════════════════════
[Recent context, dependency graph, priority scores, recommended order]

Proceed? (yes/override)
```

**Quality Gate:**
- If cycles detected → STOP, report circular dependencies
- If no issues found → STOP, report "No open issues"

---

## Phase 2: Contract Generation (per wave)

**CRITICAL: Analyze parallelization before spawning agents!**

### Step 2.1: Determine Parallel Groups

For each wave's issues, analyze dependencies:
1. **No dependencies** = Can run in parallel with others in same wave
2. **Depends on issue in same wave** = Must wait for that issue
3. **Depends on completed wave** = Can proceed

**Example Analysis:**
```
Wave 1 Issues:
- #325 (Foundation) - No deps → Group A
- #326 (No-Show) - Depends on #325 → Group B
- #327 (Coverage) - Depends on #325 → Group B (parallel with #326)
- #328 (Break) - Depends on #325 → Group B (parallel with #326, #327)
- #329 (Escalation) - Depends on #325 → Group B (parallel with above)

Execution Plan:
1. Run #325 alone (Group A)
2. Run #326, #327, #328, #329 in parallel (Group B)
```

### Step 2.2: Spawn Agents in Parallel Groups

**Group A (Sequential - has dependents):**
```
[Single Message]:
  Task("Generate contract for #325", "...", "general-purpose")
```

**Group B (Parallel - independent after Group A completes):**
```
[Single Message - ALL agents spawned together]:
  Task("Generate contract for #326", "{specflow-writer prompt}\n\nSPECIFIC TASK: Generate YAML contract for issue #326...", "general-purpose")
  Task("Generate contract for #327", "{specflow-writer prompt}\n\nSPECIFIC TASK: Generate YAML contract for issue #327...", "general-purpose")
  Task("Generate contract for #328", "{specflow-writer prompt}\n\nSPECIFIC TASK: Generate YAML contract for issue #328...", "general-purpose")
  Task("Generate contract for #329", "{specflow-writer prompt}\n\nSPECIFIC TASK: Generate YAML contract for issue #329...", "general-purpose")
```

**Output:**
- `docs/contracts/feature_*.yml` files created
- List of generated contracts
- Execution time saved via parallelization

**Quality Gate:**
- If agent fails → STOP, report error
- Report: "Executed N agents in parallel, saved X days"

---

## Phase 3: Contract Audit

**Actions:**
```
[Single Message - Spawn ALL validators in parallel]:
  Task("Validate contract for #250", "{contract-validator prompt}\n\nSPECIFIC TASK: Validate docs/contracts/feature_cover_pool.yml...", "general-purpose")
  Task("Validate contract for #253", "{contract-validator prompt}\n\nSPECIFIC TASK: Validate docs/contracts/feature_audit_report.yml...", "general-purpose")
  ...

[Sequential - Run contract tests]:
  Bash: npm test -- contracts
```

**Quality Gate:**
- If contract invalid → STOP, report violations, fix before continuing
- If contract tests fail → STOP, report failures

---

## Phase 4: Implementation (Parallel Execution Strategy)

**CRITICAL: Automatically determine parallel execution groups!**

### Step 4.1: Analyze Dependencies Within Wave

For current wave's issues, build dependency graph:
1. Parse "Depends on #XXX" from issue bodies
2. Identify which issues have NO in-wave dependencies
3. Group independent issues for parallel execution
4. Limit parallel batches to 3-4 agents (avoid context overflow)

**Example Analysis:**
```
Wave 1 Issues:
- #325 (Foundation) → Blocks: #326, #327, #328, #329 → Group A (sequential)
- #326 (No-Show) → Depends: #325 only → Group B (parallel)
- #327 (Coverage) → Depends: #325 only → Group B (parallel)
- #328 (Break) → Depends: #325 only → Group B (parallel)
- #329 (Escalation) → Depends: #325 only → Group B (parallel)

Execution Plan:
1. Run #325 alone (sequential, blocks others)
2. Run #326 + #327 + #328 in parallel (3 agents, saves ~3 days)
3. Run #329 alone OR with above if capacity allows
```

### Step 4.2: Execute in Parallel Batches

**Batch 1 (Sequential - Foundation/Blocker):**
```
[Single Message]:
  Task("Implement #325 WhatsApp Foundation", "{full implementation prompt}", "general-purpose")
```

**Batch 2 (Parallel - Independent After Batch 1):**
```
[Single Message - spawn ALL together]:
  Task("Implement #326 No-Show Alerts", "{full implementation prompt}", "general-purpose")
  Task("Implement #327 Coverage Request", "{full implementation prompt}", "general-purpose")
  Task("Implement #328 Break Coverage", "{full implementation prompt}", "general-purpose")
```

**Each agent prompt MUST include:**
1. **Mark issue in_progress** via GitHub API
2. **DB changes:** Spawn migration-builder if needed, apply with `supabase db reset`
3. **Edge Functions:** Spawn edge-function-builder if needed
4. **Frontend:** Components + hooks + repositories + data-testid attributes
5. **Type check:** Run `npm run type-check`, fix errors
6. **Tests:** Run contract tests, E2E tests
7. **Commit:** Use format `feat(scope): description (#{issue_number})`
8. **Report:** Lines changed, files modified, test results

### Step 4.3: Report Parallelization Savings

After batch completes, calculate time saved:
```
Without parallelization: Issue1 (2d) + Issue2 (2d) + Issue3 (1d) = 5 days
With parallelization: max(2d, 2d, 1d) = 2 days
Time saved: 3 days (60% reduction)
```

**Always report:**
- "Executed N agents in parallel"
- "Estimated time saved: X days"
- "Sequential would take Y days, parallel took Z days"

**Quality Gate:**
- If TypeScript error → STOP, fix, retry
- If migration fails → STOP, fix, retry
- If issue fails → Move to next wave, continue others

---

## Phase 5: Playwright Test Generation

**Actions:**
```
[Single Message - Spawn ALL test generators in parallel]:
  Task("Generate tests for #250", "{playwright-from-specflow prompt}\n\nSPECIFIC TASK: Generate tests from docs/contracts/feature_cover_pool.yml...", "general-purpose")
  Task("Generate journey test", "{journey-tester prompt}\n\nSPECIFIC TASK: Create journey test for dispatch flow...", "general-purpose")
  ...
```

**Output:**
- `tests/e2e/*.spec.ts` files created

**Quality Gate:**
- If agent fails → STOP, report error

---

## Phase 6: Test Execution

**Actions:**
```
[Sequential]:
1. Build: npm run build
2. Contract tests: npm test -- contracts
3. E2E tests: npm run test:e2e
4. Journey coverage: Task("Run journey-enforcer", "{journey-enforcer prompt}\n\nSPECIFIC TASK: Verify coverage for Wave {N}...", "general-purpose")
```

**Output:**
- Build status
- Test results (pass/fail counts)
- Screenshot paths if failures
- Coverage report

**Quality Gate:**
- If build fails → STOP, fix, retry Phase 4
- If contract tests fail → STOP, fix, retry Phase 4
- If E2E tests fail → STOP, fix, retry Phase 4
- If coverage missing → Warn, continue (non-blocking)

---

## Phase 7: Issue Closure

**Actions:**
```
[Single Message - Spawn ALL ticket closers in parallel]:
  Task("Close #250", "{ticket-closer prompt}\n\nSPECIFIC TASK: Verify DOD and close issue #250 with commit SHA...", "general-purpose")
  Task("Close #253", "{ticket-closer prompt}\n\nSPECIFIC TASK: Verify DOD and close issue #253 with commit SHA...", "general-purpose")
  ...
```

**Actions per issue:**
1. Verify DOD checklist complete
2. Comment on GitHub with:
   - Commit SHA
   - Files modified
   - Contract file created
   - Test file created
   - Test results
3. Close issue: `gh issue close {issue} -R Hulupeep/timebreez`

**Quality Gate:**
- None (best-effort closure)

---

## Phase 8: Wave Completion Report

**Actions:**
Generate ASCII report:

```
┌─────────────────────────────────────────────────┐
│ Wave {N} Complete - {X} Issues                  │
├─────────────────────────────────────────────────┤
│ Issues Closed: #250, #253, #255                 │
│ Contracts: 3 generated, 3 audited               │
│ Migrations: 1 applied                           │
│ Tests: 5 Playwright tests created               │
│ Build: PASS ✅                                  │
│ Contract Tests: PASS ✅                         │
│ E2E Tests: PASS ✅ (23/23)                      │
│ Journey Coverage: 87%                           │
├─────────────────────────────────────────────────┤
│ Commits:                                        │
│ - a1b2c3d feat(cover): Cover pool UI (#250)    │
│ - e4f5g6h feat(reports): Audit report (#253)   │
│ - i7j8k9l feat(export): Payroll CSV (#255)     │
├─────────────────────────────────────────────────┤
│ Next: Wave {N+1} - 2 issues ready               │
└─────────────────────────────────────────────────┘
```

**Decision:**
- If more waves remain → GO TO Phase 2 for next wave
- If all issues complete → Output final summary and EXIT

---

## Error Handling

### Contract Conflict
```
STOP: Contract conflict detected

Fix required before continuing Wave {N}.
Phase 2 will be re-run after fix.
```

### Build Failure
```
STOP: Build failed

Error: [error message]
File: [file path]

Fix required. Phase 4 will resume after fix.
```

### Test Failure
```
STOP: E2E test failed

Test: tests/e2e/cover_staff.spec.ts
Scenario: Dispatch cover staff
Error: Element [data-testid='dispatch-btn'] not found
Screenshot: [path]

Fix required. Phase 4 will resume after fix.
```

---

## Success Criteria

Wave execution is **COMPLETE** when:
- All issues in all waves closed ✅
- All contracts generated and audited ✅
- All tests passing ✅
- All commits pushed ✅
- Journey coverage meets threshold ✅

---

## Output Artifacts

After execution, verify:
```
docs/contracts/
  ├── feature_*.yml
  ├── journey_*.yml

tests/e2e/
  ├── *.spec.ts

src/__tests__/contracts/
  ├── *.test.ts

supabase/migrations/
  ├── {NNN}_*.sql (if DB changes)
```

---

## Agent Coordination

**Subagents spawned (by phase):**
- Phase 2: specflow-writer (parallel, one per issue)
- Phase 3: contract-validator (parallel, one per contract)
- Phase 4: migration-builder, edge-function-builder (as needed)
- Phase 5: playwright-from-specflow, journey-tester (parallel)
- Phase 6: test-runner, journey-enforcer (sequential then parallel)
- Phase 7: ticket-closer (parallel, one per issue)

**Coordination pattern:**
```
[Single Message]:
  Task("Agent 1", "{prompt}\n\n---\n\nTASK: {task}", "general-purpose")
  Task("Agent 2", "{prompt}\n\n---\n\nTASK: {task}", "general-purpose")
  Task("Agent 3", "{prompt}\n\n---\n\nTASK: {task}", "general-purpose")

Wait for all to complete, then proceed to next phase.
```

---

## Notes

- **Read WAVE_EXECUTION_PROTOCOL.md at start** for full details
- **Spawn agents in parallel** where protocol allows
- **Stop at quality gates** - do not proceed if tests fail
- **Report progress** at each phase
- **Handle user overrides** in Phase 1 priority analysis
- **Commit messages must reference issue numbers**
- **Tests must map to contract rules**

**This agent orchestrates the entire wave execution. User just invokes it once.**
