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

Before installing Specflow, ensure you have:

### Required Tools

| Tool | Version | Purpose | Check Command |
|------|---------|---------|---------------|
| **Claude Code** | Latest | Runs Specflow agents | `claude --version` |
| **Node.js** | 18.0.0+ | Agent execution environment | `node --version` |
| **Git** | Any recent | Repository operations | `git --version` |
| **GitHub CLI** | 2.0.0+ | Issue tracking integration | `gh --version` |

### Environment Setup

You need **Claude or OpenAI API access**:

```bash
# Option 1: Claude (recommended)
export ANTHROPIC_API_KEY=sk-ant-...

# Option 2: OpenAI
export OPENAI_API_KEY=sk-...
```

Add to `~/.bashrc` or `~/.zshrc` to persist.

### Repository Requirements

Specflow works best with:
- ✅ **GitHub repository** (for issue tracking)
- ✅ **Existing codebase** (Specflow adapts to your stack)
- ✅ **Test framework** (Playwright, Jest, Vitest, etc.)

**You don't need to rewrite your project.** Specflow integrates with what you have.

---

## Installation Methods

### Method 1: Extract to Existing Project (Recommended)

If you already have a project, extract Specflow into it:

```bash
# 1. Clone Specflow repository
git clone https://github.com/Hulupeep/Specflow.git /tmp/specflow

# 2. Navigate to your project
cd /path/to/your-project

# 3. Run extraction script
/tmp/specflow/scripts/extract-to-project.sh

# Output:
# ✓ Copying Specflow/ directory
# ✓ Copying scripts/agents/
# ✓ Creating docs/contracts/
# ✓ Updating .gitignore
# ✓ Installation complete!
```

This adds:
- `Specflow/` — Core methodology files
- `scripts/agents/` — 18 agent prompts
- `docs/contracts/` — Contract storage
- `.claude/hooks/` — Journey verification hooks
- Updated `.gitignore` — Excludes agent temp files

### Install Journey Verification Hooks (Recommended)

Hooks make Claude run E2E tests automatically at build boundaries:

```bash
# If using extract-to-project.sh, hooks are included
# If installing separately:
bash /tmp/specflow/install-hooks.sh /path/to/your-project

# Output:
# ✓ Created .claude/hooks/
# ✓ Installed .claude/settings.json
# ✓ Installed journey-verification.md
```

Then add configuration to your `CLAUDE.md`:

```markdown
## Test Configuration

- **Package Manager:** pnpm
- **Test Command:** `pnpm test:e2e`
- **Production URL:** `https://yourapp.com`
- **Deploy Platform:** Vercel
```

**Why hooks?** Without them, you ask "run tests" manually. You'll forget. Production breaks.

### Method 2: Fresh Project Setup

Starting from scratch:

```bash
# 1. Create new repository
mkdir my-project && cd my-project
git init

# 2. Clone Specflow
git clone https://github.com/Hulupeep/Specflow.git .

# 3. Install dependencies (if you want to run examples)
npm install
```

### Method 3: Manual Integration

For custom setups, copy just what you need:

```bash
# Copy agent prompts only
cp -r /path/to/Specflow/scripts/agents /your-project/scripts/agents

# Copy methodology docs
cp -r /path/to/Specflow/Specflow /your-project/Specflow

# Copy contract examples
cp -r /path/to/Specflow/docs/contracts /your-project/docs/contracts
```

---

## Verification

After installation, verify everything works:

```bash
# 1. Check Specflow directory exists
ls -la Specflow/
# Should show: CONTRACTS-README.md, CONTRACT-SCHEMA.md, etc.

# 2. Check agents directory exists
ls -la scripts/agents/
# Should show: waves-controller.md, specflow-writer.md, etc.

# 3. Verify Claude Code can read agents
claude --version
# Should output: Claude Code vX.X.X

# 4. Test GitHub CLI access
gh repo view
# Should show your repository details
```

### Optional: Run Verification Script

If you used Method 1, run the verification script:

```bash
./scripts/verify-setup.sh

