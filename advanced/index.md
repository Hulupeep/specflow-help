---
layout: default
title: Advanced
nav_order: 6
has_children: true
permalink: /advanced/
---

# Advanced Topics
{: .no_toc }

Manual setup, custom agents, and CI/CD integration.
{: .fs-6 .fw-300 }

---

## Overview

This section covers advanced workflows for power users:

- **[Journey Verification Hooks](/advanced/journey-verification-hooks/)** — Automatic E2E test execution
- **[CI/CD Integration](/advanced/ci-integration/)** — Running Specflow in GitHub Actions (fail-fast pipeline)
- **[Manual Contract Creation](/advanced/manual-contracts/)** — Writing YAML contracts by hand
- **[Journey Testing](/advanced/journey-testing/)** — End-to-end user journey tests
- **[Self-Healing Fix Loops](/advanced/self-healing/)** — Autonomous violation repair with confidence-tiered patterns
- **[Post-Mortem Learning System](/advanced/learning-system/)** — Violations get recorded, fixes get stored, agents get warned
- **[Simulation Layer](/advanced/simulation/)** — Deterministic replay proving contracts learn from violations
- **[CI Feedback Loop](/advanced/ci-feedback/)** — Automatic CI status reporting after every git push
- **[Drift Prevention](/advanced/drift-prevention/)** — Three enforcement layers (hooks + CI + deploy gate) that stop spec-to-code drift

**Note:** Most users should start with the [agent-first workflow](/getting-started/). Manual setup is 3-4x slower but gives more control.

---

## Journey Verification Hooks (Recommended)

**Problem:** You forget to run E2E tests. Production breaks.

**Solution:** Hooks make Claude run tests automatically at build boundaries.

```
[You run build]
[HOOK fires]
Claude: "Build passed. Running E2E tests against production..."
Claude: "18/20 passed. 2 failures. Fixing before commit."
```

See [Journey Verification Hooks](/advanced/journey-verification-hooks/) for setup.

---

## Quick Links

| Topic | What You'll Learn |
|-------|-------------------|
| [Journey Verification Hooks](/advanced/journey-verification-hooks/) | Make Claude run tests automatically at build/commit boundaries |
| [CI/CD Integration](/advanced/ci-integration/) | The fail-fast pipeline pattern with `needs: contract-tests` |
| [Manual Contract Creation](/advanced/manual-contracts/) | Write YAML contracts by hand when needed |
| [Journey Testing](/advanced/journey-testing/) | Create end-to-end user journey tests with Playwright |
| [Self-Healing Fix Loops](/advanced/self-healing/) | Autonomous violation repair with confidence-tiered fix patterns |
| [Post-Mortem Learning System](/advanced/learning-system/) | How violations and fixes become agent warnings |
| [Simulation Layer](/advanced/simulation/) | ReplayLab proves learning injection works — no vibes, only measurements |
| [CI Feedback Loop](/advanced/ci-feedback/) | Automatic CI status polling after git push |
| [Drift Prevention](/advanced/drift-prevention/) | Three enforcement layers that stop spec-to-code drift |

---

## The Three Enforcement Layers

Specflow uses **edit-time hooks**, **CI contract tests**, and **deploy gates** together:

| Layer | Where | When | Speed | Catches |
|-------|-------|------|-------|---------|
| **Edit-time hooks** | Agent's editor | Write/Edit tool call | Instant | Pattern violations in the file being edited |
| **CI contract tests** | GitHub Actions | Push/PR | Minutes | All pattern violations + journey regressions |
| **Deploy gate** | Production | After deploy | After deploy | Route availability, infrastructure, E2E |

**Hooks** catch violations while the agent is coding. **CI** is the merge gate. **Deploy gates** verify the live system.

See [Drift Prevention](/advanced/drift-prevention/) for the full framework with real-world examples.

[View on GitHub](https://github.com/Hulupeep/Specflow)
