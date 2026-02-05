---
layout: default
title: Agent Teams
parent: Agent System
nav_order: 3
permalink: /agent-system/agent-teams/
---

# Agent Teams
{: .no_toc }

Persistent peer-to-peer teammates powered by Claude Code 4.6 TeammateTool API.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What are Agent Teams?

Agent Teams is Specflow's next-generation execution model built on the **Claude Code 4.6 TeammateTool API**. Instead of spawning short-lived subagents that report back to a hub, Agent Teams creates **persistent teammates** that communicate **peer-to-peer** and maintain context across their entire lifecycle.

**Think of it like a real development team:**
- Subagent mode: A manager hands out tasks and collects results (hub-and-spoke)
- Agent Teams mode: A team of developers who talk directly to each other (peer-to-peer)

---

## How It Differs from Subagent Mode

| Aspect | Subagent Mode (Default) | Agent Teams Mode |
|--------|------------------------|------------------|
| **Topology** | Hub-and-spoke (waves-controller dispatches) | Peer-to-peer (teammates coordinate directly) |
| **Context** | Stateless (each agent starts fresh) | Persistent (context preserved for issue lifetime) |
| **Communication** | Via parent (agents report to waves-controller) | Direct (teammates message each other) |
| **Bug fixing** | Parent must re-spawn agent with error context | Teammate sees own errors and self-repairs |
| **Parallelism** | Controlled by waves-controller | Self-coordinating (teammates negotiate) |
| **Speed** | 3-4x faster than manual | 3-5x faster than manual (less overhead) |

---

## Enabling Agent Teams

