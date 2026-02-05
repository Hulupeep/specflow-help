---
layout: default
title: Agent Reference
parent: Agent System
nav_order: 2
permalink: /agent-system/agent-reference/
---

# Complete Agent Reference
{: .no_toc }

All 23+ Specflow agents and when to use them.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Agent Categories

Specflow has 23+ specialized agents grouped into 6 categories:

1. **Orchestration** (1 agent) — waves-controller
2. **Contract Generation** (3 agents) — Create and test contracts
3. **Implementation** (3 agents) — Generate code (database, backend, frontend)
4. **Testing** (4 agents) — Generate and run tests
5. **Validation & Workflow** (7 agents) — Verify quality, manage issues
6. **Agent Teams** (5 agents) — Persistent teammate coordination

---

## 1. Orchestration Agent

### waves-controller

**Role:** Orchestrate complete wave-based issue execution

**When to use:**
- User says: "Execute waves", "Run waves", "Process board"
- You want to execute multiple GitHub issues automatically

**Inputs:**
- GitHub repository (current repo by default)
- Open issues (filtered by labels, milestones, or all)

**Outputs:**
- Spawns all other agents as needed
- Generates contracts, code, tests
- Closes verified issues

**Example:**
```
Execute waves
```

**Time savings:** 3-4x faster than manual workflows

**[Full Documentation →](/agent-system/waves-controller/)**

---

## 2. Contract Generation Agents

### specflow-writer

**Role:** Create Specflow contracts from GitHub issues

**When to use:**
- GitHub issue has acceptance criteria (Gherkin format preferred)
- New feature needs contracts defined
- User asks to "create contracts" or "write specifications"

**Inputs:**
- GitHub issue URL or number
- Issue description and acceptance criteria

**Outputs:**
- `docs/contracts/feature_<name>.yml` (architectural invariants)
- `docs/contracts/journey_<name>.yml` (if workflow described)
- GitHub comment with contract summary

**Example:**
```
Read scripts/agents/specflow-writer.md, then create contracts for issue #42
```

**Quality gates:**
- YAML validates against CONTRACT-SCHEMA.md
- All acceptance criteria mapped to contract
- Invariants have enforcement method specified

**[Full Prompt →](/agent-system/agents/specflow-writer/)**

---

### contract-generator

**Role:** Generate contract YAML from natural language specs

**When to use:**
- You have feature specs but no GitHub issue yet
- Manual contract creation needed
- User provides requirements in plain text

**Inputs:**
- Natural language specification
- Feature area (e.g., "authentication", "leave management")

**Outputs:**
- Contract YAML file
- Invariants list
- Enforcement methods

**Example:**
```
Read scripts/agents/contract-generator.md, then generate a contract for: "User passwords must be hashed with bcrypt and stored with salt"
```

**Difference from specflow-writer:**
- specflow-writer: Reads GitHub issues
- contract-generator: Reads plain text specs

**[Full Prompt →](/agent-system/agents/contract-generator/)**

---

### contract-test-generator

**Role:** Generate contract tests (TypeScript) from contract YAML

**When to use:**
- Contract exists but test doesn't
- User asks to "create contract tests"
- After specflow-writer generates contracts

**Inputs:**
- Contract YAML file (e.g., `feature_authentication.yml`)
- Test framework (Vitest, Jest, etc.)

**Outputs:**
- `src/__tests__/contracts/<name>.test.ts`
- Tests that scan code for pattern violations

**Example:**
```
Read scripts/agents/contract-test-generator.md, then create tests for docs/contracts/feature_authentication.yml
```

**Quality gates:**
- Test fails when invariant is violated
- Error message includes contract ID (e.g., "CONTRACT VIOLATION: AUTH-001")
- Test scans relevant files only (performance)

**[Full Prompt →](/agent-system/agents/contract-test-generator/)**

---

## 3. Implementation Agents

### migration-builder

**Role:** Generate database migrations from feature requirements

**When to use:**
- Feature needs database schema changes
- Issue mentions tables, columns, or schema
- User asks to "create migration"

