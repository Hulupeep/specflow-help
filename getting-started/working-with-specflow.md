---
layout: default
title: "Working With Specflow"
parent: "Getting Started"
nav_order: 4
permalink: /getting-started/working-with-specflow/
---

# Working With Specflow
{: .no_toc }

How to talk to Claude once Specflow is set up, how to sense-check it is working as you go, and what status tools are available.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Setup, Verify, Work

The workflow is three steps. After the first two, you stop thinking about Specflow and just work on your project.

### 1. Set up

Install hooks, agents, contracts, and update your CLAUDE.md. This is covered in [Installation](/getting-started/install/) and [Verify It's Working](/getting-started/verify-setup/).

### 2. Verify

Before diving into real work, ask Claude two questions to confirm it loaded the Specflow context from CLAUDE.md:

```
"What contracts are active in this project?"
```

Claude should list contracts from `docs/contracts/`. If it says "I don't see any contracts" or asks what you mean, your CLAUDE.md is not being read.

```
"Are journey contracts being executed? Are hooks running for specflow?"
```

Claude should check `.claude/hooks/` and `docs/contracts/journey_*.yml` and tell you their status. If it has no idea what you are talking about, go back to installation.

### 3. Work

Once verified, just talk naturally about your project. You do not keep telling Claude to "use specflow." The CLAUDE.md is loaded at session start and stays active for the entire session. Every file edit, every test run, every issue closure will follow the rules automatically.

---

## Natural Language Works

Once Specflow is set up, you talk in workflow terms, not tool terms. You describe what you want to happen, and Claude selects the right agents and processes.

### These work

| What you say | What happens |
|:-------------|:-------------|
| `"continue planning next waves, specflow auditing the board in that planning"` | Claude runs board-auditor as part of wave planning, produces a compliance matrix alongside the dependency graph |
| `"execute waves"` | Full backlog execution: dependency mapping, parallel batching, agent spawning, contract enforcement, test gates, issue closure |
| `"what's failing?"` | Claude runs tests, reports structured results with file paths and line numbers |
| `"are we release-ready?"` | Claude checks journey coverage, contract compliance, and Definition of Done status across all open issues |
| `"make issues #126-#128 specflow-compliant"` | Runs specflow-writer and specflow-uplifter to add Gherkin, contracts, data-testid requirements, and test file references to those issues |

### You should NOT need to say

- "Use specflow for this"
- "Remember to check contracts before editing"
- "Run the board-auditor agent"
- "Follow the rules in CLAUDE.md"

If you find yourself saying any of these, something is wrong with your setup. The whole point is that Specflow rules are enforced automatically through CLAUDE.md, not through manual reminders.

---

## Sense-Check As You Go

This is the most important section on this page. Trust but verify. Periodically confirm Claude is actually operating under Specflow rules, especially at the start of new sessions or after long pauses.

### After setup

| Ask this | Expected response |
|:---------|:------------------|
| `"What contracts are active in this project?"` | Lists contracts from `docs/contracts/`, showing filenames and what they protect |
| `"Are journey contracts being executed? Are hooks running for specflow?"` | Checks `.claude/hooks/` directory and `docs/contracts/journey_*.yml`, reports their status |

If Claude responds with confusion or generic answers, your CLAUDE.md is not in the right place or is not formatted correctly.

### During wave execution

| Ask this | Expected response |
|:---------|:------------------|
| `"show me wave status"` | Produces the Wave Execution Progress table with status indicators for each issue: Complete, In Progress, Ready, Blocked, Failed |
| `"what did wave 2 close?"` | Shows the Phase 8 completion report for that wave: issues closed, contracts generated, migrations applied, test results, commits made |
| `"run board-auditor"` | Compliance matrix showing COMPLIANT, PARTIAL, and NON-COMPLIANT counts with per-issue breakdown of what is missing |

### After implementation

| Ask this | Expected response |
|:---------|:------------------|
| `"what tests are failing?"` | Structured report following the mandatory format: WHERE (local/production URL), WHICH (test file names), HOW MANY (pass/fail/skip counts), SKIPPED explained |
| `"are critical journeys passing?"` | Definition of Done status check across journey contracts |

You should also confirm passively: when Claude finishes implementing something and says "done," check whether it mentions running contract tests. If it modified a file under `src/` and did not mention contracts at all, the CLAUDE.md is not being read.

### The litmus test

> If Claude modifies a file in `src/` without mentioning contracts, the CLAUDE.md is not being read.

This is the single most reliable indicator. Contract checking is baked into the CLAUDE.md rules as a mandatory step before any protected file modification. If Claude skips it, the rules are not loaded.

---

## The Status Board

Specflow provides several status tracking tools that give you visibility into execution progress, compliance, and test health.

### Wave Progress Report

Shows current wave, next waves, dependency chains, parallel opportunities, and completion statistics. This is your high-level dashboard for where execution stands.

**How to ask for it:**

```
"show me wave status"
"where are we?"
"what's the progress?"
```

The report includes a table with each issue, its status indicator, dependencies, and which agent is working on it (if execution is in progress).

### Board Audit Matrix

A compliance breakdown of every open issue on the board. Each issue is scored as:

- **COMPLIANT** (5/5) -- Has all required elements: Gherkin acceptance criteria, SQL contracts, RLS policies, contract references, journey mapping
- **PARTIAL** (3-4/5) -- Missing some elements, listed per issue
- **NON-COMPLIANT** (0-2/5) -- Missing most required elements

**How to ask for it:**

```
"run board-auditor"
"audit the board"
"how compliant is the backlog?"
```

The matrix shows exactly what each non-compliant issue is missing, so you can fix gaps before execution.

### Phase 8 Completion Report

This appears automatically at the end of each wave. It summarizes:

- Issues closed in this wave
- Contracts generated
- Migrations applied
- Test results (pass/fail/skip)
- Commits created (with issue references)
- Journey coverage delta

You can also ask for it retroactively: `"what did wave 3 produce?"` or `"show me the wave 2 completion report."`

### Task List

Real-time progress during execution. This uses Claude Code's internal task tracking (TaskCreate/TaskUpdate) to show which agents are running, which are blocked on dependencies, and which have completed. You see this in the status line of your terminal as execution proceeds.

You do not usually need to ask for this directly -- it updates automatically. But you can check it at any time by asking `"what's running right now?"` or `"show me the task list."`

---

## When Something Feels Wrong

If Specflow does not seem to be working, here are the most common issues and what to check.

| Symptom | Likely cause | What to check |
|:--------|:-------------|:--------------|
| Claude closes work without running tests | CLAUDE.md not being read | Check that CLAUDE.md is in the project root and follows the correct format. The Specflow rules section must be present and correctly formatted. |
| Claude does not mention contracts when editing `src/` files | Contract section not prominent in CLAUDE.md | Move the contract rules section higher in CLAUDE.md. Put it before other sections so it is read first. |
| Hooks do not fire after build or commit | Hook configuration missing | Check `.claude/settings.json` for hook entries and verify `.claude/hooks/` directory contains the hook scripts. Run `bash Specflow/install-hooks.sh .` to reinstall. |
| Wave reports are missing or incomplete | waves-controller agent not loaded | Verify `scripts/agents/waves-controller.md` exists in your project. If missing, copy from the Specflow repository. |
| Board audit shows all NON-COMPLIANT | Contracts not installed | Check `docs/contracts/` directory. If empty, contracts have not been generated yet. Run specflow-writer against your issues to create them. |
| Claude asks "what is Specflow?" at session start | CLAUDE.md not found or not in project root | Ensure the file is named exactly `CLAUDE.md` (case-sensitive) and is in the root of your working directory. |

If none of these solve the problem, start a fresh Claude session and immediately ask `"What contracts are active?"` as your first message. This confirms whether CLAUDE.md is being loaded at all.
