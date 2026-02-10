---
layout: default
title: Reference
nav_order: 7
has_children: true
permalink: /reference/
---

# Reference Documentation
{: .no_toc }

Contract schemas, agent templates, CLI commands, and glossary.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Feature Contract YAML Schema

Feature contracts define architectural invariants — rules that must always hold.

**File location:** `docs/contracts/feature_*.yml`

### Full Schema

```yaml
# ─── Metadata ───────────────────────────────────────────────
contract_type: feature                    # Required: "feature"
feature_name: <string>                    # Required: snake_case identifier
version: <integer>                        # Required: increment on changes
description: <string>                     # Required: what this feature does
issue: "#<number>"                        # Optional: GitHub issue reference

# ─── Non-Negotiable Rules ──────────────────────────────────
non_negotiable_rules:
  - id: <DOMAIN-NNN>                      # Required: unique rule ID (e.g., ARCH-001)
    rule: <string>                        # Required: human-readable invariant
    severity: critical | important | future  # Required: enforcement level
    enforcement: contract_test | e2e_test    # Required: how it's verified
    required_patterns:                    # Optional: regex patterns that MUST exist
      - pattern: <regex>
        in_files: <glob>                  # e.g., "src/features/auth/**/*.ts"
        description: <string>
    forbidden_patterns:                   # Optional: regex patterns that MUST NOT exist
      - pattern: <regex>
        in_files: <glob>
        description: <string>

# ─── Compliance Checklist ──────────────────────────────────
compliance_checklist:                     # Optional: human/agent reference list
  - <string>                              # e.g., "All tables have RLS enabled"

# ─── Test Hooks ────────────────────────────────────────────
test_hooks:
  tests:
    - file: <path>                        # e.g., "src/__tests__/contracts/auth_contract.test.ts"
      covers:
        - <RULE-ID>                       # Which rules this test file verifies
```

### Example: Architecture Contract

```yaml
contract_type: feature
feature_name: architecture
version: 3
description: Cross-cutting architectural rules for the entire codebase

non_negotiable_rules:
  - id: ARCH-001
    rule: "Protected routes MUST use ProtectedRoute wrapper"
    severity: critical
    enforcement: contract_test
    required_patterns:
      - pattern: "ProtectedRoute|<Navigate"
        in_files: "src/routes/**/*.tsx"
        description: "Every route file must use ProtectedRoute or Navigate"

  - id: ARCH-002
    rule: "Domain entities MUST use Zod validation schemas"
    severity: critical
    enforcement: contract_test
    required_patterns:
      - pattern: "z\\.object|z\\.string|z\\.number|z\\.enum"
        in_files: "src/domain/entities/**/*.ts"
        description: "Every entity file must contain Zod schema definitions"

  - id: ARCH-003
    rule: "Data access MUST go through repository pattern"
    severity: critical
    enforcement: contract_test
    required_patterns:
      - pattern: "supabase\\.from|supabase\\.rpc"
        in_files: "src/adapters/repositories/**/*.ts"
        description: "Supabase access only in repository files"
    forbidden_patterns:
      - pattern: "supabase\\.from|supabase\\.rpc"
        in_files: "src/features/**/components/**/*.tsx"
        description: "No direct Supabase access in UI components"

  - id: ARCH-004
    rule: "All database tables MUST have RLS enabled"
    severity: critical
    enforcement: contract_test
    required_patterns:
      - pattern: "ENABLE ROW LEVEL SECURITY"
        in_files: "supabase/migrations/**/*.sql"
        description: "Every migration creating a table must enable RLS"

compliance_checklist:
  - "No Supabase calls outside repositories"
  - "All routes wrapped in ProtectedRoute"
  - "Zod schemas for every domain entity"
  - "RLS policies on every table"

test_hooks:
  tests:
    - file: "src/__tests__/contracts/architecture_contract.test.ts"
      covers: [ARCH-001, ARCH-002, ARCH-003, ARCH-004]
```

### Rule ID Conventions

Rule IDs follow the pattern `<DOMAIN>-<NNN>`:

| Domain Prefix | Area | Examples |
|---------------|------|----------|
| `ARCH` | Cross-cutting architecture | ARCH-001 (auth), ARCH-004 (RLS) |
| `LEAVE` | Leave management | LEAVE-001 (balance debit), LEAVE-002 (no negative) |
| `ROSTER` | Scheduling/rostering | ROSTER-001 (room-based) |
| `AUTH` | Authentication | AUTH-001 (bcrypt), AUTH-002 (session) |
| `SUB` | Subscription/billing | SUB-001 (Stripe) |
| `SET` | Settings | SET-001 (unified page) |
| `COV` | Cover staff | COV-001 (pool model) |
| `CRED` | Credentials/qualifications | CRED-001 (types catalog) |
| `PAX` | Paxton integration | PAX-001 (access control) |
| `PROG` | Programs/activities | PROG-001 (CRUD) |
| `NAV` | Navigation | NAV-001 (config-driven) |

**Numbering:** Start at 001 within each domain. Increment sequentially. Never reuse numbers.

### Severity Levels

| Severity | Build Behavior | When to Use |
|----------|---------------|-------------|
| `critical` | Blocks merge if test fails | Core business rules, security, data integrity |
| `important` | Warns but allows merge | Best practices, performance, UX standards |
| `future` | Info only, no blocking | Aspirational rules, planned features |

---

## Journey Contract YAML Schema

Journey contracts define end-to-end user workflows that serve as the Definition of Done.

**File location:** `docs/contracts/journey_*.yml`

### Full Schema

```yaml
# ─── Metadata ───────────────────────────────────────────────
journey_meta:
  id: J-<UPPER-KEBAB-NAME>               # Required: unique journey ID
  from_spec: <path>                       # Optional: source spec document
  covers_reqs:                            # Required: requirement IDs this journey tests
    - <REQ-ID>
  type: e2e                               # Required: always "e2e" for journeys
  dod_criticality: critical | important | future  # Required: enforcement level
  issue: "#<number>"                      # Required: GitHub issue reference
  status: not_tested | passing | failing  # Required: current test status
  last_verified: <ISO-date> | null        # Required: when tests last ran green
  owner: <string>                         # Optional: team or person responsible (e.g., @alice)

# ─── Preconditions ─────────────────────────────────────────
preconditions:
  - description: <string>                 # Human-readable precondition
    setup_hint: <string>                  # How to set up this precondition in test

# ─── Timing ────────────────────────────────────────────────
timing:                                   # Optional: UI timing hints for test waits
  form_submit_delay: <ms>                 # Delay after form submit
  modal_animation: <ms>                   # Modal open/close animation duration
  toast_display: <ms>                     # Toast notification display duration

# ─── Steps ─────────────────────────────────────────────────
steps:
  - step: <integer>                       # Sequential step number (1-based)
    name: <string>                        # Human-readable step description
    required_elements:                    # UI elements that must exist
      - selector: <CSS-selector>          # data-testid or element selector
    action:                               # Optional: user action to perform
      - click: <selector>                 # Click an element
      - fill: <selector>                  # Type into an input
        value: <string>
      - select: <selector>               # Select from dropdown
        value: <string>
    expected:                             # What should happen after this step
      - type: navigation | element_visible | text_contains | api_call
        path_contains: <string>           # For navigation type
        selector: <selector>              # For element_visible/text_contains
        text: <string>                    # For text_contains
        method: <HTTP-method>             # For api_call
        path_contains: <string>           # For api_call

# ─── Test Hooks ────────────────────────────────────────────
test_hooks:
  e2e_test_file: <path>                   # Playwright test file path

# ─── Acceptance Criteria ───────────────────────────────────
acceptance_criteria:                      # Human-readable DoD checklist
  - <string>
```

### Example: Staff Request Leave Journey