**Inputs:**
- GitHub issue or feature description
- Existing schema (from `supabase/migrations/`)

**Outputs:**
- `supabase/migrations/<timestamp>_<description>.sql`
- RLS policies for all tables
- Indexes for performance

**Example:**
```
Read scripts/agents/migration-builder.md, then generate migrations for issue #42
```

**Quality gates:**
- All tables have RLS policies (ARCH-004)
- Migration passes `supabase db lint`
- No foreign key conflicts

**[Full Prompt →](/agent-system/agents/migration-builder/)**

---

### edge-function-builder

**Role:** Create Supabase Edge Functions (backend RPCs)

**When to use:**
- Feature needs backend logic (authentication, data processing)
- Issue mentions "API", "endpoint", or "RPC"
- Contracts specify server-side invariants

**Inputs:**
- GitHub issue or feature description
- Contracts (for business logic rules)

**Outputs:**
- `supabase/functions/<name>/index.ts`
- RPC with input validation
- Error handling

**Example:**
```
Read scripts/agents/edge-function-builder.md, then create login RPC for issue #42
```

**Quality gates:**
- Input validation (Zod schemas)
- Error responses follow standard format
- Satisfies relevant contracts (e.g., AUTH-001: bcrypt hashing)

**[Full Prompt →](/agent-system/agents/edge-function-builder/)**

---

### frontend-builder

**Role:** Generate React components and hooks

**When to use:**
- Feature needs UI
- Issue mentions "page", "component", or "form"
- data-testid selectors specified in issue

**Inputs:**
- GitHub issue or feature description
- Contracts (for data flow requirements)
- data-testid requirements

**Outputs:**
- `src/features/<name>/<Component>.tsx`
- React Query hooks (if backend integration)
- data-testid attributes on all interactive elements

**Example:**
```
Read scripts/agents/frontend-builder.md, then create LoginPage for issue #42
```

**Quality gates:**
- All interactive elements have data-testid
- TypeScript types defined
- Integrates with repository pattern

**[Full Prompt →](/agent-system/agents/frontend-builder/)**

---

## 4. Testing Agents

### playwright-from-specflow

**Role:** Generate Playwright E2E tests from journey contracts

**When to use:**
- Journey contract exists (created by specflow-writer)
- User asks to "create E2E tests"
- After frontend implementation complete

**Inputs:**
- Journey contract YAML (e.g., `journey_user_login.yml`)
- data-testid selectors (from GitHub issue)

**Outputs:**
- `tests/e2e/journey_<name>.spec.ts`
- Tests with preconditions, steps, postconditions
- Database assertions

**Example:**
```
Read scripts/agents/playwright-from-specflow.md, then create tests for docs/contracts/journey_user_login.yml
```

**Quality gates:**
- All journey steps mapped to test steps
- preconditions in `beforeEach`, postconditions in `afterEach`
- Uses data-testid selectors (stable)

**[Full Prompt →](/agent-system/agents/playwright-from-specflow/)**

---

### journey-tester

**Role:** Create cross-feature journey tests (complex workflows)

**When to use:**
- Workflow spans multiple features
- Integration testing needed
- User asks to "create journey tests"

**Inputs:**
- Feature list (e.g., "leave management + roster + notifications")
- Workflow description

**Outputs:**
- `tests/e2e/journey_<workflow_name>.spec.ts`
- Tests that verify multiple features work together

**Example:**
```
Read scripts/agents/journey-tester.md, then create journey test for: "Manager approves leave, staff balance updates, roster shows leave dates"
```

**Difference from playwright-from-specflow:**
- playwright-from-specflow: Single-feature journeys (from contracts)
- journey-tester: Cross-feature journeys (manual design)

**[Full Prompt →](/agent-system/agents/journey-tester/)**

---

### test-runner

**Role:** Execute tests and report failures with details

**When to use:**
- After code changes (MANDATORY)
- User asks to "run tests"
- Before marking work complete

**Inputs:**
- Test suite (contract tests, E2E tests, or both)
- Optional: Test filter (e.g., `--grep @critical`)

