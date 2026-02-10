---
layout: default
title: Getting Started
nav_order: 2
has_children: true
permalink: /getting-started/
---

# Getting Started with Specflow
{: .no_toc }

Get up and running with Specflow in under 5 minutes.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Choose Your Path

**Fastest option:** Drop a single file into your project and start enforcing contracts immediately.

```bash
cp Specflow/SKILL.md your-project/
```

Then tell Claude Code: `/specflow` -- the skill activates the core methodology with security, accessibility, production readiness, and test integrity gates (adapted from [forge](https://github.com/ikennaokpala/forge) by Ikenna N. Okpala) included out of the box.

**[SKILL.md documentation](/getting-started/skill-file/)** -- what it includes, what it does not, and when to graduate to the full agent library.

---

## The Agent-First Approach

**SpecFlow isn't testing. It's containment.**

When your project outgrows SKILL.md, the full agent library lets **agents do the work**:

1. You define what matters (GitHub issues, requirements)
2. `waves-controller` orchestrates 23+ specialized agents
3. Agents generate contracts, migrations, tests, implementation
4. Tests enforce architectural boundaries automatically -- including forge-adapted quality gates for security, accessibility, and production readiness
5. You ship (or stop if contracts fail)

**This guide focuses on the agent-first workflow.** Manual setup is covered in [Advanced](/advanced/).

> **Agent Teams mode (default on Claude Code 4.6+):** Persistent peer-to-peer teammates activate automatically when TeammateTool is available. No environment variable needed. See [Agent Teams](/agent-system/agent-teams/) for details.

---

## Prerequisites

Before you start, ensure you have:

- **Node.js 18+** (for agent execution)
- **GitHub CLI** (`gh`) configured with authentication
- **Git** (for repository operations)
- **Claude or OpenAI API key** (agents require LLM access)

Check your environment:

```bash
node --version  # Should be 18.0.0 or higher
gh --version    # Should be 2.0.0 or higher
git --version   # Any recent version works
```

---

## Quick Start

The fastest path: add Specflow agents to your project and invoke via Claude Code.

```bash
# 1. Clone Specflow repository
git clone https://github.com/Hulupeep/Specflow.git

# 2. Copy agents to your project
cp -r Specflow/scripts/agents your-project/scripts/
cp Specflow/CLAUDE.md your-project/  # Optional: Agent invocation guide

# 3. Install journey verification hooks (recommended)
bash Specflow/install-hooks.sh your-project/
# Hooks make Claude run E2E tests automatically at build boundaries

# 3. Set up Claude Code (or your preferred LLM interface)
# Install: https://claude.ai/download

# 4. Create GitHub issues with Gherkin acceptance criteria
# Example issue format in Specflow/docs/contracts/

# 5. Invoke waves-controller agent via Claude Code:
# - Read scripts/agents/waves-controller.md for the agent prompt
# - Use Task tool to spawn agent with your specific request
# - Agent handles: discovery → contracts → implementation → tests → closure

# Output (from waves-controller):
# 🌊 Wave 1: Discovery
#   → Analyzing 12 open GitHub issues
#   → Calculating dependencies
#   → Priority scoring complete
#
# 🤖 Spawning Agents (Wave 1):
#   → specflow-writer: Creating contracts from issues #45, #46
#   → migration-builder: Database changes for #47
#   → playwright-from-specflow: E2E tests for #48
#
# ⚡ Parallel Execution (DPAO):
#   → 4 agents running concurrently
#   → Progress: ████████░░ 80%
#
# ✅ Wave 1 Complete:
#   → 4 contracts created
#   → 3 migrations generated
#   → 8 E2E tests written
#   → All tests passing
#
# 📊 Summary:
#   → Duration: 4m 23s
#   → Issues closed: 4/12
#   → Next wave: 8 issues ready
```

**That's it.** Specflow just:
1. Read your GitHub issues
2. Generated contracts defining what matters
3. Spawned agents to implement + test
4. Verified everything works

---

## What Just Happened?

### 1. waves-controller analyzed your project

It looked at:
- Open GitHub issues (scope, dependencies, acceptance criteria)
- Existing codebase (architecture, patterns, constraints)
- Contract definitions (what's already enforced)

### 2. It calculated dependency waves

Issues were grouped into waves based on:
- **Blockers**: What must complete before this can start
- **Priority**: Critical > Important > Future
- **Complexity**: Simple issues in early waves

### 3. It spawned specialized agents in parallel

Each agent has a specific role:
- **specflow-writer**: Creates contracts from requirements
- **migration-builder**: Generates database migrations
- **playwright-from-specflow**: Writes E2E tests from Gherkin
- **contract-validator**: Verifies implementation matches spec
- **test-runner**: Executes tests and reports failures

### 4. It verified contracts hold

Every contract is enforced by a test:
```yaml
invariant:
  id: LEAVE-001
  rule: "Leave approval MUST debit from leave_entitlements ledger"
  enforcement: e2e_test  # Playwright test runs automatically
```

If the test fails, the build fails. **No manual review needed.**

---

## Next Steps

- **[SKILL.md (Single File)](/getting-started/skill-file/)** -- Fastest path: drop one file, get instant contract enforcement
- **[Run Your First Wave](/getting-started/first-wave/)** -- Execute waves-controller on your own project
- **[Install Hooks](/advanced/journey-verification-hooks/)** -- Auto-run E2E tests at build boundaries
- **[Understand Contracts](/core-concepts/contracts/)** -- Learn what contracts enforce
- **[Team Workflows](/core-concepts/team-workflows/)** -- CSV journeys, local enforcement, CI compliance
- **[Explore Agents](/agent-system/)** -- Dive into the 23+ agent types
- **[Agent Teams](/agent-system/agent-teams/)** -- Persistent teammate coordination (Claude Code 4.6+)
- **[Read the Background](/background/)** -- Understand the academic foundation

---

## Getting Help

- **GitHub Issues**: [Report bugs or request features](https://github.com/Hulupeep/Specflow/issues)
- **Discussions**: [Ask questions and share experiences](https://github.com/Hulupeep/Specflow/discussions)
- **Documentation**: You're already here!

---

**Remember:** TypeScript rejects type errors. Specflow rejects architecture errors.

Start enforcing boundaries today.