```yaml
journey_meta:
  id: J-STAFF-REQUEST-LEAVE
  from_spec: "docs/specs/ssst_mvp.md"
  covers_reqs:
    - LEAVE-001
    - LEAVE-002
    - LEAVE-010
  type: "e2e"
  dod_criticality: critical
  issue: "#93"
  status: not_tested
  last_verified: null

preconditions:
  - description: "Staff user is logged in"
    setup_hint: "Call loginAsStaff(page) helper before journey steps"
  - description: "Staff user has leave allowance configured"
    setup_hint: "Seed employee with annual_leave_days_allowance > 0"
  - description: "At least one leave type exists (Annual Leave)"
    setup_hint: "Seed leave_types table with 'Annual Leave'"

timing:
  form_submit_delay: 500
  modal_animation: 300
  toast_display: 3000

steps:
  - step: 1
    name: "Navigate to leave requests page"
    required_elements:
      - selector: "[data-testid='nav-leave-requests']"
    expected:
      - type: "navigation"
        path_contains: "/leave-requests"

  - step: 2
    name: "Click 'Request Leave' button"
    required_elements:
      - selector: "[data-testid='add-leave-request-btn']"
    action:
      - click: "[data-testid='add-leave-request-btn']"
    expected:
      - type: "element_visible"
        selector: "[data-testid='leave-request-form']"

  - step: 3
    name: "See current balance displayed"
    required_elements:
      - selector: "[data-testid='current-balance']"
    expected:
      - type: "text_contains"
        selector: "[data-testid='current-balance']"
        text: "days remaining"

  - step: 4
    name: "Fill leave request form"
    required_elements:
      - selector: "select[name='leaveTypeId']"
      - selector: "input[name='startDate']"
      - selector: "input[name='endDate']"
      - selector: "button[type='submit']"
    action:
      - select: "select[name='leaveTypeId']"
        value: "Annual Leave"
      - fill: "input[name='startDate']"
        value: "2026-02-15"
      - fill: "input[name='endDate']"
        value: "2026-02-16"
    expected:
      - type: "element_visible"
        selector: "[data-testid='balance-after']"

  - step: 5
    name: "Submit leave request"
    action:
      - click: "button[type='submit']"
    expected:
      - type: "api_call"
        method: "POST"
        path_contains: "/leave_requests"
      - type: "element_visible"
        selector: "[data-testid='toast-success']"

  - step: 6
    name: "Verify request appears in list with pending status"
    expected:
      - type: "element_visible"
        selector: "[data-testid='leave-request-card']"
      - type: "text_contains"
        selector: "[data-testid='leave-request-status']"
        text: "pending"

test_hooks:
  e2e_test_file: "tests/e2e/journey_staff_request_leave.spec.ts"

acceptance_criteria:
  - "Staff can navigate to leave requests page"
  - "Staff can open leave request form"
  - "Form displays current leave balance"
  - "Form shows balance impact (before to after)"
  - "Staff can submit request with dates and leave type"
  - "Request appears in list with 'pending' status"
  - "Success notification is shown"
```

---

## CSV Journey Format

For teams where product designers define journeys in spreadsheets, the CSV format provides a non-technical entry point that compiles to standard journey contracts.

```bash
npm run compile:journeys -- journeys.csv
```

See the full [CSV Journey Schema Reference](/reference/csv-journey-schema/) for column definitions, validation rules, and examples.

---

## CONTRACT_INDEX.yml Schema

The contract index is a registry of all contracts, requirements, and test files.

**File location:** `docs/contracts/CONTRACT_INDEX.yml`

### Schema

```yaml
# ─── Index Metadata ────────────────────────────────────────
version: <integer>
last_updated: <ISO-date>
total_contracts: <integer>
total_journeys: <integer>

# ─── Contracts ─────────────────────────────────────────────
contracts:
  - id: <CONTRACT-ID>                     # e.g., "feature_architecture" or "J-STAFF-REQUEST-LEAVE"
    file: <path>                          # Relative to docs/contracts/
    status: active | draft | deprecated
    type: feature | journey
    dod_criticality: critical | important | future  # For journeys
    covers_reqs:                          # Requirement IDs covered
      - <REQ-ID>
    e2e_test: <path>                      # For journeys: Playwright test file
    issue: "#<number>"                    # GitHub issue reference

# ─── Requirements Coverage ─────────────────────────────────
requirements_coverage:
  - req_id: <REQ-ID>                      # e.g., LEAVE-001
    description: <string>
    covered_by:                           # Contracts that verify this requirement
      - <CONTRACT-ID>
    status: covered | partial | uncovered

# ─── Test Files ────────────────────────────────────────────
test_files:
  - file: <path>
    type: contract | e2e | unit
    covers:                               # Contract/journey IDs this test covers
      - <CONTRACT-ID>
    status: passing | failing | not_created
```

---

## Agent Prompt Template Structure

Specflow agents are markdown files that define specialized LLM behaviors. Each agent follows a standard template.

**File location:** `scripts/agents/<agent-name>.md` (project copies) or `Specflow/agents/<agent-name>.md` (canonical)

### Template Structure

```markdown
# Agent: <agent-name>

## Role
One paragraph defining the agent's identity, expertise, and purpose.

## Operating Modes
### Mode A: <Name>
Description of first operating mode.

### Mode B: <Name>
Description of second operating mode.

## Trigger Conditions
- When the user says "..."
- When a specific condition is met
- When another agent completes a prerequisite task

## Inputs
- What the agent needs to start working
- References to files, issues, or other artifacts

## Process
### Step 1: <Name>
Detailed instructions for the first step.

### Step 2: <Name>
Detailed instructions for the second step.

## Output Templates
### Template A
```
Expected output format with placeholders.
```

## Quality Gates
- [ ] Checklist items the agent must verify before finishing

## Anti-Patterns to Avoid
1. **Pattern name:** Description of what NOT to do
```

