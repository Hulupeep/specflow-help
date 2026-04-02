---
layout: default
title: Contract Graph
parent: Core Concepts
nav_order: 3
permalink: /core-concepts/contract-graph/
---

# Contract Graph
{: .no_toc }

How contracts, specs, journeys, tests, and ADRs reference each other — and why validating those references matters.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why "Graph"?

Contracts aren't a flat list. They form a directed graph of cross-references:

```
Spec (docs/specs/feature.md)
  → references invariant I-SEC-001
    → defined in contract (docs/contracts/security_defaults.yml)
      → scopes files (src/**/*.ts)
      → points to test (tests/contracts/security.test.js)

Issue (#500)
  → references journey J-SIGNUP-FLOW
    → defined in contract (docs/contracts/journey_signup.yml)
      → points to test (tests/e2e/journey_signup.spec.ts)
      → covers reqs [AUTH-001, AUTH-002]
        → defined in contract (docs/contracts/feature_auth.yml)

ADR (docs/adr/use-chrome-storage.md)
  → declares invariant I-SEC-001
    → enforced by contract (docs/contracts/security_defaults.yml)
      → verified by journey (docs/contracts/journey_login.yml)

CONTRACT_INDEX.yml
  → lists all contracts
  → maps requirements to contracts
  → maps journeys to test files
```

**Nodes** are: specs, contracts, journeys, tests, ADRs, invariant IDs, requirement IDs.

**Edges** are: "references", "defined in", "points to", "covers".

A broken edge means a contract claims a test file exists but it doesn't, or a spec references an invariant ID nobody defined. That silently disconnects enforcement downstream — the rule exists on paper but nothing checks it.

---

## What the Graph Validator Checks

Run it with:

```bash
npx @colmbyrne/specflow graph
```

It performs 7 checks:

| Check | What it catches |
|-------|----------------|
| **Test file references** | `test_hooks.e2e_test_file` in a contract YAML points to a file that doesn't exist on disk |
| **Journey ID uniqueness** | Same journey ID (e.g. `J-SIGNUP-FLOW`) defined in two different YAML files |
| **CONTRACT_INDEX coverage** | A contract file exists in `docs/contracts/` but isn't listed in `CONTRACT_INDEX.yml` |
| **Invariant ID references** | A spec references `SEC-010` but no contract defines that ID |
| **ADR frontmatter** | An ADR claims a journey contract exists but the file doesn't |
| **Waiver expiry** | A waiver has an `expires` date that's already past |
| **Contract directory** | `docs/contracts/` exists and has YAML files |

---

## Graph vs Verify

`verify` and `graph` check different things:

| | `npx specflow verify` | `npx specflow graph` |
|---|---|---|
| **What** | Is Specflow installed correctly? | Do contracts reference each other correctly? |
| **Checks** | Hooks, CLAUDE.md, directories, settings.json | Cross-references between contracts, tests, specs, ADRs |
| **Scope** | Infrastructure | Data integrity |
| **When** | After install or update | After adding/editing contracts |
| **Analogy** | "Is the plumbing connected?" | "Does the water flow to the right taps?" |

Both can pass independently. You can have a perfect installation (`verify` passes) with broken contract references (`graph` fails), or vice versa.

---

## Common Graph Errors

### Duplicate journey IDs

```
Error: Duplicate journey ID J-SIGNUP-FLOW in: journey_signup.yml, CONTRACT_INDEX.yml
```

**Cause:** `CONTRACT_INDEX.yml` has full journey definitions that duplicate the individual `journey_*.yml` files. The index should reference journeys, not redefine them.

**Fix:** Remove the full journey entries from `CONTRACT_INDEX.yml`, keep only the reference list at the top.

### Missing test file

```
Error: journey_signup.yml → tests/e2e/journey_signup.spec.ts NOT FOUND
```

**Cause:** A contract's `test_hooks.e2e_test_file` points to a file that doesn't exist on disk.

**Fix:** Either create the test file, or remove the `test_hooks` entry from the contract until the test is written.

### Orphan contract files

```
Warning: feature_auth.yml not listed in CONTRACT_INDEX.yml
```

**Cause:** A contract exists in `docs/contracts/` but isn't registered in the index.

**Fix:** Add the contract to `CONTRACT_INDEX.yml`.

### Undefined invariant reference

```
Warning: feature-spec.md references SEC-010 — not found in any contract
```

**Cause:** A spec references an invariant ID that no contract defines.

**Fix:** Either add the invariant to the relevant contract, or remove the reference from the spec.

### Expired waiver

```
Error: security_defaults.yml: waiver for SEC-003 expired on 2026-01-15
```

**Cause:** A waiver's `expires` date has passed. The exemption is no longer valid.

**Fix:** Either fix the underlying issue and remove the waiver, or extend the expiry with a new tracking issue.

---

## When to Run

- After adding or editing any contract YAML
- After running `specflow-writer` on tickets
- Before wave execution (waves-controller runs it automatically)
- As part of CI (catches broken references before merge)

---

## Related

- [Contracts](/core-concepts/contracts/) — How contracts work
- [Verify Setup](/getting-started/verify-setup/) — Installation verification
- [Contract Schema](/reference/contract-schema/) — YAML format specification

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
