---
layout: default
title: Verify It's Working
parent: Getting Started
nav_order: 3
permalink: /getting-started/verify-setup/
---

# Verify It's Working
{: .no_toc }

After installation, confirm that Specflow is active and enforcing contracts automatically.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What's Automatic (No Prompting Needed)

Once Specflow is installed, **you do not need to tell Claude to use it.** These things happen automatically:

| Behavior | How It Works | You Say Nothing |
|----------|-------------|-----------------|
| **CLAUDE.md is read at session start** | Claude Code reads `CLAUDE.md` in the project root when a session begins. Contract rules, agent registries, and auto-trigger rules are loaded automatically. | You never say "use specflow." |
| **Contract checking on protected files** | When Claude modifies a file listed in a contract's `protected_files`, it checks the contract rules first. | You never say "check contracts." |
| **Hooks fire on build/commit** | Journey verification hooks run Playwright tests automatically after `pnpm build` or `git commit`. | You never say "run tests." |
| **Security/accessibility gates are active** | If default contract templates are installed (`docs/contracts/security_defaults.yml`, etc.), those patterns are enforced on every change. | You never say "check for security issues." |

**You just work normally.** The guardrails are there. If something violates a contract, the build fails and tells you why.

---

## Quick Verification Test

After installation, start a **new** Claude Code session in your project directory and ask:

```
What contracts are active in this project?
```

### Expected Response (Specflow is working)

Claude should list contracts from `docs/contracts/` with their IDs and rules:

```
Active contracts in this project:

Feature contracts:
  - feature_authentication.yml: AUTH-001 through AUTH-004
  - security_defaults.yml: SEC-001 through SEC-005
  - accessibility_defaults.yml: A11Y-001 through A11Y-004

Journey contracts:
  - journey_user_signup.yml: J-USER-SIGNUP (critical)
  - journey_checkout.yml: J-CHECKOUT (critical)

Total: 5 contracts, 18 invariants enforced.
```

### Unexpected Response (Specflow is NOT working)

If Claude says any of these, the setup is incomplete:

- "I don't see any contracts in this project."
- "What contracts are you referring to?"
- "I'm not familiar with Specflow."

**This means** `CLAUDE.md` is not being read, or contracts are not installed in `docs/contracts/`.

---

## What You Should NOT Need to Say

If Specflow is set up correctly, you should **never** need to prompt Claude with:

| Do Not Say This | Why It Means Setup Is Broken |
|-----------------|------------------------------|
| "Re-read CLAUDE.md" | Claude Code reads it automatically at session start |
| "Use specflow for this" | CLAUDE.md auto-trigger rules handle this |
| "Remember to check contracts" | Contract checking is triggered by protected file modification |
| "Follow the rules in CLAUDE.md" | Rules are loaded into context automatically |
| "Run the tests before closing" | Hooks and auto-trigger rules handle test execution |

If you find yourself saying these things, your `CLAUDE.md` configuration needs attention.

---

## If It's Not Working

Check these five things in order:

### 1. Contract section is at the TOP of CLAUDE.md

Claude Code attends most strongly to content at the beginning of files. If the contract rules are buried after 500 lines of other instructions, they may not be followed consistently.

**Fix:** Move the `## Architectural Contracts` section to the first major heading in `CLAUDE.md`.

### 2. File is named exactly `CLAUDE.md` in the project root

Not `claude.md`, not `Claude.md`, not `.claude.md`. The filename must be `CLAUDE.md` (all caps, in the project root directory).

```bash
ls -la CLAUDE.md
# Should show: -rw-r--r-- ... CLAUDE.md
```

### 3. You are in the right directory when starting the session

Claude Code reads the `CLAUDE.md` from the **current working directory** when the session starts. If you start Claude Code in a parent directory or a subdirectory, it reads a different `CLAUDE.md` (or none at all).

```bash
# Correct: start in the project root
cd /path/to/your-project
claude

# Wrong: starting from a parent directory
cd /path/to
claude  # Will not read your-project/CLAUDE.md
```

### 4. Contracts exist in `docs/contracts/*.yml`

The CLAUDE.md tells Claude to check contracts, but if no contract files exist, there is nothing to enforce. Install the default templates if you have not already:

```bash
# Check if contracts exist
ls docs/contracts/*.yml

# If empty, install defaults
cp Specflow/templates/contracts/*.yml docs/contracts/
```

### 5. Run the verification script

The `verify-setup.sh` script checks all 10 sections of a Specflow installation:

```bash
bash verify-setup.sh .

# Expected output:
# [1/10] CLAUDE.md exists .............. PASS
# [2/10] Contract section in CLAUDE.md . PASS
# [3/10] docs/contracts/ exists ........ PASS
# [4/10] Contract files found .......... PASS (3 contracts)
# [5/10] Agent prompts installed ....... PASS (23 agents)
# [6/10] Hooks installed ............... PASS
# [7/10] Test command configured ....... PASS
# [8/10] GitHub CLI authenticated ...... PASS
# [9/10] Node.js version .............. PASS (v20.11.0)
# [10/10] API key configured .......... PASS
#
# Result: 10/10 checks passed
```

---

## Commands That Trigger the Agent System

For projects with the full agent library installed, these explicit commands invoke specific agents:

| Command | What It Does |
|---------|-------------|
| `"Execute waves"` | Runs waves-controller: full backlog execution in dependency-ordered waves |
| `"Execute issues #50, #51, #52"` | Runs waves-controller for specific issues only |
| `"Execute waves for milestone v1.0"` | Runs waves-controller filtered by milestone |
| `"Run board-auditor"` | Audits which GitHub issues are specflow-compliant |
| `"Run e2e-test-auditor"` | Finds tests that silently pass when broken |
| `"Run test-runner"` | Executes tests and reports failures with file:line details |
| `"Run journey-enforcer"` | Checks journey coverage and release readiness |
| `"Run contract-validator"` | Verifies implementation matches contract spec |
| `/specflow` | Activates SKILL.md methodology (if SKILL.md installed) |
| `/specflow verify` | Runs contract verification scan against codebase |
| `/specflow spec` | Generates contracts from requirements |
| `/specflow heal` | Attempts auto-fix of contract violations |

These are **explicit** invocations. The automatic behaviors described above (CLAUDE.md reading, contract checking, hook firing) do not require any of these commands.

---

## The Litmus Test

**If Claude modifies a file in `src/` without mentioning contracts, the CLAUDE.md is not being read.**

When Specflow is working correctly, any modification to a protected file should produce output like:

```
Checking contracts before modifying src/features/auth/signup.ts...
  - feature_authentication.yml: AUTH-001 through AUTH-004
  - security_defaults.yml: SEC-001 through SEC-005
  No violations found. Proceeding with edit.
```

If you see Claude edit `src/` files with no mention of contracts, go back to the [troubleshooting checklist](#if-its-not-working) above.

---

## Related Pages

- [Installation](/getting-started/install/) -- Full installation guide
- [SKILL.md (Single File)](/getting-started/skill-file/) -- Minimal setup with one file
- [Your First Wave](/getting-started/first-wave/) -- Execute waves-controller on your project
- [Contracts](/core-concepts/contracts/) -- How contracts work in detail

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
