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

Specflow isn't just a framework. It's a **methodology** powered by 23+ specialized agents:

1. **Define contracts** (what must hold true)
2. **Run `waves-controller`** (orchestrates agent execution)
3. **Agents generate** implementation + tests
4. **Tests enforce** contracts automatically
5. **Ship or stop** (no manual review needed)

**3-4x faster than manual workflows.** Proven on production projects.

**New: [Agent Teams](/agent-system/agent-teams/)** — Persistent peer-to-peer teammates with three-tier journey gates (requires Claude Code 4.6+).

---

## What You Get

### Core Framework
- **Feature Contracts**: Architectural rules that must hold (invariants)
- **Journey Contracts**: End-to-end workflows that define "done"
- **Automated Testing**: Playwright E2E tests enforce contracts
- **Journey Verification Hooks**: Automatic E2E execution at build boundaries

### Quality Gates (Out of the Box)
- **[Security Gates](/core-concepts/security-accessibility/)**: OWASP Top 10 coverage — hardcoded secrets, SQL injection, XSS, eval, path traversal (SEC-001..005)
- **[Accessibility Gates](/core-concepts/security-accessibility/)**: WCAG AA basics — alt text, aria-labels, form labels, focus order (A11Y-001..004)
- **[Production Readiness](/core-concepts/production-readiness/)**: No demo data, placeholder domains, or hardcoded IDs in production (PROD-001..003)
- **[Test Integrity](/core-concepts/security-accessibility/)**: No mocking in E2E tests, no placeholder assertions, no swallowed errors (TEST-001..005)

### Self-Healing & Learning
- **[Self-Healing Fix Loops](/advanced/self-healing/)**: Autonomous violation repair with confidence-tiered fix patterns (Platinum → Bronze)
- **[Post-Mortem Learning](/advanced/learning-system/)**: Violations get recorded, fixes get stored, agents get warned before repeating mistakes
- **[CI Feedback Loop](/advanced/ci-feedback/)**: Automatic CI status reporting after every git push

### Agent System
- **[23+ Specialized Agents](/agent-system/)**: Complete delivery pipeline from spec to ship
- **[Model Routing](/agent-system/model-routing/)**: Haiku/Sonnet/Opus routing per agent — 40-60% cost savings
- **[Agent Teams](/agent-system/agent-teams/)**: Persistent peer-to-peer teammates with three-tier journey gates (Claude Code 4.6+)
- **DPAO Orchestration**: Discovery → Parallel → Analysis → Orchestration

### Portable Adoption
- **SKILL.md**: Single-file portable skill — drop one file into any project for instant Specflow
- **Full Agent Library**: 23+ agents for graduated adoption
- **Academic Foundation**: 40+ years of CS research (DbC, Property Testing, MDE)

---

## Journey Verification Hooks

**Problem:** You forget to run E2E tests. Production breaks.

**Solution:** Hooks make Claude run tests automatically.

| Without Hooks | With Hooks |
|---------------|------------|
| You: "Run tests" | [HOOK fires automatically] |
| You forget → prod breaks | Can't forget |
| "Tests passed" (vague) | WHERE/WHAT/HOW MANY (explicit) |

```bash
# Install hooks
bash install-hooks.sh /path/to/project
```

Hooks trigger at build boundaries:
- **PRE-BUILD** → Run baseline (LOCAL)
- **POST-BUILD** → Verify changes (LOCAL)
- **POST-COMMIT** → Verify production (PRODUCTION URL)

[Learn More →](/advanced/journey-verification-hooks/)

---

## Quick Start

### Option A: Single File (Fastest)

```bash
cp Specflow/SKILL.md your-project/
```

Then tell Claude Code: `/specflow` — the skill activates the core methodology with security, accessibility, and production readiness gates included.

### Option B: Full Agent Library

```bash
# 1. Add Specflow to your project
git clone https://github.com/Hulupeep/Specflow.git
cp -r Specflow/agents/ your-project/scripts/agents/

# 2. Copy default contract templates
cp Specflow/templates/contracts/*.yml your-project/docs/contracts/

# 3. Install hooks
bash Specflow/install-hooks.sh your-project/

# 4. Tell Claude Code
# "Execute waves"
```

### Option C: One Prompt (Zero Setup)

```
Read Specflow/README.md and set up my project with Specflow agents
including updating my CLAUDE.md. Then execute my backlog in waves.
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
