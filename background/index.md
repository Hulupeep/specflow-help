---
layout: default
title: Background
nav_order: 5
has_children: true
permalink: /background/
---

# Background & Academic Foundation
{: .no_toc }

> **"We already knew how to control untrusted execution. We just forgot — until LLMs forced us to remember."**

---

## The Intellectual Lineage

Specflow synthesizes 40+ years of computer science research:

### Design by Contract (Eiffel, 1986)
- **What it is:** Preconditions, postconditions, invariants
- **Why Specflow is different:** DbC assumes author = executor. Specflow assumes executor is untrusted.

### Property-Based Testing (QuickCheck, 2000)
- **What it is:** Define properties that should hold for all inputs
- **Why Specflow extends it:** QuickCheck tests functions. Specflow tests systems, architectures, workflows.

### Model-Driven Engineering (MDE, 1990s)
- **What it was:** Complete upfront models, rigid enforcement
- **Why Specflow works:** Partial models, selective enforcement, cheap generation

### Static Analysis (Lint, 1970s)
- **What it does:** Catch bugs before runtime
- **Why Specflow democratizes it:** Custom rules in YAML (not expert-only)

---

## Why Now?

LLMs changed the failure mode:

| Before LLMs | After LLMs |
|-------------|------------|
| Slow generation | Infinite generation |
| Deterministic execution | Probabilistic execution |
| Review scales | Review doesn't scale |
| Violations are bugs | Violations are normal |

**Old tools (DbC, TDD, linters) are insufficient.** Specflow fills the gap.

---

## Coming Soon

Full background content is under development. Check back soon for:

- Origins Story (Meyer → QuickCheck → Specflow)
- Academic Claims & Disclaimers
- The HCI Angle (formalizing editing)
- Research References

[View on GitHub](https://github.com/Hulupeep/Specflow)