**Outputs:**
- Test results (pass/fail counts)
- Failure details (stack traces, screenshots)
- Recommendations for fixes

**Example:**
```
Read scripts/agents/test-runner.md, then run contract tests and E2E tests
```

**Quality gates:**
- All critical tests must pass
- Failures include actionable debugging info
- Maps failures to GitHub issues

**[Full Prompt →](/agent-system/agents/test-runner/)**

---

### e2e-test-auditor

**Role:** Verify E2E coverage for UI-related issues

**When to use:**
- Before release
- User asks to "check E2E coverage"
- Verifying journey coverage exists

**Inputs:**
- GitHub issues tagged with UI/frontend labels
- Journey contracts (all journeys)

**Outputs:**
- Coverage report (% of UI issues with E2E tests)
- List of issues missing journey coverage
- Recommendations for missing tests

**Example:**
```
Read scripts/agents/e2e-test-auditor.md, then audit E2E coverage for all UI issues
```

**Quality gates:**
- All critical UI issues have journey coverage
- Report includes missing data-testid selectors

**[Full Prompt →](/agent-system/agents/e2e-test-auditor/)**

---

## 5. Validation & Workflow Agents

### contract-validator

**Role:** Verify implementation satisfies contracts

**When to use:**
- After feature implementation
- User asks to "validate contracts"
- Before closing GitHub issue

**Inputs:**
- Contract YAML files
- Implementation code (relevant files)

**Outputs:**
- Validation report (invariants satisfied or violated)
- List of contract violations with file locations
- Recommendations for fixes

**Example:**
```
Read scripts/agents/contract-validator.md, then validate implementation against docs/contracts/feature_authentication.yml
```

**Quality gates:**
- All critical invariants satisfied
- Violations include file and line number
- Maps to contract tests (shows which test will fail)

**[Full Prompt →](/agent-system/agents/contract-validator/)**

---

### journey-enforcer

**Role:** Ensure critical journeys have coverage and pass

**When to use:**
- Before release
- User asks to "check journey coverage"
- Verifying release readiness

**Inputs:**
- All journey contracts
- E2E test results

**Outputs:**
- Journey coverage report
- List of critical journeys without tests
- List of critical journeys failing
- Release readiness verdict (APPROVED or BLOCKED)

**Example:**
```
Read scripts/agents/journey-enforcer.md, then verify journey coverage for release
```

**Quality gates:**
- All critical journeys have E2E tests
- All critical journeys passing
- Important journeys: warns if failing, but doesn't block

**[Full Prompt →](/agent-system/agents/journey-enforcer/)**

---

### ticket-closer

**Role:** Update and close GitHub issues after completion

**When to use:**
- After feature implementation and validation
- User asks to "close tickets"
- When all acceptance criteria met

**Inputs:**
- GitHub issue number
- Implementation details (files generated, tests created)
- Test results

**Outputs:**
- GitHub comment with completion summary
- Issue closed
- Labels added (`specflow-verified`, `ready-to-merge`)

**Example:**
```
Read scripts/agents/ticket-closer.md, then close issue #42 with completion summary
```

**Quality gates:**
- All acceptance criteria verified
- All tests passing
- Commits reference issue number

**[Full Prompt →](/agent-system/agents/ticket-closer/)**

---

### dependency-mapper

**Role:** Calculate issue dependencies for wave planning

**When to use:**
- Before executing waves
- User asks to "analyze dependencies"
- Verifying no circular dependencies

**Inputs:**
- All open GitHub issues
- Issue body text (looking for "blocked by", "depends on")

**Outputs:**
- Dependency graph
- List of issues with no blockers (Wave 1 candidates)
- Warning if circular dependencies detected

**Example:**
```
Read scripts/agents/dependency-mapper.md, then analyze dependencies for all open issues
```

**Quality gates:**
- No circular dependencies
- All blockers are open issues (not closed)

**[Full Prompt →](/agent-system/agents/dependency-mapper/)**

---

### board-auditor