Set the environment variable before starting Claude Code:

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true
```

Or add it to your shell profile:

```bash
# ~/.bashrc or ~/.zshrc
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true
```

When this variable is **not set**, Specflow falls back to subagent mode automatically. All existing workflows continue to work unchanged.

---

## New Agents (5)

Agent Teams introduces 5 new specialized agents:

### issue-lifecycle

**Role:** Full lifecycle management for a single GitHub issue.

Unlike subagents that handle one phase (e.g., "write contracts" or "generate migration"), issue-lifecycle **owns an entire issue** from contracts through closure. It maintains persistent context, so when tests fail, it already knows the code it generated and can fix its own bugs without re-reading everything.

**Capabilities:**
- Reads issue, generates contracts, builds migrations, creates tests
- Runs tests and interprets failures
- Self-repairs: fixes its own bugs without re-spawning
- Requests help from teammates (e.g., asks db-coordinator for migration numbers)
- Reports completion to quality-gate

---

### db-coordinator

**Role:** Migration number management and conflict detection.

When multiple issue-lifecycle teammates work in parallel, they all need to create database migrations. db-coordinator is the **single source of truth** for migration numbering, preventing conflicts like two agents creating `migration_175_*.sql`.

**Capabilities:**
- Assigns sequential migration numbers on request
- Detects schema conflicts (two agents modifying the same table)
- Maintains a lock on the migration sequence
- Validates foreign key dependencies across parallel migrations

---

### quality-gate

**Role:** Test execution service for the team.

quality-gate runs contract tests, E2E tests, and build verification on behalf of the team. It operates at three tiers (see below) and determines whether work can proceed.

**Capabilities:**
- Executes `pnpm test -- contracts` (contract verification)
- Executes `pnpm test:e2e` (Playwright journey tests)
- Executes `pnpm build` (production build check)
- Compares results against `.specflow/baseline.json`
- Reports pass/fail to requesting teammates
- Blocks wave advancement on critical failures

---

### journey-gate

**Role:** Three-tier journey enforcement.

journey-gate implements a graduated testing strategy that matches test scope to execution phase. It ensures that issues are verified individually, waves are verified collectively, and the full suite passes before merge.

**Capabilities:**
- Tier 1: Per-issue journey tests (run after each issue completes)
- Tier 2: Cross-issue wave tests (run after wave completes)
- Tier 3: Full regression suite (run before merge to main)
- Tracks which journeys are affected by which issues
- Produces release readiness verdict

---

### PROTOCOL

**Note:** PROTOCOL is not an agent that executes. It is a **reference document** that defines the communication protocol all teammates use. Think of it as the "API specification" for inter-agent messaging.

**Purpose:**
- Defines all valid message types
- Specifies required fields per message
- Documents expected responses
- Ensures teammates can coordinate without ambiguity

---

## Three-Tier Journey Enforcement

Agent Teams introduces a graduated testing strategy:

### Tier 1: Issue Gate

**Scope:** Per-issue
**Run by:** issue-lifecycle (requests quality-gate to execute)
**When:** After an issue-lifecycle teammate finishes implementation
**Blocks:** Issue closure

```
issue-lifecycle (#42) completes work
  → Requests quality-gate: "Run tests for #42"
  → quality-gate runs: contract tests + journey tests for #42
  → Result: PASS → issue can be closed
  → Result: FAIL → issue-lifecycle self-repairs and retries
```

### Tier 2: Wave Gate

**Scope:** Cross-issue (all issues in current wave)
**Run by:** quality-gate
**When:** After all issues in a wave pass Tier 1
**Blocks:** Next wave from starting

```
All Wave 1 issues pass Tier 1
  → quality-gate runs: all contract tests + all journey tests
  → Compares against baseline (no regressions)
  → Result: PASS → Wave 2 can start
  → Result: FAIL → identify regressing issue, re-open for fix
```

### Tier 3: Regression Gate

**Scope:** Full test suite
**Run by:** quality-gate
**When:** After final wave completes, before merge
**Blocks:** Merge to main

```
All waves complete
  → quality-gate runs: full test suite (contracts + E2E + build)
  → Compares against baseline
  → Result: PASS → safe to merge
  → Result: FAIL → block merge, report regressions
```

---

## TeammateTool API

Agent Teams communicates using three primitives from the Claude Code 4.6 TeammateTool API:

### write (Direct Message)

Send a message to a specific teammate:

```javascript
TeammateTool.write({
  to: "db-coordinator",
  message: "REQUEST_MIGRATION: issue #42, table: audit_log"
})
```

**Use cases:**
- Requesting migration numbers from db-coordinator
- Reporting completion to quality-gate
- Asking another issue-lifecycle for schema info

### broadcast (All Teammates)

Send a message to all active teammates:

```javascript
TeammateTool.broadcast({
  message: "TOUCHING_FILE: src/features/auth/LoginPage.tsx"
})
```

**Use cases:**
- Announcing file modifications (prevent merge conflicts)
- Reporting blocking failures
- Sharing discovered information (e.g., "this RPC signature changed")

### Shared TaskList

All teammates share a TaskList for tracking work items:

```javascript
TaskList.update({
  issueNumber: 42,
  status: "implementing",
  agent: "issue-lifecycle-42",
  phase: "migration-generation"
})
```

**Use cases:**
- Tracking which issues are in progress
- Preventing duplicate work
- Providing progress visibility to waves-controller

---

## Communication Protocol

All inter-agent messages follow a structured protocol. Each message type has a defined sender, receiver, and payload:

| Message | Sender | Receiver | Purpose |
|---------|--------|----------|---------|
| `REQUEST_MIGRATION` | issue-lifecycle | db-coordinator | Request next migration number |
| `MIGRATION_ASSIGNED` | db-coordinator | issue-lifecycle | Return assigned migration number |
| `MIGRATION_CONFLICT` | db-coordinator | issue-lifecycle | Alert: schema conflict detected |
| `TOUCHING_FILE` | issue-lifecycle | broadcast | Announce file being modified |
| `FILE_CONFLICT` | any | issue-lifecycle | Alert: another agent modifying same file |
| `RUN_CONTRACTS` | issue-lifecycle | quality-gate | Request contract test execution |
| `RUN_JOURNEYS` | issue-lifecycle | quality-gate | Request journey test execution |
| `RUN_WAVE_GATE` | waves-controller | quality-gate | Request cross-issue validation |
| `RUN_REGRESSION` | waves-controller | quality-gate | Request full regression suite |
| `TEST_RESULTS` | quality-gate | requester | Return test execution results |
| `ISSUE_COMPLETE` | issue-lifecycle | waves-controller | Report issue finished |
| `ISSUE_BLOCKED` | issue-lifecycle | waves-controller | Report issue blocked |
| `WAVE_APPROVED` | quality-gate | waves-controller | Wave passed all gates |
| `WAVE_REJECTED` | quality-gate | waves-controller | Wave failed quality gate |
| `SCHEMA_INFO` | issue-lifecycle | issue-lifecycle | Share schema discovery between peers |
| `DEFER_TEST` | issue-lifecycle | quality-gate | Defer a failing test to journal |
| `BASELINE_UPDATE` | quality-gate | broadcast | Announce baseline.json updated |

### Message Format

```json
{
  "type": "REQUEST_MIGRATION",
  "from": "issue-lifecycle-42",
  "to": "db-coordinator",
  "payload": {
    "issueNumber": 42,
    "tableName": "audit_log",
    "operation": "CREATE TABLE"
  },
  "timestamp": "2026-02-05T14:30:00Z"
}
```

---

## Backward Compatibility

Agent Teams is **fully backward compatible**. When `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` is not set:

- waves-controller uses the standard subagent spawning model
- All existing agent prompts in `scripts/agents/*.md` work unchanged
- No new infrastructure files are required
- The 18 original agents operate identically

**Upgrade path:**
1. Set the environment variable
2. Add the 5 new agent prompt files to `scripts/agents/`
3. waves-controller auto-detects and switches to team mode
4. Remove the env var to revert at any time

---

## Infrastructure Files

Agent Teams introduces three infrastructure files:

### `.specflow/baseline.json`

Stores the current test baseline for regression detection:

```json
{
  "timestamp": "2026-02-05T14:00:00Z",
  "contractTests": {
    "total": 42,
    "passing": 42,
    "failing": 0
  },
  "e2eTests": {
    "total": 18,
    "passing": 16,
    "failing": 2,
    "knownFailures": ["journey_whatsapp_alert", "journey_payroll_export"]
  },
  "build": "passing"
}
```

**Purpose:** quality-gate compares current results against this baseline to detect regressions. Known failures are tracked so they do not block new work.

### `.claude/.defer-journal`

Tracks deferred test failures that are not blocking:

```
2026-02-05T14:30:00Z | issue-lifecycle-42 | DEFER | journey_whatsapp_alert | Known failure, not related to #42
2026-02-05T15:00:00Z | issue-lifecycle-43 | DEFER | contract_CSV-003 | Flaky test, tracked in #99
```

**Purpose:** When a teammate encounters a test failure that is unrelated to their work, they log it here instead of blocking. The journal is reviewed during Tier 3 (Regression Gate).

### `scripts/compare-baseline.js`

Utility script that compares current test results against the baseline:

```bash
node scripts/compare-baseline.js --baseline .specflow/baseline.json --results test-results.json
```

**Output:**
```
Baseline comparison:
  Contract tests: 42/42 → 44/44 (+2 new, 0 regressions)
  E2E tests: 16/18 → 17/18 (+1 fixed, 0 regressions)
  Build: passing → passing

Verdict: NO REGRESSIONS (safe to proceed)
```

**Purpose:** Automates the comparison logic so quality-gate can make pass/fail decisions objectively.

---

## Example Workflow

Here is how a typical wave executes in Agent Teams mode:

```
1. waves-controller detects CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true
2. Phase 1 (Discovery): Same as subagent mode
3. Phase 2 (Team Spawning):
   - Spawn issue-lifecycle for each Wave 1 issue
   - Spawn db-coordinator (1 instance)
   - Spawn quality-gate (1 instance)
   - Spawn journey-gate (1 instance)

4. Teammates self-coordinate:
   issue-lifecycle-42: "REQUEST_MIGRATION for audit_log"
   db-coordinator: "MIGRATION_ASSIGNED: 176"
   issue-lifecycle-42: "TOUCHING_FILE: src/features/auth/LoginPage.tsx"
   issue-lifecycle-43: (sees broadcast, avoids that file)
   issue-lifecycle-42: "RUN_CONTRACTS for #42"
   quality-gate: "TEST_RESULTS: 4/4 passed"
   issue-lifecycle-42: "ISSUE_COMPLETE: #42"

5. waves-controller: "RUN_WAVE_GATE"
   quality-gate: runs all tests, compares baseline
   quality-gate: "WAVE_APPROVED"

6. waves-controller proceeds to Wave 2

7. After final wave:
   waves-controller: "RUN_REGRESSION"
   quality-gate: full suite passes
   quality-gate: "WAVE_APPROVED" (regression gate)

8. Merge to main
```

---

## Next Steps

- **[Agent Reference](/agent-system/agent-reference/)** -- All 23+ agents including team agents
- **[waves-controller](/agent-system/waves-controller/)** -- How waves-controller adapts for teams
- **[DPAO Methodology](/agent-system/dpao/)** -- Parallel execution foundation

---

## Compiler Analogy Reminder

> **A build system has workers: parallel compilation units.**
>
> **Specflow Agent Teams has teammates: persistent peer-to-peer agents.**

Same pattern. Persistent context. Direct coordination.
