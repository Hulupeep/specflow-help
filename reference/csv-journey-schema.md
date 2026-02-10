---
layout: default
title: CSV Journey Schema
parent: Reference
nav_order: 3
permalink: /reference/csv-journey-schema/
---

# CSV Journey Schema
{: .no_toc }

Column reference and validation rules for the CSV journey format.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

The CSV journey format lets product designers define user journeys in a spreadsheet-friendly format. The `specflow-compile` script converts CSV rows into YAML journey contracts and Playwright test stubs.

**Template:** `templates/journeys-template.csv`

---

## Column Schema

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `journey_id` | string | Yes | Unique journey identifier. Must match `/^J-[A-Z][A-Z0-9-]+$/` |
| `journey_name` | string | Yes | Human-readable journey name (e.g., "User Signup") |
| `step` | integer | Yes | Step number within the journey. Must be sequential starting at 1 |
| `user_does` | string | Yes | What the user does in this step |
| `system_shows` | string | Yes | What the system shows after the user action |
| `critical` | string | Yes | `yes` or `no`. Maps to `dod_criticality`: yes=critical, no=important |
| `owner` | string | Yes | Team or person responsible (e.g., `@alice`, `design-team`) |
| `notes` | string | No | Extra context. Non-empty notes become `acceptance_criteria[]` entries |

---

## Column-to-YAML Mapping

| CSV Column | YAML Field |
|------------|------------|
| `journey_id` | `journey_meta.id` |
| `journey_name` | Used in Playwright `test.describe()` label |
| `step` | `steps[].step` |
| `user_does` | `steps[].name` |
| `system_shows` | `steps[].expected[].description` |
| `critical` | `journey_meta.dod_criticality` (yes=critical, no=important) |
| `owner` | `journey_meta.owner` |
| `notes` | `acceptance_criteria[]` (only when non-empty) |

---

## Validation Rules

The compiler enforces these rules and exits with an error on violation:

### Journey ID Format

Must match `/^J-[A-Z][A-Z0-9-]+$/`:
- Starts with `J-`
- Followed by an uppercase letter
- Then uppercase letters, digits, or hyphens

```
J-SIGNUP-FLOW     # valid
J-LOGIN-FLOW      # valid
J-A               # valid (minimum)
signup-flow       # INVALID (no J- prefix)
J-signup-flow     # INVALID (lowercase)
J-1FLOW           # INVALID (digit after J-)
```

### Step Sequencing

Steps within each journey must be sequential integers starting at 1:

```csv
J-FLOW,Flow,1,...    # OK: starts at 1
J-FLOW,Flow,2,...    # OK: sequential
J-FLOW,Flow,4,...    # ERROR: expected step 3, got 4
```

### No Duplicate Steps

Each `(journey_id, step)` pair must be unique:

```csv
J-FLOW,Flow,1,...    # OK
J-FLOW,Flow,1,...    # ERROR: duplicate (J-FLOW, 1)
```

### Required Fields

- `owner` must be non-empty for every row
- `critical` must be exactly `yes` or `no` (case-insensitive)
- `step` must be a positive integer

### Criticality Consistency

If rows within the same journey have different `critical` values, the compiler uses the value from the first row and emits a warning to stderr:

```
Warning: journey "J-LOGIN-FLOW" has inconsistent critical values. Using "yes" from first row.
```

---

## Example CSV

```csv
journey_id,journey_name,step,user_does,system_shows,critical,owner,notes
J-SIGNUP-FLOW,User Signup,1,Clicks "Sign Up",Shows registration form,yes,@alice,
J-SIGNUP-FLOW,User Signup,2,Fills email + password,Validates in real-time,yes,@alice,
J-SIGNUP-FLOW,User Signup,3,Clicks submit,Shows success + redirect to dashboard,yes,@alice,Must receive welcome email
J-LOGIN-FLOW,User Login,1,Clicks "Log In",Shows login form,yes,@bob,
J-LOGIN-FLOW,User Login,2,Enters credentials,Authenticates,yes,@bob,
J-LOGIN-FLOW,User Login,3,Clicks submit,Redirects to dashboard,yes,@bob,Session cookie set
J-CHECKOUT-FLOW,Checkout,1,Adds item to cart,Shows cart with item,yes,@carol,
J-CHECKOUT-FLOW,Checkout,2,Clicks checkout,Shows payment form,yes,@carol,
J-CHECKOUT-FLOW,Checkout,3,Enters payment details,Validates card,yes,@carol,
J-CHECKOUT-FLOW,Checkout,4,Clicks pay,Processes payment + shows confirmation,yes,@carol,Must send receipt email
```

---

## CLI Usage

```bash
# Compile CSV to YAML contracts + Playwright test stubs
npm run compile:journeys -- <csv-file>

# Example
npm run compile:journeys -- templates/journeys-template.csv
```

### Output

For each unique `journey_id` in the CSV, the compiler generates:

| Output | Path | Purpose |
|--------|------|---------|
| Journey contract | `docs/contracts/journey_<slug>.yml` | YAML contract matching CONTRACT-SCHEMA.md |
| Playwright stub | `tests/e2e/journey_<slug>.spec.ts` | Test stub with TODO body per step |

The slug is derived from the journey ID: `J-SIGNUP-FLOW` becomes `signup_flow`.

### Error Output

On validation failure, the compiler prints an error message and exits with code 1:

```
Line 3: journey_id "signup" must match /^J-[A-Z][A-Z0-9-]+$/
```

---

## Quoted Fields

The CSV parser handles quoted fields correctly:

```csv
J-FLOW,Flow,1,"Clicks ""Sign Up""",Shows form,yes,@alice,
```

- Commas inside quotes are preserved
- Escaped quotes (`""`) are handled
- Leading/trailing whitespace in fields is trimmed

---

## Next Steps

- **[Team Workflows](/core-concepts/team-workflows/)** — Full team workflow guide
- **[Journey Contract Schema](/reference/)** — Full YAML schema reference
- **[What Are Journeys?](/core-concepts/journeys/)** — Journey concepts and patterns
