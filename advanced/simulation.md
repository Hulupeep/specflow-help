---
layout: default
title: "Simulation Layer"
parent: "Advanced"
nav_order: 7
permalink: /advanced/simulation/
---

# Simulation Layer
{: .no_toc }

Deterministic replay that proves contracts learn from violations — no vibes, everything measurable.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why Simulation Exists

Static contracts catch violations. Agents fix them. Tests pass. Then the same violation happens again — in a different file, a different session, a different agent. Nothing was learned.

**The real cost is the 50th repeat of the same mistake.**

The simulation layer closes this gap by proving, under controlled replay, that the learning injection system actually reduces repeat violations and accelerates time-to-green. It does not rely on LLM behavior. It does not measure "feels smarter." It measures quantitative improvement with deterministic fixtures.

---

## Why This Belongs in Specflow

Specflow already owns the enforcement boundary. It sees every violation (contract ID, signature, file, pattern) and every fix (before/after diff, validation result). That makes it the natural place to close the feedback loop:

1. **Specflow sees every violation** — structured data that no other tool has
2. **Specflow sees every fix** — when tests pass after a violation, Specflow knows the before and after
3. **The contract runner is the integration point** — event emission requires zero new infrastructure
4. **Contracts stay pure** — Specflow remains the "compiler" (what is forbidden). ruvector becomes the "lint + autofix memory" layer. No contract YAML bloat.

Putting learning anywhere else would require duplicating Specflow's violation detection logic. That is the wrong abstraction boundary.

---

## ReplayLab: The Deterministic Harness

ReplayLab is the core of the simulation layer. It proves the learning loop works without any LLM involvement.

### Fixture Structure

```
fixtures/
  repos/
    minimal-nextjs/        # tiny Next.js app
    minimal-express/       # tiny Express app
    minimal-react/         # tiny React SPA
  contracts/
    AUTH-001.yml           # localStorage token
    LAYER-001.yml          # db in controller
    SEC-001.yml            # SQL injection
    ... (10 total)
  violations/
    AUTH-001/
      variant-a.diff       # localStorage.setItem('token', ...)
      variant-b.diff       # localStorage.token = ...
      variant-c.diff       # window.localStorage.setItem(...)
    ... (30 total, 3 variants per contract)
  remedies/
    AUTH-001/
      httponly-cookie.json
    ...
  expected/
    fingerprints/
      AUTH-001-variant-a.json
    guardrails/
      AUTH-001-variant-a.json
```

### What ReplayLab Proves

| Test | What It Validates |
|------|-------------------|
| Fingerprint extraction | Same diff always produces same fingerprint |
| Signature matching | Violations of the same type match the same signature |
| Retrieval ranking | Highest-confidence + closest-context recipe ranks first |
| Gate sizing | Min-cut produces consistent injection sizes |
| Full loop | Violation captured, remedy stored, next run gets warned |
| No bloat | 90%+ of injections stay at 3 items or fewer |
| No superstition | Fixes that fail contract validation are rejected |

**Invariant:** Same input fixture always yields the same guardrail brief. No randomness. No LLM inference.

---

## The Learning Loop

### How Events Flow

```
Agent starts
  |
  v
Preflight: query ruvector for guardrails
  |
  v
Inject GuardrailBrief into agent context
  |
  v
Agent writes code
  |
  v
Contract tests run
  |
  +-- PASS --> emit REMEDIATION_APPLIED --> store recipe in ruvector
  |
  +-- FAIL --> emit CONTRACT_VIOLATION --> store violation in ruvector
                |
                v
              Agent fixes code --> re-test --> loop
```

### Event: CONTRACT_VIOLATION

Emitted when a contract test fails. One event per violated rule_id.

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

### Event: REMEDIATION_APPLIED

Emitted when a fix passes validation. Linked to the original violation.

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

A remedy is only stored if `validation.contracts_passed` is `true`. This prevents superstitious fixes from entering the knowledge base.

---

## Min-Cut Gating

Not every situation needs the same amount of context. The min-cut gate measures how strongly the current edit connects to prior violation/remedy patterns and adjusts injection size.

### The Edit-Intent Graph

For each agent run, a small graph is built:

- **Left side (intent):** Touched file paths, new imports, API calls detected, candidate contracts
- **Right side (memory):** Remediation recipes, prior incidents in this repo, ADR-derived constraints

The min-cut between the intent cluster and the remediation library determines injection size.

### Tension Levels

| Cut Value | Tension | Injection | Action |
|-----------|---------|-----------|--------|
| High (well-connected) | Low | 1-3 remedies | Hints only |
| Medium | Medium | 5-7 remedies | Guidance |
| Low (fragile) | High | 10+ remedies | Full guardrails |
| Very low | Critical | All + escalation | Force stop, human review |

