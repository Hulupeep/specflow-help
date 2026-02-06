---
layout: default
title: DPAO Methodology
parent: Agent System
nav_order: 3
permalink: /agent-system/dpao/
---

# DPAO Methodology
{: .no_toc }

Dependency-Based Parallel Agent Orchestration for 3-4x faster execution.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What is DPAO?

**DPAO (Dependency-Based Parallel Agent Orchestration)** is Specflow's methodology for executing multiple GitHub issues simultaneously while respecting dependencies.

**Key principle:** Independent work runs in parallel. Dependent work runs in sequence.

---

## The Four Phases

### 1. Discovery
**Goal:** Find all open issues and extract dependencies

```bash
gh issue list --state open --json number,title,body,labels
```

**Output:** Dependency graph showing which issues block others

### 2. Parallel Execution
**Goal:** Spawn agents for all independent issues in a wave

**Example Wave 1:**
- Issue #50: Add Login (no dependencies)
- Issue #51: Add Signup (no dependencies)
- Issue #52: Add Dashboard (blocked by #50)

**Action:** Spawn agents for #50 and #51 in parallel. Wait for completion before #52.

### 3. Analysis
**Goal:** Check quality gates

**Verification:**
- All contract tests pass
- All E2E tests pass
- Build succeeds
- No merge conflicts

**If fails:** Stop wave execution, report failures

### 4. Orchestration
**Goal:** Move to next wave or report completion

**Next wave:** Unblocked issues become available
**Complete:** All issues closed, all tests passing

---

## Wave Calculation Algorithm

```typescript
function calculateWaves(issues: Issue[]): Wave[] {
  const waves: Wave[] = []
  const completed = new Set<number>()

  while (completed.size < issues.length) {
    // Find issues with all dependencies completed
    const ready = issues.filter(issue =>
      !completed.has(issue.number) &&
      issue.dependencies.every(dep => completed.has(dep))
    )

    if (ready.length === 0) break // Circular dependency

    waves.push({ issues: ready })
    ready.forEach(issue => completed.add(issue.number))
  }

  return waves
}
```

---

## Speedup Metrics

| Scenario | Sequential | DPAO | Speedup |
|----------|-----------|------|---------|
| 8 independent issues | 16 hours | 2-3 hours | **4-5.3x** |
| 9 issues (2 waves) | 18 hours | 4 hours | **4.5x** |
| 12 issues (3 waves) | 24 hours | 8 hours | **3x** |

**Real example:** Wave 2 Batch 2 (Timebreez)
- 9 issues → 8 completed
- Time: 2-3 hours (vs 18+ hours sequential)
- Speedup: **6-9x**

---

## Contract Extensions for DPAO

To achieve <5% failure rates with parallel execution, use these contract extensions:

### 1. Anti-Patterns Section
Prevents agents from using wrong libraries:

```yaml
anti_patterns:
  libraries:
    forbidden:
      - name: "shadcn/ui"
        reason: "Project uses native HTML + Tailwind"
```

### 2. Completion Verification Checklist
Catches lost work:

```yaml
completion_verification:
  required_files:
    - path: "supabase/functions/webhook/index.ts"
      must_contain: "export async function handler"
```

### 3. Parallel Coordination Rules
Prevents merge conflicts:

```yaml
parallel_coordination:
  consolidation_files:
    - path: "src/features/*/index.ts"
      rule: "DO NOT MODIFY during parallel execution"
```

**Full details:** [CONTRACT-SCHEMA-EXTENSIONS.md](https://github.com/Hulupeep/Specflow/blob/main/CONTRACT-SCHEMA-EXTENSIONS.md)

---

## Quality Gates

DPAO enforces quality at each wave:

| Gate | Check | Action on Failure |
|------|-------|-------------------|
| **Contracts** | All contract tests pass | Block wave, report violations |
| **Build** | `npm run build` succeeds | Block wave, report errors |
| **Tests** | E2E journeys pass | Block wave, report failures |
| **Merge** | No conflicts | Block wave, resolve conflicts |

**Philosophy:** Ship only when all gates pass. Speed with safety.

---

## Case Study: Timebreez Wave 2 Batch 2

**Setup:**
- 9 GitHub issues
- 2 waves calculated
- 23+ specialized agents

**Execution:**
- **Wave 1:** 6 independent issues (parallel)
  - Time: 1.5 hours
  - Result: 5 completed, 1 failed (lost work)

- **Wave 2:** 3 dependent issues (parallel)
  - Time: 1 hour
  - Result: 3 completed

**Total:**
- **8/9 completed** (88.9% success)
- **2-3 hours total** (vs 18+ sequential)
- **Failure:** 1 agent reported complete but lost files (no completion verification)

**Lesson learned:** Added Completion Verification Checklist → <5% failure rate in subsequent waves

---

## Prerequisites for DPAO

**Before running DPAO:**
1. ✅ GitHub issues formatted with Gherkin acceptance criteria
2. ✅ `data-testid` attributes specified for E2E tests
3. ✅ Contract extensions added (Anti-Patterns, Completion Verification, Parallel Coordination)
4. ✅ CI pipeline configured to run contracts + E2E tests

**Without these:** Failure rate >10%, agents drift, rework required

---

## Running DPAO

```bash
# 1. Invoke waves-controller via Claude Code
# Read scripts/agents/waves-controller.md
# Task("Execute waves", "{prompt content}", "general-purpose")

# 2. waves-controller handles:
# - Discovery: Reads all open issues
# - Dependency calculation: Builds wave graph
# - Parallel spawning: Launches agents for Wave 1
# - Quality gates: Verifies tests pass
# - Next wave: Moves to Wave 2 when Wave 1 completes
# - Completion: Reports results, closes issues

# 3. Monitor progress:
# - ASCII wave diagrams show current state
# - Agent reports show per-issue progress
# - Test failures block waves automatically

# 4. Results:
# - Issues closed: X/Y
# - Time saved: Zx
# - Failure rate: N%
```

---

## Comparison to Other Approaches

### vs Sequential (Manual)
- **Sequential:** 1 issue at a time, 2 hours each = 18 hours for 9 issues
- **DPAO:** 2-3 waves, 1-2 hours per wave = 2-3 hours total
- **Speedup:** 6-9x

### vs Full Parallel (No Dependencies)
- **Full Parallel:** All issues at once, merge conflicts, integration hell
- **DPAO:** Respects dependencies, no conflicts
- **Trade-off:** Slightly slower than full parallel, but safe

### vs Claude Flow
- **Claude Flow:** Coordination overhead, mesh topology
- **DPAO:** Direct agent spawning, wave-based coordination
- **Result:** DPAO is simpler, faster for backlog execution

---

## Next Steps

- **[waves-controller Deep Dive](/agent-system/waves-controller/)** — Learn the 8-phase orchestration
- **[Agent Reference](/agent-system/agent-reference/)** — See all 23+ agent types
- **[Contract Extensions](https://github.com/Hulupeep/Specflow/blob/main/CONTRACT-SCHEMA-EXTENSIONS.md)** — Reduce failure rates

---

## DPAO in Production

**Used by:**
- Timebreez (childcare scheduling system)
- HookTunnel (webhook infrastructure)
- Specflow documentation site (this site!)

**Results consistently show:**
- 3-4x speedup vs sequential
- <5% failure rate with extensions
- 90%+ completion rate per wave

**DPAO is production-ready.**
