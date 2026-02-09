---
layout: default
title: "SKILL.md (Single File)"
parent: "Getting Started"
nav_order: 2
permalink: /getting-started/skill-file/
---

# SKILL.md: Single-File Specflow
{: .no_toc }

Drop one file into any project for instant contract enforcement.
{: .fs-6 .fw-300 }

> Single-file skill packaging inspired by [forge](https://github.com/ikennaokpala/forge) by [Ikenna N. Okpala](https://github.com/ikennaokpala).

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What is SKILL.md?

SKILL.md is a single-file portable Specflow skill (~442 lines) that gives any project instant access to the Specflow methodology. It is designed for Claude Code's `/skill` invocation system -- drop the file into your project root, and the skill is available immediately.

**One file. No dependencies. No setup.**

---

## What It Includes

SKILL.md packs the core Specflow methodology into a single document:

| Capability | Details |
|------------|---------|
| **Core loop** | Spec, Contract, Test, Code, CI Enforces |
| **Contract enforcement** | Feature contracts, journey contracts, severity levels |
| **Security gates** | All SEC-001 through SEC-005 patterns (hardcoded secrets, SQL injection, XSS, eval, path traversal) |
| **Accessibility gates** | All A11Y-001 through A11Y-004 patterns (alt text, aria-labels, form labels, tab order) |
| **Production readiness gates** | PROD-001 through PROD-003 (demo data, placeholder domains, hardcoded IDs) |
| **Test integrity gates** | TEST-001 through TEST-005 (no mocking in E2E, no silent failures, no placeholders) |
| **Condensed agent behaviors** | Core behaviors from key agents (specflow-writer, contract-validator, test-runner) |
| **Model routing** | Haiku/Sonnet/Opus routing recommendations per task type |
| **Self-healing fix patterns** | Confidence-tiered auto-fix with Platinum/Gold/Silver/Bronze tiers |
| **Compliance checklists** | Pre-edit checks for contracts, security, accessibility, and test integrity |

---

## What It Does NOT Include

SKILL.md is intentionally limited to keep the file portable and small. For advanced features, graduate to the full agent library.

| Feature | Requires |
|---------|----------|
| **Wave orchestration** | Full agent library (`scripts/agents/waves-controller.md`) |
| **GitHub issue management** | Full agent library + `gh` CLI |
| **23+ agent prompts** | Full agent library (`scripts/agents/*.md`) |
| **Agent Teams mode** | Full agent library + `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true` |
| **Three-tier journey gates** | Full agent library (quality-gate, journey-gate agents) |
| **Post-mortem learning** | Full agent library + ruvector integration |
| **CI feedback loop** | Full agent library + GitHub Actions setup |
| **Contract completeness enforcement** | Full agent library + `CONTRACT_INDEX.yml` |

---

## Installation

```bash
cp Specflow/SKILL.md your-project/
```

That is the entire installation.

---

## How to Invoke

Tell Claude Code to use the skill:

| Command | Effect |
|---------|--------|
| `/specflow` | Activate the Specflow methodology for the current session |
| `/specflow verify` | Run contract verification against the codebase |
| `/specflow spec` | Generate contracts from requirements or issue descriptions |
| `/specflow heal` | Attempt to auto-fix contract violations using fix patterns |

### Example Session

```
You: /specflow

Claude: Specflow skill activated. I will enforce architectural contracts
for this session. Security (SEC-001..005), accessibility (A11Y-001..004),
production readiness (PROD-001..003), and test integrity (TEST-001..005)
gates are active.

What would you like to work on?

You: Create a contract for user authentication

Claude: [Generates feature_authentication.yml with invariants,
compliance checklist, and enforcement methods]

You: /specflow verify

Claude: Scanning codebase against active contracts...
  SEC-001: PASS (no hardcoded secrets)
  AUTH-001: FAIL (password stored without bcrypt in src/auth/signup.ts:23)
  ...
```

---

## Graduation Path

SKILL.md is the entry point. As your project grows, graduate to more capable setups:

### Level 1: SKILL.md (Zero Setup)

```bash
cp Specflow/SKILL.md your-project/
```

**Best for:** Solo developers, small projects, trying Specflow for the first time.

**Capabilities:** Contract enforcement, security/accessibility/production/test gates, basic agent behaviors.

### Level 2: Full Agent Library (5 Minutes)

```bash
cp -r Specflow/agents/ your-project/scripts/agents/
cp Specflow/templates/contracts/*.yml your-project/docs/contracts/
bash Specflow/install-hooks.sh your-project/
```

**Best for:** Teams, multi-issue backlogs, projects with GitHub issue tracking.

**Adds:** 23+ specialized agents, wave orchestration, journey verification hooks, parallel execution.

### Level 3: Agent Teams (Environment Variable)

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true
```

**Best for:** Large backlogs (10+ issues), complex multi-feature waves, teams that need persistent agent context.

**Adds:** Persistent peer-to-peer teammates, db-coordinator for migration conflicts, three-tier journey gates, regression baseline tracking.

---

## When to Use SKILL.md vs Full Agents

| Scenario | Use SKILL.md | Use Full Agents |
|----------|-------------|-----------------|
| Quick prototype | Yes | No |
| Solo developer | Yes | Optional |
| Team project | Start here | Graduate when backlog grows |
| 1-3 issues | Yes | Optional |
| 10+ issues | No | Yes |
| Need wave orchestration | No | Yes |
| Need GitHub issue integration | No | Yes |
| Need journey gates | No | Yes |
| Need Agent Teams | No | Yes |

---

## Related Pages

- [Installation](/getting-started/install/) -- Full agent library setup
- [Your First Wave](/getting-started/first-wave/) -- Wave orchestration walkthrough
- [Agent Reference](/agent-system/agent-reference/) -- Complete list of 23+ agents
- [Agent Teams](/agent-system/agent-teams/) -- Persistent teammate coordination

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