**Role:** Analyze GitHub project board health

**When to use:**
- Weekly health check
- User asks to "audit board"
- Before sprint planning

**Inputs:**
- GitHub project board or milestone
- Issue metadata (age, labels, assignees)

**Outputs:**
- Health report (% completed, % blocked, % stale)
- List of stale issues (>30 days no activity)
- List of issues missing acceptance criteria
- Recommendations for cleanup

**Example:**
```
Read scripts/agents/board-auditor.md, then audit the GitHub project board
```

**Quality gates:**
- <20% stale issues (healthy)
- All issues have acceptance criteria
- No orphaned blockers (blocking closed issues)

**[Full Prompt →](/agent-system/agents/board-auditor/)**

---

### sprint-executor

**Role:** Execute all issues in a sprint/milestone

**When to use:**
- Sprint start (user says "execute sprint")
- Milestone deadline approaching
- Batch execution of related issues

**Inputs:**
- Sprint/milestone name
- Optional: Label filter

**Outputs:**
- Executes waves-controller for all sprint issues
- Sprint completion report

**Example:**
```
Read scripts/agents/sprint-executor.md, then execute all issues in milestone "v1.0"
```

**Difference from waves-controller:**
- waves-controller: Executes all open issues (or filtered)
- sprint-executor: Executes specific sprint/milestone only

**[Full Prompt →](/agent-system/agents/sprint-executor/)**

---

### specflow-uplifter

**Role:** Add Specflow to legacy codebases (mid-project adoption)

**When to use:**
- Existing project wants to adopt Specflow
- User asks to "add Specflow to project"
- Migration from manual testing to contracts

**Inputs:**
- Existing codebase
- Existing tests (if any)

**Outputs:**
- `Specflow/` directory with methodology files
- `docs/contracts/` with generated contracts (from existing code patterns)
- Updated `.gitignore`, `CLAUDE.md`
- Migration guide

**Example:**
```
Read scripts/agents/specflow-uplifter.md, then add Specflow to this legacy project
```

**Quality gates:**
- No breaking changes to existing code
- Contracts generated for existing patterns
- Tests still pass after migration

**[Full Prompt →](/agent-system/agents/specflow-uplifter/)**

---

## 6. Agent Teams Agents

These agents are available when `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true` is set. They enable persistent peer-to-peer coordination instead of hub-and-spoke subagent dispatching.

**[Full Agent Teams Documentation -->](/agent-system/agent-teams/)**

### issue-lifecycle

**Role:** Full lifecycle management for a single GitHub issue

**When to use:**
- Agent Teams mode is enabled
- waves-controller spawns one per issue in a wave
- Not invoked manually (spawned by waves-controller)

**Capabilities:**
- Owns entire issue from contracts through closure
- Maintains persistent context (no re-reading between phases)
- Self-repairs: detects test failures and fixes own bugs
- Communicates with db-coordinator, quality-gate, and peer issue-lifecycle agents

**Key difference from subagents:**
- Subagent mode: 6 separate agents per issue, each stateless
- issue-lifecycle: 1 persistent agent per issue, full context retained

---

### db-coordinator

**Role:** Migration number management and conflict detection

**When to use:**
- Agent Teams mode is enabled
- Multiple issues need database migrations simultaneously
- Spawned once per wave by waves-controller

**Capabilities:**
- Assigns sequential migration numbers (prevents `migration_175` collisions)
- Detects schema conflicts (two agents modifying same table)
- Validates foreign key dependencies across parallel migrations
- Responds to `REQUEST_MIGRATION` messages from issue-lifecycle agents

**Example interaction:**
```
issue-lifecycle-42: REQUEST_MIGRATION { table: "audit_log", operation: "CREATE TABLE" }
db-coordinator:     MIGRATION_ASSIGNED { number: 176, file: "176_create_audit_log.sql" }
```

---

### quality-gate

**Role:** Test execution service for teams

**When to use:**
- Agent Teams mode is enabled
- Spawned once per wave by waves-controller
- Handles all three tiers of testing

