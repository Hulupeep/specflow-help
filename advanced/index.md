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
- **[Manual Contract Creation](/advanced/manual-contracts/)** — Writing YAML contracts by hand
- **[Journey Testing](/advanced/journey-testing/)** — End-to-end user journey tests
- **CI/CD Integration** — Running Specflow in GitHub Actions

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

## Coming Soon

Full advanced content is under development. Check back soon for:

- Writing Contracts Manually (YAML authoring)
- Custom Agent Creation (prompt templates)
- GitHub Actions Integration
- Schema Reference

[View on GitHub](https://github.com/Hulupeep/Specflow)
