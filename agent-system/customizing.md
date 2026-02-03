---
layout: default
title: Customizing Agents
parent: Agent System
nav_order: 4
permalink: /agent-system/customizing/
---

# Customizing Agents
{: .no_toc }

Adapt agent behavior to your project's needs.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

Specflow agents are **prompt-based**: each agent is defined by a markdown file in `scripts/agents/`. You can customize their behavior by editing these prompts.

**No code changes required.** Just edit the markdown.

---

## Agent Anatomy

Every agent has this structure:

```markdown
# Agent: agent-name

## Role
Brief description of what this agent does

## Trigger Conditions
- When to invoke this agent
- User phrases that indicate need

## Inputs
What this agent needs to function

## Process
Step-by-step workflow the agent follows

## Outputs
What this agent produces

## Quality Gates
Success criteria

## Rules
Non-negotiable constraints
```

---

## Common Customizations

### 1. Add Project-Specific Patterns

**Example:** Your project uses a custom repository pattern

**File:** `scripts/agents/migration-builder.md`

**Add to Rules section:**
```markdown
## Rules
- All migrations MUST use snake_case table names
- All timestamps MUST include timezone: `timestamptz`
- All foreign keys MUST have ON DELETE CASCADE (project requirement)
- All tables MUST have created_at, updated_at, deleted_at columns
```

**Result:** Agent now follows your conventions automatically.

---

### 2. Change Technology Stack

**Example:** You use MySQL instead of PostgreSQL

**File:** `scripts/agents/migration-builder.md`

**Update Process section:**
```markdown
## Process
### Step 1: Analyze Requirements
...

### Step 2: Generate MySQL Migration
```sql
-- MySQL syntax (not PostgreSQL)
CREATE TABLE employees (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  INDEX idx_name (name)
);
```
```

**Result:** Agent generates MySQL-compatible migrations.

---

### 3. Add Quality Checks

**Example:** Enforce TypeScript strict mode

**File:** `scripts/agents/frontend-builder.md`

**Add to Quality Gates:**
```markdown
## Quality Gates
- [ ] All files pass `tsc --noEmit --strict`
- [ ] No `any` types except in type definitions
- [ ] All props interfaces exported
- [ ] All components have displayName
```

**Result:** Agent verifies strict TypeScript compliance.

---

### 4. Customize Test Framework

**Example:** You use Vitest instead of Jest

**File:** `scripts/agents/test-runner.md`

**Update Commands Used section:**
```markdown
## Commands Used
```bash
# Run all tests
pnpm vitest run

# Run specific test
pnpm vitest run src/__tests__/contracts/

# Watch mode
pnpm vitest watch
```
```

**Result:** Agent uses Vitest commands.

---

## Creating Custom Agents

### When to Create a Custom Agent

Create a new agent when you have a **repeated workflow** that:
- Requires multiple steps
- Has clear inputs/outputs
- Has quality gates
- Is triggered by specific user phrases

**Examples:**
- Custom deployment agent
- Custom notification agent
- Custom data migration agent

### Template for New Agents

Create `scripts/agents/your-agent-name.md`:

```markdown
# Agent: your-agent-name

## Role
[One sentence: what this agent does]

## Trigger Conditions
- User says: "[trigger phrase 1]", "[trigger phrase 2]"
- User provides: [required inputs]

## Inputs
- **Input 1:** Description
- **Input 2:** Description

## Process

### Step 1: [First Step]
[What happens]

```bash
# Commands
command here
```

### Step 2: [Second Step]
[What happens]

## Outputs
- **Output 1:** Description
- **Output 2:** Description

## Quality Gates
- [ ] Gate 1: Description
- [ ] Gate 2: Description

## Rules
1. RULE 1 (why it matters)
2. RULE 2 (why it matters)
```

---

## Agent Composition Patterns

### Pattern 1: Sequential Agents

One agent spawns another:

```markdown
# Agent: orchestrator

## Process
### Step 3: Spawn Database Agent
1. Read `scripts/agents/migration-builder.md`
2. Task("Build migration", "{prompt + task}", "general-purpose")
3. Wait for completion

### Step 4: Spawn Test Agent
1. Read `scripts/agents/test-runner.md`
2. Task("Run tests", "{prompt + task}", "general-purpose")
```

### Pattern 2: Parallel Agents

One agent spawns multiple agents at once:

```markdown
# Agent: parallel-orchestrator

## Process
### Step 2: Spawn All Agents (Parallel)
[Single Message]:
  Task("Database", "{migration-builder prompt + task}", "general-purpose")
  Task("Frontend", "{frontend-builder prompt + task}", "general-purpose")
  Task("Backend", "{edge-function-builder prompt + task}", "general-purpose")
```