**Capabilities:**
- Executes contract tests (`pnpm test -- contracts`)
- Executes E2E tests (`pnpm test:e2e`)
- Executes build verification (`pnpm build`)
- Compares results against `.specflow/baseline.json`
- Determines pass/fail for issue gate, wave gate, and regression gate

**Tiers:**
- **Tier 1 (Issue Gate):** Per-issue tests, blocks issue closure
- **Tier 2 (Wave Gate):** Cross-issue tests, blocks next wave
- **Tier 3 (Regression Gate):** Full suite, blocks merge to main

---

### journey-gate

**Role:** Three-tier journey enforcement

**When to use:**
- Agent Teams mode is enabled
- Spawned once per wave by waves-controller
- Works alongside quality-gate for journey-specific verification

**Capabilities:**
- Maps which journeys are affected by which issues
- Ensures per-issue journey tests pass before closure (Tier 1)
- Validates cross-issue journey integrity after wave completion (Tier 2)
- Runs full journey regression before merge (Tier 3)
- Produces release readiness verdict (APPROVED or BLOCKED)

---

### PROTOCOL

**Role:** Communication protocol reference document

**Note:** PROTOCOL is not an executable agent. It is a reference document (`scripts/agents/PROTOCOL.md`) that defines all valid inter-agent message types, required fields, and expected responses.

**Contents:**
- All message types (REQUEST_MIGRATION, MIGRATION_ASSIGNED, TOUCHING_FILE, etc.)
- Required payload fields per message type
- Expected response patterns
- Error handling conventions