### Key Agents

| Agent | Purpose | Modes |
|-------|---------|-------|
| `specflow-writer` | Generate GitHub issues with Gherkin, SQL contracts, journeys | Epic, Subtask Slices, Single Ticket |
| `migration-builder` | Create Supabase migration SQL from data contracts | Single migration |
| `frontend-builder` | Build React components, hooks, routes from specs | Component, Hook, Page |
| `edge-function-builder` | Create Supabase Edge Functions | Single function |
| `contract-generator` | Generate YAML contracts from issue specs | Feature, Journey |
| `contract-test-generator` | Generate test files from contract YAML | Contract test, E2E test |
| `playwright-from-specflow` | Generate Playwright tests from Gherkin scenarios | Single journey, Batch |
| `journey-tester` | Create cross-feature E2E journey tests | Single journey |
| `journey-gate` | Enforce journey tests at three tiers | Tier 1 (issue), Tier 2 (wave), Tier 3 (regression) |
| `waves-controller` | Orchestrate full pipeline execution | Autonomous, Guided |
| `board-auditor` | Audit GitHub issues for specflow compliance | Full audit, Wave audit |
| `ticket-closer` | Close issues with commit summaries | Single, Batch |

### Agent Invocation Pattern

To invoke an agent using Claude Code's Task tool:

```
1. Read scripts/agents/{agent-name}.md
2. Task(
     description: "Short description",
     prompt: "{full agent prompt}\n\n---\n\nSPECIFIC TASK: {what to do}",
     subagent_type: "general-purpose"
   )
```

### Agent Orchestration Pipeline

```
specflow-writer → dependency-mapper → sprint-executor
    │                                       │
    ▼                                       ▼
migration-builder + frontend-builder + edge-function-builder
    │
    ▼
contract-validator → playwright-from-specflow → journey-tester
    │
    ▼
test-runner → e2e-test-auditor → journey-enforcer → ticket-closer
```

---

## Wave Execution System

Waves group related GitHub issues for parallel execution.

### Wave Structure

```
Wave N:
  Slot a: Issue #X (Size: S/M/L)
  Slot b: Issue #Y (Size: S/M/L)
  Slot c: Issue #Z (Size: S/M/L)
```

Each slot runs as a parallel agent. Agents within a wave must not have dependencies on each other. Cross-wave dependencies are tracked in the plan.

### Wave Lifecycle

1. **Plan** — Group issues into waves, check dependencies
2. **Execute** — Launch parallel agents (1 per slot)
3. **Verify** — `lint + build + contract tests` after all slots complete
4. **Commit** — Single commit per wave with all issue references
5. **Push** — Deploy to production via Vercel auto-deploy
6. **Close** — Close GitHub issues with commit hash references

### Journey Gates Between Waves

| Gate | Scope | Blocks |
|------|-------|--------|
| **Tier 1: Issue** | Journey tests from one issue | Issue closure |
| **Tier 2: Wave** | All journey tests from wave issues | Next wave start |
| **Tier 3: Regression** | Full E2E suite vs baseline | Merge to main |

### Baseline Management

The file `.specflow/baseline.json` tracks the last known-good test state. It's updated only after a clean Tier 3 pass. Regression detection compares current results against the baseline.

---

## CLI Commands

### Verification Commands

```bash
# Run all contract tests
pnpm test -- contracts

# Run specific contract test
pnpm test -- src/__tests__/contracts/architecture_contract.test.ts

# Run all E2E tests
pnpm test:e2e

# Run specific journey test
pnpm test:e2e -- tests/e2e/journey_staff_request_leave.spec.ts

# Run E2E tests by criticality tag
pnpm test:e2e -- --grep @critical

# Check contract completeness
node scripts/check-contract-completeness.mjs

# Full verification gate
pnpm lint && pnpm build && pnpm test && pnpm test:e2e
```

### GitHub Integration

```bash
# List open issues
gh issue list -R Hulupeep/timebreez

# View issue details
gh issue view 123 -R Hulupeep/timebreez

# Create issue with specflow template
gh issue create -R Hulupeep/timebreez \
  --title "feat: Description" \
  --body "$(cat <<'EOF'
## Scope
...
## Data Contract
...
## Gherkin Scenarios
...
## Acceptance Criteria
...
EOF
)"

# Close issue with commit reference
gh issue close 123 -R Hulupeep/timebreez \
  --comment "Implemented in commit abc1234"
```

