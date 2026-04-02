---
layout: default
title: Installation
parent: Getting Started
nav_order: 1
permalink: /getting-started/install/
---

# Installation
{: .no_toc }

Get Specflow running in your project in under 5 minutes.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

### Required Tools

| Tool | Version | Purpose | Check Command |
|------|---------|---------|---------------|
| **Claude Code** | Latest | Runs Specflow agents | `claude --version` |
| **Node.js** | 18.0.0+ | CLI and test execution | `node --version` |
| **Git** | Any recent | Repository operations | `git --version` |
| **GitHub CLI** | 2.0.0+ | Issue tracking integration | `gh --version` |
| **jq** | Any | Hook JSON parsing | `jq --version` |

### Repository Requirements

- ✅ **GitHub repository** (for issue tracking)
- ✅ **Existing codebase** (Specflow adapts to your stack)
- ✅ **Test framework** (Playwright, Jest, Vitest, etc.)

**You don't need to rewrite your project.** Specflow integrates with what you have.

---

## Installation Methods

### Method 1: npx (Recommended)

One command. No cloning. No samples. Just what your project needs.

```bash
cd /path/to/your-project
npx @colmbyrne/specflow init .
```

This creates:
- `docs/contracts/` — Default contract templates (security, accessibility, test integrity)
- `scripts/agents/` — 30+ agent prompts
- `tests/contracts/` — Contract schema tests
- `.claude/hooks/` — Journey verification, pipeline compliance, CI polling hooks
- `.git/hooks/commit-msg` — Rejects commits without issue numbers
- `.specflow/baseline.json` — Regression test baseline
- `package.json` — npm scripts for test/verify/compile (if not already present)

### Method 2: Clone + Setup Script

If you prefer a local Specflow source (for version checking and offline use):

```bash
# 1. Clone Specflow
git clone https://github.com/Hulupeep/Specflow.git /path/to/Specflow

# 2. Run setup from your project
cd /path/to/your-project
bash /path/to/Specflow/setup-project.sh .
```

### Method 3: Hooks Only

If you already have agents and contracts but need the hooks:

```bash
npx @colmbyrne/specflow update . --ci
```

Or from a local clone:

```bash
bash /path/to/Specflow/install-hooks.sh . --ci
```

The `--ci` flag also installs GitHub Actions workflows for PR compliance and post-merge audit.

---

## Verification

After installation, verify everything works:

```bash
npx @colmbyrne/specflow verify
```

This runs 13 sections of checks:

| Section | What it checks |
|---------|---------------|
| 1. Directory structure | `docs/contracts/`, test dirs exist |
| 2. Contract files | Count and list all contracts |
| 3. YAML validation | All contracts are valid YAML |
| 4. Test infrastructure | package.json, jest/vitest, test scripts |
| 5. CLAUDE.md | Exists, has contracts, no placeholders |
| 6. CI/CD | specflow-compliance.yml, specflow-audit.yml |
| 7. E2E setup | Playwright config and test count |
| 8. Hooks | All 5 scripts installed, executable, wired in settings.json |
| 9. Agent library | scripts/agents/, waves-controller present |
| 10. Fix patterns | .specflow/ state files |
| 11. Contract metadata | test_hooks.e2e_test_file paths resolve |
| 12. Graph validator | Cross-reference integrity |
| 13. Version check | Local hooks match Specflow source |

**Passing setup shows:**
- 20+ checks passed
- 0 failed
- Warnings are optional recommendations

---

## What Gets Installed

### Hooks (automatic enforcement)

| Hook | Trigger | What it does |
|------|---------|-------------|
| `post-build-check.sh` | Build / commit | Triggers journey tests |
| `run-journey-tests.sh` | Called by post-build | Maps issues → journeys → Playwright tests |
| `check-pipeline-compliance.sh` | Write / Edit | Catches contract violations in real-time |
| `post-push-ci.sh` | Push | Reports CI status (advisory) |
| `commit-msg` | Git commit | Rejects commits without `#issue-number` |

### Agent Prompts (30+)

| Agent | Role |
|-------|------|
| `waves-controller` | Master orchestrator — executes entire backlog |
| `specflow-writer` | Creates contracts from issues |
| `board-auditor` | Checks issue compliance |
| `pre-flight-simulator` | Simulates specs before code is written |
| `journey-gate` | Three-tier journey enforcement |
| ... | See `scripts/agents/README.md` for full list |

### Default Contracts

| Contract | What it catches |
|----------|----------------|
| `security_defaults.yml` | Hardcoded secrets, SQL injection, XSS, eval |
| `accessibility_defaults.yml` | Missing alt text, aria-labels, form labels |
| `test_integrity_defaults.yml` | Mocking in E2E tests, placeholder markers |
| `production_readiness_defaults.yml` | Demo data in production, placeholder domains |

---

## CLI Reference

```bash
npx @colmbyrne/specflow init .          # Full setup (safe to re-run)
npx @colmbyrne/specflow verify          # 13-section installation check
npx @colmbyrne/specflow update . --ci   # Update hooks + CI workflows
npx @colmbyrne/specflow audit 500       # Audit issue #500 for compliance
npx @colmbyrne/specflow graph           # Validate contract cross-references
npx @colmbyrne/specflow help            # Show all commands
```

---

## Updating

```bash
# Update hooks and CI workflows to latest
npx @colmbyrne/specflow update . --ci

# Check if anything is outdated
npx @colmbyrne/specflow verify
```

Section 13 of `verify` diffs your installed hooks against the latest Specflow source and shows what's outdated with severity and impact.

---

## Configuration

After installation, update `CLAUDE.md` with your project context:

```markdown
## Project Context

**Repository:** your-org/your-repo
**Project Board:** GitHub Issues
**Board CLI:** gh
**Tech Stack:** React, Node, Supabase
```

This is required — without it, Claude doesn't know your project and agents can't fetch issues.

---

## Troubleshooting

### "npx: command not found"

Node.js 18+ includes npx. Update Node: `nvm install 20`

### "gh: command not found"

Install GitHub CLI: `brew install gh` (mac) or `sudo apt install gh` (linux)

### "jq: command not found"

Install jq: `brew install jq` (mac) or `sudo apt install jq` (linux). Required for hook JSON parsing.

### Commit rejected unexpectedly

The `commit-msg` hook requires `#issue-number` in every commit. Bypass with `--no-verify` (not recommended).

### Pipeline compliance hook fires on example files

If `docs/journeys/` contains index or metadata CSVs, the hook may false-positive. Index files (`*INDEX*`, `*README*`) are now excluded automatically.

---

## Next Steps

1. **[Verify It's Working](/getting-started/verify-setup/)** — Confirm enforcement is active
2. **[Run Your First Wave](/getting-started/first-wave/)** — Execute waves-controller on your issues
3. **[Understand Output](/getting-started/understanding-output/)** — Read contracts and test results

---

[View on GitHub](https://github.com/Hulupeep/Specflow) | [npm package](https://www.npmjs.com/package/@colmbyrne/specflow)