### Edge Weights

| Connection | Higher Weight | Lower Weight |
|------------|---------------|--------------|
| Intent to Contract | Exact pattern match | Fuzzy similarity |
| Contract to Remediation | Same signature | Different signature |
| Remediation to Context | Same folder and stack | Different stack |
| Recency | Less than 7 days old | More than 30 days old |
| Success Rate | Above 0.8 | Below 0.5 |

### Default Behavior

The default injection is 3 items (WARN mode). The target is 90% of runs staying at 3 items or fewer.

```typescript
interface GateDecision {
  injection_size: 3 | 5 | 10;
  mode: 'WARN' | 'BLOCK' | 'ESCALATE';
  reason: string;
}
```

**Modes:**
- **WARN** (default): Inject the brief, let the agent proceed
- **BLOCK**: Deterministic signature match found in the diff — halt until resolved
- **ESCALATE**: Fragile area with high historical failure rate — request human review

---

## The Guardrail Brief

What gets injected into the agent's context before code generation:

### Low Tension (3 items)

```markdown
## Guardrail Brief (3 items, low tension)

You are about to edit `controllers/auth.ts`.

**Known constraints:**
- AUTH-001: No localStorage for tokens (use httpOnly cookies)

**Proven fixes for similar edits:**
1. Replace `localStorage.setItem('token', ...)` with
   `res.cookie('token', ..., { httpOnly: true })`

**If you violate AUTH-001, the build will fail.**
```

### High Tension (10 items)

```markdown
## Guardrail Brief (10 items, HIGH TENSION)

This edit touches multiple protected areas.

**Active constraints:**
- AUTH-001: No localStorage for tokens
- LAYER-001: Controllers cannot import prisma directly
- SEC-002: All inputs must be validated

**Prior incidents in this repo:**
- 3 violations of AUTH-001 in past 30 days
- 2 violations of LAYER-001 in this folder

**Proven fixes:**
1. localStorage to httpOnly cookie
2. Direct prisma to service layer
3. req.body to validateInput(req.body)
...

**Recommendation:** Review docs/contracts/ before proceeding.
```

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

### Decay Function

Recipes that have not been used recently lose confidence:

```
effective_confidence = confidence * exp(-days_since_last_seen / 30)
```

Half-life of 30 days. A fix for Next.js 14 that has not been validated in 60 days will be significantly demoted even if it had a perfect success rate.

### Retrieval Ranking

When multiple recipes match a violation:

1. Context match score (framework, language, path pattern)
2. Success rate
3. Recency
4. Confidence (with decay applied)

Only the top-ranked recipe per contract is returned to avoid conflicting suggestions.

---

## Two-Stage Gating

The simulation layer uses two enforcement points:

| Stage | When | What Happens |
|-------|------|--------------|
| **Preflight** | Before the agent writes code | WARN with guardrail brief |
| **Pre-commit** | After writing, before committing | BLOCK if violation detected in diff |

```
Agent intent --> Preflight WARN --> Agent writes --> Pre-commit check --> BLOCK if violation
```

This prevents repeat violations proactively (preflight) while still catching anything that slips through (pre-commit).

---

## Expected Results

All targets measured via ReplayLab with deterministic fixtures. No LLM inference involved in measurement.

| Metric | Target | Measurement |
|--------|--------|-------------|
| Repeat violation rate | 60% or greater reduction vs no-injection baseline | Violations per class per 50 runs |
| Time-to-green | 30% or greater reduction in median attempts | Attempts to pass contracts |
| Injection stays small | 90% or more of runs inject 3 items or fewer | Guardrail brief item count |
| No regression | Contract pass rate equal to or above baseline | Pass/fail ratio |
| Deterministic replay | Same seed produces same output 100% of the time | Unit tests |

### Baseline Definition

The baseline is measured as violations per 100 runs without injection (control group: same fixtures, no guardrail brief injected). Statistical significance threshold: p < 0.05.

---

## Test Plan

### Unit Tests

| Category | Tests | What They Prove |
|----------|-------|-----------------|
| FP-* | Fingerprint extraction | Same diff always produces same fingerprint; correct paths, imports, API calls detected |
| SIG-* | Signature generation | AST signatures stable across variable renames; import signatures catch direct client imports; regex fallback has low false positives |
| RET-* | Retrieval ranking | Highest confidence + closest context ranks first; old low-confidence recipes demoted; no recipes for mismatched contract IDs |
| GATE-* | Gate sizing | Strong match injects 3 items; fragile + high failure escalates; deterministic match in diff blocks |

