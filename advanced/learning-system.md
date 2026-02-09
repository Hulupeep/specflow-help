---
layout: default
title: "Post-Mortem Learning System"
parent: "Advanced"
nav_order: 6
permalink: /advanced/learning-system/
---

# Post-Mortem Learning System
{: .no_toc }

Violations get recorded, fixes get stored, agents get warned before repeating mistakes.
{: .fs-6 .fw-300 }

> Post-mortem learning adapted from [forge](https://github.com/ikennaokpala/forge) by [Ikenna N. Okpala](https://github.com/ikennaokpala).

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Core Idea

Standard contract enforcement is reactive: tests fail, agents fix, tests pass. But agents have no memory of _why_ something failed or _what worked_. The same violation pattern repeats across sessions, repos, and team members.

The post-mortem learning system closes this loop:

```
Contract fails  -->  Violation recorded  -->  Fix applied  -->  Fix stored
      ^                                                             |
      |                                                             v
      +----------  Agent warned before repeating mistake  <---------+
```

**The shift:** Contracts go from "what is forbidden" to "what is forbidden AND what to do instead."

Specflow remains the enforcement authority. The learning layer (ruvector) never overrides contracts -- it provides remediation context alongside failures.

---

## How It Works End-to-End

### 1. Violation Capture

When a contract test fails, Specflow emits a structured **Violation Record**:

```json
{
  "event_type": "CONTRACT_VIOLATION",
  "contract_id": "AUTH-001",
  "rule_id": "AUTH-001",
  "signature": {
    "type": "forbidden_pattern",
    "match": "localStorage\\.setItem.*token",
    "ast_hash": "abc123"
  },
  "evidence": {
    "paths": ["src/auth/login.ts"],
    "snippets": ["localStorage.setItem('token', jwt)"],
    "line_numbers": [47]
  },
  "repo_context": {
    "repo": "myorg/myapp",
    "branch": "feature/auth",
    "commit": "def456"
  },
  "timestamp": "2024-01-15T10:00:00Z"
}
```

Every contract failure emits exactly one violation per violated `rule_id`. These are stored as artifacts for later analysis.

### 2. Fix Capture

When the agent fixes the violation and tests pass, a **Remediation Record** is emitted:

```json
{
  "event_type": "REMEDIATION_APPLIED",
  "contract_id": "AUTH-001",
  "violation_ref": "run_789/AUTH-001/1705312800",
  "fix_recipe": {
    "type": "patch",
    "before": "localStorage.setItem('token', jwt)",
    "after": "document.cookie = `token=${jwt}; HttpOnly; Secure`"
  },
  "validation": {
    "tests_passed": true,
    "contracts_passed": true
  },
  "timestamp": "2024-01-15T10:05:00Z"
}
```

A remedy is only stored if `validation.contracts_passed` is `true`. This prevents "superstitious" fixes (changes that appear to work but do not actually resolve the contract violation) from entering the knowledge base.

### 3. Guardrail Brief Injection

Before an agent writes code, the system queries stored patterns and injects a **Guardrail Brief** into the agent's context:

```markdown
## Guardrail Brief (3 items, low tension)

You are about to edit `controllers/auth.ts`.

**Known constraints:**
- AUTH-001: No localStorage for tokens (use httpOnly cookies)

**Proven fixes for similar edits:**
1. Replace `localStorage.setItem('token', ...)` with `res.cookie('token', ..., { httpOnly: true })`

**If you violate AUTH-001, the build will fail.**
```

The brief is injected during pre-flight, before the agent begins planning or editing files. This is the mechanism that prevents repeat violations.

---

## Memory Schema

All learning data lives under the `specflow/` namespace:

```
specflow/
  violations/
    AUTH-001/
      1704067200.json
      1704153600.json
    SEC-001/
      1704326400.json
  fixes/
    AUTH-001/
      1704067500.json
      1704154000.json
    SEC-001/
      1704327000.json
  patterns/
    AUTH-001.json
    SEC-001.json
```

### Violation Record

Stored on every contract test failure. Contains the contract ID, file path, matched pattern, code snippet with surrounding context, and the agent/session that produced it.

### Fix Record

Stored when a violation is resolved. Links back to the original violation via `violation_ref` and includes the before/after diff. Marked `verified: true` only after contract tests pass.

### Pattern Summary

Aggregated view of violations and fixes per contract rule. Updated periodically by the pattern aggregator.

```json
{
  "contract_id": "AUTH-001",
  "name": "No localStorage for auth tokens",
  "total_violations": 7,
  "total_fixes": 7,
  "success_rate": 1.0,
  "common_violations": [
    { "pattern": "localStorage.setItem('token', ...)", "occurrences": 5 },
    { "pattern": "localStorage.setItem('authToken', ...)", "occurrences": 2 }
  ],
  "proven_fixes": [
    {
      "description": "Use httpOnly cookie instead of localStorage",
      "example_before": "localStorage.setItem('token', jwt)",
      "example_after": "document.cookie = `token=${jwt}; HttpOnly; Secure; SameSite=Strict`",
      "success_count": 6
    }
  ],
  "last_updated": "2024-01-15T10:00:00Z"
}
```

---

## Min-Cut Gating

Not every situation needs the same amount of context. The min-cut gate measures how strongly the current edit connects to prior violation/remedy patterns and adjusts injection size accordingly.

### How It Works

The system builds a small graph for each agent run:

- **Left side (intent):** Touched paths, new imports, API calls detected, candidate contracts
- **Right side (memory):** Remediation recipes, prior incidents, ADR constraints

The min-cut between the intent cluster and the remediation library determines injection size:

| Cut Value | Tension | Injection Size | Action |
|-----------|---------|----------------|--------|
| High (well-connected) | Low | 1-3 remedies | Hints only |
| Medium | Medium | 5-7 remedies | Guidance |
| Low (fragile) | High | 10+ remedies | Full guardrails |
| Very low | Critical | All + escalation | Force stop, require human review |

### Default Behavior

For most edits, the gate injects **3 items or fewer** (low tension). The target is 90% of injections staying at 3 items or fewer, ensuring agents are not overwhelmed with context.

```typescript
interface GateDecision {
  injection_size: 3 | 5 | 10;
  mode: 'WARN' | 'BLOCK' | 'ESCALATE';
  reason: string;
}
```

**Modes:**
- **WARN** (default): Inject the brief, let the agent proceed
- **BLOCK**: Deterministic signature match found in diff -- halt until resolved
- **ESCALATE**: Fragile area with high historical failure rate -- request human review

---

## Confidence and Decay

Remediation recipes track their own reliability:

| Metric | Description |
|--------|-------------|
| `occurrences` | Total times this violation was seen |
| `applied_count` | Times this fix was applied |
| `success_count` | Times it resolved the violation |
| `success_rate` | `success_count / applied_count` |
| `confidence` | Overall reliability score (0.0 - 1.0) |

### Confidence Decay

Recipes that have not been used recently lose confidence over time:

```
effective_confidence = confidence * exp(-days_since_last_seen / 30)
```

This prevents stale recipes (e.g., a fix for Next.js 14 that may not work in Next.js 15) from being suggested with high confidence.

### Retrieval Ranking

When multiple recipes match a violation, they are ranked by:

1. Context match score (framework, language, path pattern)
2. Success rate
3. Recency
4. Confidence (with decay applied)

Only the top-ranked recipe per contract is returned to avoid conflicting suggestions.

---

## Integration with ruvector

The learning system communicates with ruvector via three endpoints:

| Endpoint | Direction | Purpose |
|----------|-----------|---------|
| `POST /violations` | Specflow --> ruvector | Store violation, link to signature |
| `POST /remediations` | Specflow --> ruvector | Store recipe (if valid), update metrics |
| `GET /guardrails?fingerprint=...` | ruvector --> Agent | Return ranked guardrail brief |

### Change Fingerprint

The fingerprint sent with guardrail queries describes the agent's current intent:

```json
{
  "paths_touched": ["src/auth/login.ts"],
  "imports_added": ["jsonwebtoken"],
  "api_calls_detected": ["localStorage.setItem"],
  "intent_tags": ["auth", "token-handling"]
}
```

This allows ruvector to return only relevant remedies rather than the entire knowledge base.

---

## Relationship to Self-Healing Fix Loops

The learning system and the [self-healing fix loops](/advanced/self-healing/) work together but serve different purposes:

| Concern | Learning System | Self-Healing Fix Loops |
|---------|-----------------|------------------------|
| **When** | Before writing code (preflight) | After contract tests fail |
| **Goal** | Prevent violations from happening | Fix violations that already occurred |
| **Mechanism** | Guardrail brief injection | Confidence-tiered auto-fix patterns |
| **Data source** | ruvector remediation library | `.specflow/fix-patterns.json` |

The two systems share data: when the heal-loop successfully fixes a violation, that fix is fed back into the learning system as a new remediation recipe. Over time, fixes that reach high confidence in the pattern store also appear as proven remedies in guardrail briefs.

---

## Rollout Phases

The learning system is designed for incremental adoption:

### Phase 1: Capture Only

- Deploy the violation parser
- Store violations in ruvector on every contract failure
- No changes to agent behavior

### Phase 2: Fix Tracking

- Deploy the fix tracker hook
- Store before/after diffs when violations are resolved
- Manual pattern aggregation

### Phase 3: Pattern Queries

- Deploy pre-generation queries
- Inject learned patterns into agent prompts before code generation
- Monitor pattern reuse rates

### Phase 4: Full Loop

- Automated pattern aggregation
- Swarm broadcasting (one fix benefits all agents)
- Min-cut gating for injection size control
- Metrics dashboard

---

## Configuration

### Environment Variables

```bash
# Enable/disable the learning system
SPECFLOW_MEMORY_ENABLED=true
SPECFLOW_MEMORY_NAMESPACE=specflow

# Retention periods
SPECFLOW_VIOLATION_RETENTION_DAYS=90
SPECFLOW_FIX_RETENTION_DAYS=365
SPECFLOW_PATTERN_REFRESH_HOURS=24

# Thresholds
SPECFLOW_MIN_FIXES_FOR_PATTERN=3
SPECFLOW_PATTERN_SUCCESS_THRESHOLD=0.8
```

### Hook Configuration

```yaml
# .claude-flow/hooks.yml
hooks:
  post-edit:
    - script: ./hooks/post-edit-specflow.sh
      enabled: true

  post-task:
    - script: ./scripts/specflow-learn.sh
      on: test:contracts
      enabled: true
```

---

## Success Criteria

| Criterion | Target |
|-----------|--------|
| Repeat violation rate reduction | >= 60% (measured per violation class per 50 runs) |
| Time-to-green reduction | >= 30% (median attempts to pass contracts) |
| Injection stays small | >= 90% of runs with 3 items or fewer |
| No regression | Contract pass rate >= baseline |

---

## Related Pages

- [Self-Healing Fix Loops](/advanced/self-healing/) -- Auto-fix with confidence-tiered patterns
- [Security & Accessibility Gates](/core-concepts/security-accessibility/) -- Default SEC/A11Y rules
- [Production Readiness Gates](/core-concepts/production-readiness/) -- PROD-001 through PROD-003
- [CI/CD Integration](/advanced/ci-integration/) -- Running contracts in GitHub Actions
- [CI Feedback Loop](/advanced/ci-feedback/) -- Post-push CI status reporting

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