**See:** [Communication Protocol table](/agent-system/agent-teams/#communication-protocol)

---

## Agent Invocation Patterns

### Method 1: Let waves-controller Orchestrate (Recommended)

```
Execute waves
```

**Pros:**
- Automatic agent selection
- Parallel execution
- Handles dependencies
- 3-4x faster

**Cons:**
- Less control over individual agents

---

### Method 2: Invoke Individual Agents

```
Read scripts/agents/specflow-writer.md, then create contracts for issue #42
```

**Pros:**
- Full control over single agent
- Useful for testing/debugging
- Custom workflows

**Cons:**
- Slower (sequential execution)
- Must manually coordinate agents

---

### Method 3: Sequential Pipeline

```
1. Read scripts/agents/specflow-writer.md, create contracts for #42
2. Read scripts/agents/migration-builder.md, generate migrations for #42
3. Read scripts/agents/playwright-from-specflow.md, create tests for #42
4. Read scripts/agents/test-runner.md, run all tests
```

**Pros:**
- Step-by-step visibility
- Custom agent order

**Cons:**
- Much slower (no parallelism)
- Manual coordination required

---

### Method 4: Agent Teams (Recommended for large backlogs)

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true
```

Then:
```
Execute waves
```

**Pros:**
- Persistent context (agents fix their own bugs)
- Peer-to-peer coordination (less overhead)
- 3-5x faster than manual workflows
- Three-tier journey enforcement
- Migration conflict detection (db-coordinator)
- Automatic regression baseline tracking

**Cons:**
- Requires Claude Code 4.6+ with TeammateTool API
- Higher memory usage (persistent agent context)

**[Full Agent Teams Documentation -->](/agent-system/agent-teams/)**

---

## Agent Coordination Example

**Scenario:** Issue #42 "Add user authentication"

### Automatic Coordination (waves-controller)

```
User: Execute waves

waves-controller:
  → Phase 1: Analyze issue #42
  → Phase 2: Plan Wave 1 (1 issue)
  → Phase 3: Spawn agents in parallel
      - specflow-writer (#42)
      - migration-builder (#42)
      - edge-function-builder (#42)
      - frontend-builder (#42)
      - contract-test-generator (#42)
      - playwright-from-specflow (#42)
  → Phase 4: Agents work in parallel (3-4 minutes)
  → Phase 5: Run test-runner (validate)
  → Phase 6: Run contract-validator + journey-enforcer
  → Phase 7: Run ticket-closer (#42)
  → Phase 8: Report completion
```

**Total time:** 8-10 minutes

### Manual Coordination

```
User: Read scripts/agents/specflow-writer.md, create contracts for #42
      (2 min)
User: Read scripts/agents/migration-builder.md, generate migrations
      (2 min)
User: Read scripts/agents/edge-function-builder.md, create login RPC
      (3 min)
User: Read scripts/agents/frontend-builder.md, create LoginPage
      (3 min)
User: Read scripts/agents/contract-test-generator.md, create tests
      (2 min)
User: Read scripts/agents/playwright-from-specflow.md, create E2E tests
      (2 min)
User: Read scripts/agents/test-runner.md, run all tests
      (3 min)
User: Read scripts/agents/ticket-closer.md, close issue
      (1 min)
```

**Total time:** 18 minutes

**waves-controller is 2x faster** even on a single issue (parallelism + automation).

---

## Agent Customization

All agents are **prompt files** in `scripts/agents/*.md`. You can edit them.

**Example: Make specflow-writer always add contract tests**

```markdown
# scripts/agents/specflow-writer.md

## Process
...
6. Generate contract test for each invariant
   - Call contract-test-generator automatically
   - Output: src/__tests__/contracts/<name>.test.ts
```

Now specflow-writer also generates contract tests (previously separate agent).

**See:** [Customizing Agents](/agent-system/customizing/)

---

## Quick Reference Table

| Agent | Category | When Used | Time Saved |
|-------|----------|-----------|------------|
| **waves-controller** | Orchestration | "Execute waves" | 3-4x |
| **specflow-writer** | Contracts | Issue has acceptance criteria | 30-45 min |
| **contract-generator** | Contracts | Plain text specs | 20-30 min |
| **contract-test-generator** | Contracts | Contract needs test | 15-20 min |
| **migration-builder** | Implementation | Database changes needed | 20-30 min |
| **edge-function-builder** | Implementation | Backend logic needed | 45-60 min |
| **frontend-builder** | Implementation | UI needed | 45-60 min |
| **playwright-from-specflow** | Testing | Journey contract exists | 30-45 min |
| **journey-tester** | Testing | Cross-feature workflow | 45-60 min |
| **test-runner** | Testing | After code changes | 0 min (automated) |
| **e2e-test-auditor** | Testing | Check coverage | 20-30 min |
| **contract-validator** | Validation | After implementation | 15-20 min |
| **journey-enforcer** | Validation | Before release | 10-15 min |
| **ticket-closer** | Workflow | After completion | 5-10 min |
| **dependency-mapper** | Workflow | Before wave execution | 10-15 min |
| **board-auditor** | Workflow | Weekly health check | 30-45 min |
| **sprint-executor** | Workflow | Sprint start | 2-4x |
| **specflow-uplifter** | Workflow | Mid-project adoption | 2-3 hours |
| **issue-lifecycle** | Agent Teams | Per-issue full lifecycle | 4-6 hours |
| **db-coordinator** | Agent Teams | Migration conflict prevention | 30-60 min |
| **quality-gate** | Agent Teams | Three-tier test execution | 15-30 min |
| **journey-gate** | Agent Teams | Journey enforcement (3 tiers) | 20-30 min |
| **PROTOCOL** | Agent Teams | Communication reference | N/A (reference) |

**Total potential time savings per feature:** 4-6 hours → 20-30 minutes

---

## Next Steps

- **[waves-controller Deep Dive](/agent-system/waves-controller/)** — Understand 8-phase orchestration
- **[Agent Teams](/agent-system/agent-teams/)** — Persistent teammate coordination
- **[DPAO Methodology](/agent-system/dpao/)** — How parallel execution works
- **[Customizing Agents](/agent-system/customizing/)** — Edit agent prompts

---

## Compiler Analogy Reminder

> **TypeScript has tools: `tsc`, `ts-node`, `tslint`.**
>
> **Specflow has agents: 23+ specialized LLM workers.**

Each agent is a compiler for a specific domain.

**Compose them to automate architecture enforcement.**