### Integration Tests

| Test | What It Proves |
|------|----------------|
| LOOP-001: Learns and reuses | Violation stored, fix stored, next variant gets warned, uses fix on first attempt |
| LOOP-002: Does not bloat | Over 100 random runs, 90%+ injections are 3 items or fewer |
| LOOP-003: No superstition | Fix that fails contract validation is rejected from the store |

### E2E Test

Real Specflow runner with a seeded remedy against a known violation. Verifies the output includes "Known fix available" with the correct contract ID.

---

## Invariants

| ID | Rule |
|----|------|
| SF-EVT-001 | One violation event per rule_id |
| SF-EVT-002 | Remediation linked to violation via violation_ref |
| RV-STORE-001 | No recipe stored unless contracts_passed is true |
| RV-STORE-002 | Retrieval returns recipes ranked by confidence + context match |
| INJ-001 | Agent receives GuardrailBrief before planning or editing |
| INJ-002 | GuardrailBrief is 3 items or fewer unless gate escalates |
| GATE-001 | Default mode is WARN with injection_size of 3 |
| GATE-002 | BLOCK only on deterministic signature match in diff |
| GATE-003 | ESCALATE only when cohesion is low AND historical failure rate is high |
| TEST-REPLAY-001 | Same fixture input produces same guardrail brief output |
| ARCH-001 | Specflow remains enforcement authority; ruvector never overrides contracts |

---

## Rollout Phases

### Phase 1: Prove the Loop (Week 1-2)

Build ReplayLab with 10 contracts and 30 violation fixtures. Ship all unit tests (FP-*, SIG-*, RET-*, GATE-*) and integration tests (LOOP-001, LOOP-002, LOOP-003). Implement minimal store/retrieve and brief generation.

**Exit:** All tests green, deterministic replay proven.

### Phase 2: Wire Into Agents (Week 3-4)

Add preflight injection to Claude Code skill. Add "Known fix available" output on CI failure. Measure baseline violation rate (no injection) then measure post-injection violation rate.

**Exit:** Measurable reduction in repeat violations.

### Phase 3: Min-Cut Gating (Week 5-6)

Implement cohesion scoring. Verify against LOOP-002 (no bloat) and LOOP-001 (success rate). Tune thresholds based on data.

**Exit:** Gating improves efficiency without harming success rate.

---

## Relationship to Other Systems

| System | Role | Simulation Layer Adds |
|--------|------|----------------------|
| [Contract Tests](/core-concepts/contracts/) | Detect violations (static scan) | Structured event emission |
| [Self-Healing Fix Loops](/advanced/self-healing/) | Auto-repair after failure | Fix recipes fed back into learning store |
| [Post-Mortem Learning](/advanced/learning-system/) | Record violations and fixes | ReplayLab proves the learning actually works |
| [Journey Tests](/core-concepts/journeys/) | Verify user flows end-to-end | Unaffected — simulation operates at contract level |
| [CI Feedback Loop](/advanced/ci-feedback/) | Report CI status after push | "Known fix available" output on failures |

The simulation layer sits underneath the learning system and self-healing loops, providing the deterministic proof that they work. Without simulation, learning is a claim. With simulation, it is a measurement.

---

## Sample Apps

Three sample applications demonstrate simulation concepts:

| App | What It Demonstrates |
|-----|---------------------|
| **[AISP-Specflow](https://github.com/Hulupeep/Specflow/tree/main/sample%20apps/AISP-Specflow)** | AISP formal spec (less than 2% ambiguity) combined with Specflow contract enforcement. 11/11 contract tests passing. |
| **[MindSplit](https://github.com/Hulupeep/Specflow/tree/main/sample%20apps/mindsplit)** | Graph min-cut algorithm for separating interleaved ideas. Proves deterministic min-cut results (same input, same output). |
| **[QueueCraft](https://github.com/Hulupeep/Specflow/tree/main/sample%20apps/queuecraft)** | Real product built entirely with Specflow in approximately 4 hours. 8-part dev blog documenting the process. |

---

## Related Pages

- [Post-Mortem Learning System](/advanced/learning-system/) — The learning layer that simulation validates
- [Self-Healing Fix Loops](/advanced/self-healing/) — Autonomous violation repair that feeds into the learning store
- [What Are Contracts?](/core-concepts/contracts/) — The enforcement mechanism simulation builds on
- [CI Feedback Loop](/advanced/ci-feedback/) — Where "Known fix available" appears
- [Security & Accessibility Gates](/core-concepts/security-accessibility/) — Default gates that benefit most from learned remedies

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
