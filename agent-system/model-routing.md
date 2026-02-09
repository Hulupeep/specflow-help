---
layout: default
title: "Model Routing"
parent: "Agent System"
nav_order: 5
permalink: /agent-system/model-routing/
---

# Model Routing
{: .no_toc }

Route agents to the right model tier for cost efficiency without sacrificing quality.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why Model Routing?

Not every agent needs the most expensive model. A board-auditor that checks labels on GitHub issues does not require the same reasoning depth as a heal-loop agent generating code fixes from contract semantics.

Model routing assigns each agent to the cheapest model tier that can reliably perform its task. The result is a **40-60% reduction in token cost** with no measurable drop in output quality.

> Inspired by [forge](https://github.com/ikennaokpala/forge) by Ikenna N. Okpala, adapted from the TinyDancer model routing pattern by Mondweep Chakravorty.

---

## Model Tiers

| Tier | Model | Strengths | Cost |
|------|-------|-----------|------|
| **Haiku** | claude-haiku | Fast, cheap, good at structured checks and validation | Lowest |
| **Sonnet** | claude-sonnet | Strong code generation, contract writing, test creation | Medium |
| **Opus** | claude-opus | Deep reasoning, complex fixes, architectural decisions | Highest |

---

## Full Routing Table

### Haiku Tier (Lightweight Checks)

These agents perform structured validation, pattern matching, or simple report generation. They do not generate code or make architectural decisions.

| Agent | Purpose | Why Haiku |
|-------|---------|-----------|
| **board-auditor** | Check issue labels, staleness, compliance | Simple field checks |
| **contract-validator** | Verify patterns exist/absent in files | Regex matching and reporting |
| **journey-enforcer** | Check journey coverage and status | Read YAML, compare to test results |
| **test-runner** | Execute tests, report pass/fail | Run commands, parse output |
| **e2e-test-auditor** | Find unreliable or missing E2E tests | Compare test files to contracts |
| **ticket-closer** | Close issues with completion summary | Template-based GitHub API calls |

### Sonnet Tier (Code & Contract Generation)

These agents generate code, contracts, or tests. They need good code generation but not deep architectural reasoning.

| Agent | Purpose | Why Sonnet |
|-------|---------|------------|
| **waves-controller** | Orchestrate 8-phase pipeline | Coordination logic, not deep reasoning |
| **specflow-writer** | Generate contracts from GitHub issues | YAML generation from structured input |
| **specflow-uplifter** | Add Specflow to existing codebases | Code analysis and contract inference |
| **contract-generator** | Generate YAML from plain text specs | Structured output generation |
| **contract-test-generator** | Generate test files from contracts | Code generation from YAML |
| **dependency-mapper** | Parse issue dependencies, build graph | Text parsing and graph construction |
| **sprint-executor** | Execute sprint issues | Coordination, delegates to other agents |
| **migration-builder** | Generate SQL migrations | SQL generation from requirements |
| **frontend-builder** | Generate React components | Code generation from specs |
| **edge-function-builder** | Generate serverless functions | Code generation from contracts |
| **playwright-from-specflow** | Generate Playwright tests from journeys | Test code generation from YAML |
| **journey-tester** | Create cross-feature E2E tests | Test design and code generation |

### Opus Tier (Deep Reasoning)

These agents require understanding of code semantics, architectural intent, and potential side effects.

| Agent | Purpose | Why Opus |
|-------|---------|----------|
| **heal-loop** | Autonomous fix loop for contract violations | Must reason about fix correctness and side effects |

---

## Cost Savings

### Example: 9-Issue Wave Execution

Without model routing (all agents on Opus):

| Phase | Agents Invoked | Tokens (est.) |
|-------|---------------|---------------|
| Contracts | specflow-writer (x9) | ~180K |
| Implementation | migration + frontend + edge (x9) | ~540K |
| Testing | playwright + test-runner (x9) | ~270K |
| Validation | contract-validator + journey-enforcer | ~30K |
| Closure | ticket-closer (x9) | ~45K |
| **Total** | | **~1,065K tokens** |

With model routing:

| Phase | Tier | Tokens (est.) |
|-------|------|---------------|
| Contracts (Sonnet) | Sonnet | ~120K |
| Implementation (Sonnet) | Sonnet | ~360K |
| Testing (Sonnet + Haiku) | Mixed | ~150K |
| Validation (Haiku) | Haiku | ~10K |
| Closure (Haiku) | Haiku | ~15K |
| **Total** | | **~655K tokens** |

**Savings: ~38% fewer tokens, plus Haiku/Sonnet tokens cost less per token.**

Effective cost reduction: **40-60%** depending on wave composition.

---

## When to Use Each Tier

### Use Haiku When...

- The task is **read and report** (no code generation)
- The output is **structured and predictable** (pass/fail, labels, counts)
- The agent follows a **template** (close issue with summary, run tests and parse output)
- Speed matters more than nuance

### Use Sonnet When...

- The task involves **code generation** (TypeScript, SQL, YAML)
- The agent needs to **understand requirements** and produce artifacts
- The output needs to be **correct and idiomatic** but follows established patterns
- Most Specflow agents fall here

### Use Opus When...

- The task requires **reasoning about consequences** (will this fix break something else?)
- The agent must **understand intent** beyond pattern matching
- The fix involves **multiple interacting constraints**
- Currently, only the heal-loop agent needs this level

---

## Override Mechanism

Override the default model routing in `.specflow/config.json`:

```json
{
  "model_routing": {
    "overrides": {
      "specflow-writer": "opus",
      "board-auditor": "sonnet"
    }
  }
}
```

### When to Override

| Scenario | Override |
|----------|---------|
| Complex domain with nuanced contracts | `specflow-writer` -> `opus` |
| Large codebase with subtle patterns | `contract-validator` -> `sonnet` |
| Simple project with few contracts | `waves-controller` -> `haiku` |
| Debugging agent behavior | Any agent -> `opus` (temporarily) |

### Override Rules

- Overrides apply per-project, not globally
- You can only override to a **higher** tier (upgrading), or back to the **default** tier
- Downgrading below the recommended tier is allowed but may reduce output quality
- Override settings are checked into version control alongside your contracts

---

## Integration with waves-controller

The waves-controller reads model routing configuration when spawning agents:

```
waves-controller (Sonnet):
  → spawn specflow-writer (Sonnet)
  → spawn migration-builder (Sonnet)
  → spawn frontend-builder (Sonnet)
  → spawn contract-validator (Haiku)
  → spawn test-runner (Haiku)
  → spawn heal-loop (Opus)      ← Only if violations found
  → spawn ticket-closer (Haiku)
```

The waves-controller itself runs on Sonnet. It does not need Opus because its job is coordination and delegation, not deep reasoning.

---

## Related Pages

- [Agent Reference](/agent-system/agent-reference/) -- All 23+ agents and their roles
- [Self-Healing Fix Loops](/advanced/self-healing/) -- The Opus-tier heal-loop agent
- [DPAO Methodology](/agent-system/dpao/) -- How parallel execution works
- [Acknowledgments](/acknowledgments/) -- Attribution for TinyDancer and forge

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
