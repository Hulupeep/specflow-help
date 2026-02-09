---
layout: default
title: "Self-Healing Fix Loops"
parent: "Advanced"
nav_order: 5
permalink: /advanced/self-healing/
---

# Self-Healing Fix Loops
{: .no_toc }

Autonomous violation repair with confidence-tiered fix patterns.
{: .fs-6 .fw-300 }

> Self-healing fix loops adapted from [forge](https://github.com/ikennaokpala/forge) by [Ikenna N. Okpala](https://github.com/ikennaokpala). The concept of automated violation repair was adapted into Specflow's confidence-tiered fix pattern system.

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Problem

When contract tests fail, someone has to fix them. In a manual workflow, that means:

1. Read the violation message
2. Find the relevant contract
3. Understand the rule
4. Write a fix
5. Re-run tests
6. Repeat if still failing

With 10+ violations across a wave, this takes hours. The heal-loop agent automates steps 1-6.

---

## How the Heal-Loop Agent Works

The heal-loop agent follows a deterministic cycle:

```
Parse violation → Read contract → Generate fix → Apply → Re-test
       ↑                                                    │
       └────────────── Loop (max 3 iterations) ─────────────┘
```

### Step-by-Step

1. **Parse violation output** -- Extract the rule ID, file path, line number, and matched text from the test failure output (e.g., `CONTRACT VIOLATION: SEC-001 in src/config.ts:42`)
2. **Read the contract** -- Load the contract YAML and find the violated rule by ID
3. **Generate a fix** -- Use the contract's `auto_fix` hints, example_compliant code, and fix pattern store to produce a minimal patch
4. **Apply the fix** -- Edit the file with the smallest possible change
5. **Re-test** -- Run only the relevant contract test to verify the fix
6. **Report** -- If fixed, log the pattern. If still failing, try again (up to max iterations) or escalate

---

## What Gets Auto-Fixed vs. Escalated

Not all violations can be fixed automatically. The heal-loop agent classifies violations into two categories:

### Auto-Fixable Violations

| Violation Type | Example | Fix Strategy |
|----------------|---------|--------------|
| `forbidden_patterns` with `auto_fix` hint | Hardcoded secret detected | Replace with env var reference |
| `required_patterns` missing | Missing `authMiddleware` | Insert the required pattern |
| `forbidden_patterns` (simple) | `eval()` usage | Replace with safe alternative |
| Known fix pattern (high confidence) | `innerHTML` assignment | Replace with `textContent` |

### Escalated Violations (Human Required)

| Violation Type | Why It Cannot Be Auto-Fixed |
|----------------|----------------------------|
| Journey test failures | Require understanding of user flow intent |
| Build errors | May indicate deeper architectural issues |
| Logic errors | Contract tests catch patterns, not logic |
| Multiple interacting violations | Fix order matters; risk of cascading breaks |
| Low-confidence fix patterns | Score below Bronze threshold |

---

## Configuration

### Max Iterations

The heal-loop agent retries up to a configurable maximum before escalating:

```json
// .specflow/config.json
{
  "heal_loop": {
    "max_iterations": 3,
    "auto_fix_enabled": true
  }
}
```

**Default:** 3 iterations. After 3 failed attempts, the agent escalates with a detailed report of what it tried and why each attempt failed.

### Auto-Fix Hints in Contract YAML

Add `auto_fix` to any rule to guide the heal-loop agent:

```yaml
rules:
  non_negotiable:
    - id: SEC-001
      title: "No hardcoded secrets"
      scope:
        - "src/**/*.ts"
      behavior:
        forbidden_patterns:
          - pattern: /(password|api_key|secret)\s*[:=]\s*['"][^'"]+['"]/i
            message: "Hardcoded secret detected"
            auto_fix:
              strategy: "replace"
              hint: "Replace hardcoded value with process.env.VARIABLE_NAME"
              wrap_pattern: "process.env.${UPPER_SNAKE_CASE_NAME}"
```

### Auto-Fix Fields

| Field | Type | Description |
|-------|------|-------------|
| `strategy` | string | Fix approach: `replace`, `insert`, `wrap`, `remove` |
| `hint` | string | Human-readable instruction for the LLM to follow |
| `wrap_pattern` | string | Template for wrapping or replacing the matched text |

---

## Example: Auto-Fix in Action

### The Violation

```
CONTRACT VIOLATION: SEC-003
  File: src/components/UserProfile.tsx
  Line: 28
  Match: element.innerHTML = userData.bio
```

### The Contract Rule

```yaml
- id: SEC-003
  title: "No innerHTML with unsanitized content"
  scope:
    - "src/**/*.tsx"
  behavior:
    forbidden_patterns:
      - pattern: /\.innerHTML\s*=/
        message: "Direct innerHTML assignment (XSS risk)"
        auto_fix:
          strategy: "replace"
          hint: "Use textContent for plain text or a sanitization library for HTML"
```

### What the Heal-Loop Does

**Iteration 1:**
1. Reads violation: `SEC-003` in `src/components/UserProfile.tsx:28`
2. Reads contract: finds `auto_fix.hint` = "Use textContent..."
3. Checks fix pattern store: finds a Platinum-tier pattern for `.innerHTML` replacements
4. Generates fix: `element.textContent = userData.bio`
5. Applies fix to line 28
6. Re-runs `SEC-003` test -- passes

**Result:** Fixed in 1 iteration. Pattern score updated (+0.05).

---

## Relationship to waves-controller

The heal-loop agent is invoked during **Phase 6a** of the waves-controller pipeline:

```
Phase 5: Run tests
    ↓
Phase 6: Validate contracts
    ↓ (violations found?)
Phase 6a: Heal-loop (auto-fix cycle)
    ↓ (all fixed?)
Phase 7: Close issues
```

If the heal-loop cannot fix all violations within `max_iterations`, the waves-controller halts the wave and reports the unresolved violations.

---

## Confidence-Tiered Fix Patterns

The heal-loop learns from past fixes. Each fix pattern has a confidence score that determines whether it is applied automatically or requires human approval.

### Confidence Tiers

| Tier | Score Range | Behavior |
|------|-------------|----------|
| **Platinum** | 0.90 - 1.00 | Apply automatically, no confirmation needed |
| **Gold** | 0.70 - 0.89 | Apply automatically, log for review |
| **Silver** | 0.40 - 0.69 | Apply but re-test immediately; escalate if test fails |
| **Bronze** | 0.00 - 0.39 | Suggest to user but do not apply automatically |

### Score Evolution

- **Successful fix:** +0.05 to the pattern's confidence score
- **Failed fix (caused new violation):** -0.10 to the pattern's confidence score
- **Score decay:** -0.01 per 90 days of inactivity (patterns that are not used gradually lose confidence)
- **New patterns** start at 0.50 (Silver tier)

### Example Score Trajectory

```
Day 1:   Pattern created (innerHTML → textContent)     Score: 0.50 (Silver)
Day 1:   Applied successfully on 3 files                Score: 0.65 (Silver)
Day 5:   Applied successfully on 2 files                Score: 0.75 (Gold)
Day 12:  Applied successfully on 4 files                Score: 0.95 (Platinum)
Day 100: No usage, decay applied                        Score: 0.94 (Platinum)
```

---

## Fix Pattern Store

Fix patterns are stored in `.specflow/fix-patterns.json`:

```json
{
  "version": 1,
  "patterns": [
    {
      "id": "fix-sec-003-innerhtml",
      "rule_id": "SEC-003",
      "description": "Replace innerHTML assignment with textContent",
      "match": "\\.innerHTML\\s*=",
      "replacement_template": ".textContent =",
      "confidence": 0.95,
      "tier": "platinum",
      "applied_count": 12,
      "success_count": 12,
      "failure_count": 0,
      "last_used": "2025-11-20",
      "created": "2025-10-01"
    },
    {
      "id": "fix-sec-001-hardcoded-key",
      "rule_id": "SEC-001",
      "description": "Replace hardcoded API key with env var",
      "match": "api_key\\s*[:=]\\s*['\"][^'\"]+['\"]",
      "replacement_template": "process.env.${UPPER_SNAKE_CASE_NAME}",
      "confidence": 0.72,
      "tier": "gold",
      "applied_count": 8,
      "success_count": 7,
      "failure_count": 1,
      "last_used": "2025-11-15",
      "created": "2025-10-05"
    }
  ]
}
```

### Schema

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique pattern identifier |
| `rule_id` | string | Contract rule this pattern fixes |
| `description` | string | Human-readable description |
| `match` | string | Regex pattern that triggers this fix |
| `replacement_template` | string | Template for the replacement text |
| `confidence` | number | Current confidence score (0.00 - 1.00) |
| `tier` | string | Current tier: `platinum`, `gold`, `silver`, `bronze` |
| `applied_count` | number | Total times this pattern was applied |
| `success_count` | number | Times the fix resolved the violation |
| `failure_count` | number | Times the fix failed or caused new violations |
| `last_used` | string | ISO date of last application |
| `created` | string | ISO date when the pattern was first recorded |

---

## Setup

### 1. Copy the fix pattern store template

```bash
mkdir -p .specflow
cp Specflow/templates/fix-patterns.json .specflow/fix-patterns.json
```

### 2. Enable heal-loop in config

```json
// .specflow/config.json
{
  "heal_loop": {
    "max_iterations": 3,
    "auto_fix_enabled": true,
    "min_confidence_for_auto_apply": 0.40
  }
}
```

### 3. Add auto_fix hints to contracts (optional)

```yaml
forbidden_patterns:
  - pattern: /eval\s*\(/
    message: "eval() is forbidden"
    auto_fix:
      strategy: "replace"
      hint: "Use JSON.parse() for data or Function constructor alternatives"
```

---

## Model Tier

The heal-loop agent runs on **Opus** (the most capable model tier). This is because fix generation requires deep reasoning about code semantics, contract intent, and potential side effects.

See [Model Routing](/agent-system/model-routing/) for the full routing table.

---

## Related Pages

- [Contract Schema](/reference/contract-schema/) -- `auto_fix` field reference
- [Model Routing](/agent-system/model-routing/) -- Why heal-loop uses Opus
- [waves-controller](/agent-system/waves-controller/) -- Phase 6a integration
- [Security & Accessibility Gates](/core-concepts/security-accessibility/) -- Default SEC/A11Y rules that benefit most from auto-fix

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
