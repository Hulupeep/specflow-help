---
layout: default
title: Home
nav_order: 1
description: "Specflow enforces architectural contracts like a compiler enforces types."
permalink: /
---

# Stop Trusting. Start Verifying.
{: .fs-9 }

Specflow enforces architectural contracts like a compiler enforces types.
{: .fs-6 .fw-300 }

[Get Started](/getting-started/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 data-testid="get-started-btn" }
[View on GitHub](https://github.com/Hulupeep/Specflow){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## The Compiler Analogy

| Most Workflows | Specflow |
|----------------|----------|
| Intent → Prompt | Intent → Contract |
| → Hope | → Generate |
| → Review → Fix | → Test → Stop or Ship |
| **Trust the middle** | **Verify the boundary** |

### TypeScript rejects type errors. Specflow rejects architecture errors.

Before LLMs, humans were slow at writing code. We built tools to speed them up: IDEs, linters, autocomplete.

After LLMs, **humans are slow at reviewing code**. We need tools to enforce boundaries automatically.

**Specflow is that tool.**

---

## Why Now?

### Before LLMs (Pre-2023)
- ✅ Execution was deterministic
- ✅ Humans were the bottleneck (slow generation)
- ✅ Review scaled poorly but worked
- ✅ Violations were rare bugs

**Old tools were sufficient:** Design by Contract, TDD, linters, code review

### After LLMs (2023+)
- ⚠️ Generation is infinite
- ⚠️ Execution is probabilistic
- ⚠️ Humans cannot keep up (slow review)
- ⚠️ Drift is invisible until too late
- ⚠️ Violations are normal behavior

**Old tools are insufficient.** Specflow fills the gap.

---

## How It Works

Specflow uses **contracts** to define architectural rules:

```yaml
# Feature Contract Example
contract_type: feature
feature_name: leave_management
invariants:
  - id: LEAVE-001
    rule: "Leave approval MUST debit from leave_entitlements ledger"
    severity: critical
    enforcement: e2e_test
```

When code violates a contract, the build **fails** — just like a type error.

---

## The Agent-First Workflow

Specflow isn't just a framework. It's a **methodology** powered by 18 specialized agents:

1. **Define contracts** (what must hold true)
2. **Run `waves-controller`** (orchestrates agent execution)
3. **Agents generate** implementation + tests
4. **Tests enforce** contracts automatically
5. **Ship or stop** (no manual review needed)

**3-4x faster than manual workflows.** Proven on production projects.

---

## What You Get

- **Feature Contracts**: Architectural rules that must hold (invariants)
- **Journey Contracts**: End-to-end workflows that define "done"
- **Automated Testing**: Playwright E2E tests enforce contracts
- **Agent Orchestration**: DPAO methodology (Discovery → Parallel → Analysis → Orchestration)
- **Academic Foundation**: 40+ years of CS research (DbC, Property Testing, MDE)

---

## Quick Start

```bash
# 1. Add Specflow to your project
git clone https://github.com/Hulupeep/Specflow.git
cp -r Specflow/scripts/agents your-project/scripts/

# 2. Create GitHub issues with acceptance criteria (Gherkin)
# See: https://github.com/your-org/your-project/issues/new

# 3. Use Claude Code to invoke waves-controller agent
# Read scripts/agents/waves-controller.md
# Task("Execute waves", "{prompt content}", "general-purpose")

# 4. Agents autonomously:
# - Generate contracts from issues (specflow-writer)
# - Build database migrations (migration-builder)
# - Create E2E tests (playwright-from-specflow)
# - Verify contracts pass (test-runner)

# 5. Ship when contracts pass ✅
```

[Full Getting Started Guide →](/getting-started/)

---

## The Intellectual Foundation

> **"We already knew how to control untrusted execution. We just forgot — until LLMs forced us to remember."**

Specflow synthesizes:
- **Design by Contract** (Eiffel, 1986) — preconditions, postconditions, invariants
- **Property-Based Testing** (QuickCheck, 2000) — properties for systems
- **Static Analysis** (Lint, 1970s) — custom rules in YAML

**This isn't new theory.** It's old theory applied to a new failure mode: **probabilistic code generation**.

[Learn the Background →](/background/)

---

## The Compiler Doesn't Trust Your Types. Why Trust Your Architecture?

**Start enforcing boundaries today.**

[Get Started](/getting-started/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 data-testid="get-started-btn" }
