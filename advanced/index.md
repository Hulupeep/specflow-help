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

---

## The Two Enforcement Layers

Specflow uses **local hooks** AND **remote CI** together:

| Layer | Where | When | Speed |
|-------|-------|------|-------|
| **Hooks** | Your machine | Build, commit | Seconds |
| **CI** | GitHub | PR, merge | Minutes |

**Hooks** catch problems before you push. **CI** is the authoritative gate.

[View on GitHub](https://github.com/Hulupeep/Specflow)