### Pattern 3: Conditional Agents

Agent decides which other agent to invoke:

```markdown
# Agent: smart-router

## Process
### Step 2: Route to Specialist
If database changes needed:
  - Invoke migration-builder
Else if Edge Function needed:
  - Invoke edge-function-builder
Else:
  - Invoke frontend-builder
```

---

## Configuration Files

Some agents read configuration from your project:

### 1. Contract Files (`docs/contracts/*.yml`)
Agents like `specflow-writer` generate these based on issues.

### 2. CLAUDE.md (Project Instructions)
Global project context all agents inherit.

**Add project-specific rules:**
```markdown
## Project-Specific Rules
- We use Tailwind, never inline styles
- We use React Query, never SWR
- We use Supabase, never Firebase
```

### 3. Agent Metadata (Optional)
Create `scripts/agents/.metadata.json`:

```json
{
  "project": "your-project",
  "stack": "React + Vite + Supabase",
  "conventions": {
    "tableNames": "snake_case",
    "componentNames": "PascalCase",
    "testFiles": "*.spec.ts"
  }
}
```

Agents can read this for context.

---

## Testing Agent Customizations

### 1. Dry Run
Invoke agent with `--dry-run` flag (if supported):

```bash
# Conceptual - adjust for your setup
claude-code task run migration-builder --dry-run
```

### 2. Small Test Case
Run on a single, simple issue first:

```markdown
# Issue #999: Test Agent Customization
Simple test to verify agent follows new rules.

## Acceptance Criteria
- [ ] Agent creates file following new naming convention
```

### 3. Verify Output
Check agent output matches expectations:
- File names follow conventions
- Code patterns match rules
- Quality gates enforced

---

## Common Pitfalls

### ❌ Overly Specific Rules
**Bad:**
```markdown
- Table `employees` MUST have column `full_name_with_title`
```

**Good:**
```markdown
- All tables MUST follow snake_case naming
```

**Why:** Specific rules don't generalize to other tables.

---

### ❌ Contradictory Rules
**Bad:**
```markdown
- All files MUST be under 200 lines
- All logic MUST be in single file (no splitting)
```

**Why:** These rules conflict.

---

### ❌ Vague Triggers
**Bad:**
```markdown
## Trigger Conditions
- When needed
```

**Good:**
```markdown
## Trigger Conditions
- User says: "create migration", "add table", "update schema"
- Files changed: `supabase/migrations/*.sql`
```

**Why:** Clear triggers = correct agent invocation.

---

## Example: Custom Deployment Agent

**File:** `scripts/agents/deploy-to-vercel.md`

```markdown
# Agent: deploy-to-vercel

## Role
Deploy application to Vercel with environment variables and preview URLs

## Trigger Conditions
- User says: "deploy to vercel", "push to production", "create preview"
- After: All tests pass, contracts verified

## Inputs
- **Environment:** `production` or `preview`
- **Branch:** Git branch to deploy
- **Env Vars:** List of environment variables to set

## Process
### Step 1: Verify Prerequisites
```bash
# Check Vercel CLI installed
vercel --version

# Check logged in
vercel whoami
```

### Step 2: Deploy
```bash
# Production
vercel --prod

# Preview
vercel
```

### Step 3: Set Environment Variables
```bash
vercel env add DATABASE_URL production
vercel env add API_KEY production
```

### Step 4: Verify Deployment
- Check deployment URL returns 200
- Run smoke tests against deployed URL

## Outputs
- **Deployment URL:** `https://your-app.vercel.app`
- **Preview URL:** `https://your-app-git-branch.vercel.app`
- **Deployment logs:** Link to Vercel dashboard

## Quality Gates
- [ ] Deployment succeeds (exit code 0)
- [ ] Health check returns 200
- [ ] Environment variables set
- [ ] SSL certificate valid

## Rules
1. NEVER deploy if tests failing
2. ALWAYS create preview for non-main branches
3. ALWAYS tag production deployments in Git
```

**Usage:**
```
Task("Deploy to Vercel production", "{deploy-to-vercel.md prompt}\n\n---\n\nBranch: main, Environment: production", "general-purpose")
```

---

## Next Steps

- **[Agent Reference](/agent-system/agent-reference/)** — See all 18 default agents
- **[DPAO Methodology](/agent-system/dpao/)** — Understand parallel orchestration
- **[Manual Contracts](/advanced/manual-contracts/)** — Write contracts without agents

---

## Best Practices

1. **Start small:** Customize one agent at a time
2. **Test in isolation:** Run agent on simple issues first
3. **Document changes:** Add comments explaining custom rules
4. **Version control:** Commit agent prompts to Git
5. **Share patterns:** If a customization works well, contribute back to Specflow repo