### Specflow Hook Commands

```bash
# Install git hooks
bash Specflow/install-hooks.sh .

# Defer tests temporarily
touch .claude/.defer-tests

# Re-enable tests
rm .claude/.defer-tests

# Defer specific journey
echo "J-STAFF-REQUEST-LEAVE: tracking #456" >> .claude/.defer-journal
```

### Database Commands

```bash
# Push migrations to Supabase
supabase db push --linked

# Push out-of-order migrations
supabase db push --linked --include-all

# List migration status
supabase migration list --linked

# Reset local database
supabase db reset
```

---

## Glossary

| Term | Definition |
|------|-----------|
| **Contract** | A YAML file defining rules that must hold. Either a *feature contract* (architectural invariants) or a *journey contract* (end-to-end workflow). |
| **Contract test** | A test that scans code for pattern compliance. Runs fast (milliseconds). Verifies file structure, required patterns, forbidden patterns. |
| **DoD criticality** | How important a journey is for release. `critical` blocks merge, `important` warns, `future` is aspirational. |
| **E2E test** | End-to-end test using Playwright. Runs the application in a browser and verifies user workflows. |
| **Feature contract** | YAML file defining architectural invariants (e.g., "all tables must have RLS"). File pattern: `feature_*.yml`. |
| **Forbidden pattern** | A regex that must NOT appear in specified files. Used in feature contracts to prevent anti-patterns. |
| **Invariant** | A rule that must always be true. Numbered as `<DOMAIN>-<NNN>` (e.g., ARCH-001, LEAVE-002). |
| **Journey** | An end-to-end user workflow that defines "done" for a feature. Maps to a Playwright E2E test. |
| **Journey contract** | YAML file defining a user journey with preconditions, steps, expected outcomes, and acceptance criteria. File pattern: `journey_*.yml`. |
| **Journey gate** | A verification checkpoint. Tier 1 gates an issue, Tier 2 gates a wave, Tier 3 gates a release. |
| **Non-negotiable rule** | A contract rule that cannot be overridden by agents or prompts. Only a human can explicitly override it. |
| **Precondition** | State that must exist before a journey can run (e.g., "user is logged in"). Set up in `beforeEach`. |
| **Required pattern** | A regex that MUST appear in specified files. Used in feature contracts to enforce architectural patterns. |
| **Severity** | How strictly a rule is enforced. `critical` = blocks build. `important` = warns. `future` = info only. |
| **Specflow** | The contract-driven development system. Defines architectural rules as testable YAML, generates enforcement tests, and gates releases. |
| **Wave** | A group of 2-3 GitHub issues executed in parallel by separate agents. Waves are executed sequentially; slots within a wave run concurrently. |
| **data-testid** | HTML attribute used as stable selectors for E2E tests. Immune to CSS class or text content changes. |
| **Baseline** | `.specflow/baseline.json` — the last known-good test results. Used for regression detection at Tier 3. |
| **Defer journal** | `.claude/.defer-journal` — file that tracks temporarily deferred journey tests with tracking issue references. |

---

## File Conventions

### Contract Files

| Pattern | Purpose | Example |
|---------|---------|---------|
| `docs/contracts/feature_*.yml` | Architectural invariants | `feature_architecture.yml` |
| `docs/contracts/journey_*.yml` | End-to-end workflows | `journey_staff_request_leave.yml` |
| `docs/contracts/CONTRACT_INDEX.yml` | Registry of all contracts | — |

### Test Files

| Pattern | Purpose | Example |
|---------|---------|---------|
| `src/__tests__/contracts/*_contract.test.ts` | Contract verification tests | `architecture_contract.test.ts` |
| `tests/e2e/journey_*.spec.ts` | Playwright journey tests | `journey_staff_request_leave.spec.ts` |

### Agent Files

| Pattern | Purpose | Example |
|---------|---------|---------|
| `scripts/agents/*.md` | Project agent copies | `specflow-writer.md` |
| `Specflow/agents/*.md` | Canonical agent definitions | `specflow-writer.md` |

### Migration Files

| Pattern | Purpose | Example |
|---------|---------|---------|
| `supabase/migrations/<NNN>_<name>.sql` | Database schema changes | `228_explanation_templates.sql` |

**Migration numbering:** Three-digit sequential. Check for collisions when running parallel agents.

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