# Output:
# ✓ Specflow/ directory found
# ✓ scripts/agents/ directory found (18 agents)
# ✓ docs/contracts/ directory found
# ✓ CLAUDE.md or .clauderc configured
# ✓ GitHub CLI authenticated
# ✓ API key configured (ANTHROPIC_API_KEY or OPENAI_API_KEY)
#
# ✅ Specflow is ready to use!
```

---

## Configuration

### 1. Add Specflow Instructions to Claude Code

Create or update `CLAUDE.md` in your project root:

```markdown
# Specflow Integration

This project uses Specflow contracts for architectural enforcement.

## Agent Execution Rules

1. **Check GitHub Issues First**: All work MUST have a GitHub issue
2. **Run Contract Tests**: `npm test -- contracts` before marking work complete
3. **Verify Journeys**: Critical journeys MUST pass before close
4. **Use Subagent Library**: Read `scripts/agents/*.md` before spawning agents

## Subagent Library

| Agent | Prompt File | When to Use |
|-------|------------|-------------|
| `waves-controller` | `scripts/agents/waves-controller.md` | User says "execute waves" |
| `specflow-writer` | `scripts/agents/specflow-writer.md` | Create contracts from issues |
| `contract-validator` | `scripts/agents/contract-validator.md` | Verify implementation matches spec |
| `test-runner` | `scripts/agents/test-runner.md` | Run tests after changes |

See `scripts/agents/README.md` for full agent list.
```

### 2. Update .gitignore (Already Done by Script)

Ensure these are ignored:

```gitignore
# Specflow temp files
.specflow-temp/
agent-outputs/
.agent-state/
```

### 3. Configure Test Commands

Add to `package.json`:

```json
{
  "scripts": {
    "test:contracts": "jest --testPathPattern=contracts",
    "test:e2e": "playwright test",
    "verify:all": "npm run test:contracts && npm run test:e2e"
  }
}
```

---

## What You Just Installed

### Specflow Methodology Files

Located in `Specflow/`:

| File | Purpose |
|------|---------|
| `CONTRACTS-README.md` | System overview, how contracts work |
| `CONTRACT-SCHEMA.md` | YAML format specification |
| `LLM-MASTER-PROMPT.md` | How LLMs should work with contracts |
| `SPEC-FORMAT.md` | Writing specifications |
| `CI-INTEGRATION.md` | GitHub Actions setup |

### Agent Prompts

Located in `scripts/agents/`:

| Agent | File | Role |
|-------|------|------|
| **waves-controller** | `waves-controller.md` | Orchestrates all other agents |
| **specflow-writer** | `specflow-writer.md` | Creates contracts from issues |
| **migration-builder** | `migration-builder.md` | Generates database migrations |
| **playwright-from-specflow** | `playwright-from-specflow.md` | Creates E2E tests from Gherkin |
| **contract-validator** | `contract-validator.md` | Verifies implementation |
| **test-runner** | `test-runner.md` | Executes tests, reports failures |
| ... | (18 total) | See `scripts/agents/README.md` |

### Contract Storage

Located in `docs/contracts/`:

This is where generated contracts will be stored:
- `feature_*.yml` — Architectural invariants
- `journey_*.yml` — End-to-end workflows

**Starts empty.** Agents populate it as you work.

---

## Next Steps

Now that Specflow is installed:

1. **[Run Your First Wave](/getting-started/first-wave/)** — Execute waves-controller on a GitHub issue
2. **[Understand Output](/getting-started/understanding-output/)** — Learn to read contracts and test results
3. **[Explore Agents](/agent-system/)** — Dive deeper into the 18 agent types

---

## Troubleshooting

### "command not found: claude"

**Solution:** Install Claude Code CLI:
```bash
npm install -g @anthropic-ai/claude-code
```

### "gh: command not found"

**Solution:** Install GitHub CLI:
```bash
# macOS
brew install gh

# Linux
sudo apt install gh

# Windows
winget install GitHub.cli
```

### "API key not configured"

**Solution:** Set environment variable:
```bash
export ANTHROPIC_API_KEY=sk-ant-...
# Add to ~/.bashrc or ~/.zshrc to persist
```

### "Specflow/ directory not found after extraction"

**Solution:** Run extraction script from correct directory:
```bash
cd /your-project  # Must be in project root
/tmp/specflow/scripts/extract-to-project.sh
```

---

## Compiler Analogy Reminder

> **TypeScript rejects type errors at compile time.**
> **Specflow rejects architecture errors at test time.**

Installation complete. Time to enforce some boundaries.
