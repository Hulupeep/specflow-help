---
layout: default
title: Your First Wave
parent: Getting Started
nav_order: 5
permalink: /getting-started/first-wave/
---

# Your First Wave
{: .no_toc }

Execute waves-controller and watch agents work together.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What is a Wave?

A **wave** is a group of GitHub issues that can be executed in parallel. Issues are grouped into waves based on:

- **Dependencies**: What blocks what
- **Priority**: Critical > Important > Future
- **Complexity**: Simple issues in early waves

**waves-controller** orchestrates the entire process:
1. Analyzes all open issues
2. Calculates dependency waves
3. Spawns specialized agents to handle each issue
4. Verifies all contracts pass
5. Closes issues when complete

**3-4x faster than manual workflows.** Proven on production projects.

---

## Prerequisites

Before running your first wave:

- ✅ [Specflow is installed](/getting-started/install/)
- ✅ Claude Code CLI is available (`claude --version` works)
- ✅ GitHub CLI is authenticated (`gh auth status`)
- ✅ API key is set (`ANTHROPIC_API_KEY` or `OPENAI_API_KEY`)
- ✅ You have at least one open GitHub issue

---

## Step 1: Create a GitHub Issue (If Needed)

Specflow works from **GitHub issues**. If you don't have one, create a simple example:

```bash
gh issue create --title "FEAT-001: Add user authentication" --body "$(cat <<'EOF'
## Description
Add basic email/password authentication to the app.

## Acceptance Criteria
- [ ] User can sign up with email and password
- [ ] User can log in with credentials
- [ ] User can log out
- [ ] Invalid credentials show error message

## Technical Notes
- Use bcrypt for password hashing
- Store sessions in database
- Return JWT tokens on successful login
EOF
)"
```

**Output:**
```
https://github.com/your-org/your-repo/issues/42
```

Take note of the issue number (e.g., `#42`).

---

## Step 2: Execute Your First Wave

Open Claude Code and run:

```
Execute waves
```

That's it. **One command.**

### What Happens Next

**Phase 1: Discovery (30s - 1m)**

```
🌊 WAVE EXECUTION STARTED
📊 Phase 1: Discovery

→ Fetching open issues from GitHub...
  Found 8 open issues

→ Analyzing dependencies...
  Issue #42: No blockers
  Issue #43: Blocked by #42
  Issue #44: No blockers
  Issue #45: Blocked by #42, #44

→ Calculating waves...
  Wave 1: Issues #42, #44 (2 issues, can run in parallel)
  Wave 2: Issue #43 (1 issue, blocked by wave 1)
  Wave 3: Issue #45 (1 issue, blocked by waves 1 & 2)

→ Priority scoring...
  #42: Critical (has "auth" tag, blocks 2 issues)
  #44: Important
  #43: Important
  #45: Future

✓ Discovery complete (42s)
```

**Phase 2: Agent Spawning (Parallel)**

```
🤖 Phase 2: Spawning Agents

Wave 1 (Parallel Execution):

  Issue #42: "Add user authentication"
  → specflow-writer: Creating contracts from issue
  → migration-builder: Analyzing database needs
  → edge-function-builder: Checking if RPC needed

  Issue #44: "Add user profile page"
  → specflow-writer: Creating contracts from issue
  → frontend-builder: Generating React component

✓ 6 agents spawned (Wave 1)
```

**Phase 3: Parallel Work (2-4m per wave)**

```
⚡ Phase 3: Agents Working

[specflow-writer → Issue #42]
  ✓ Created docs/contracts/feature_authentication.yml
  ✓ Defined 4 invariants (AUTH-001 to AUTH-004)
  ✓ Created journey contract: J-USER-SIGNUP

[migration-builder → Issue #42]
  ✓ Generated migration: 20260203_create_users_table.sql
  ✓ Added RLS policies for users table
  ✓ Created indexes on email column

[playwright-from-specflow → Issue #42]
  ✓ Created tests/e2e/journey_user_signup.spec.ts
  ✓ Mapped all acceptance criteria to test steps
  ✓ Added data-testid selectors to spec

[frontend-builder → Issue #44]
  ✓ Created src/features/profile/ProfilePage.tsx
  ✓ Added TypeScript types
  ✓ Integrated with repository pattern

✓ Wave 1 complete (3m 42s)
```

**Phase 4: Verification**

```
🧪 Phase 4: Running Tests

→ Contract tests...
  ✓ AUTH-001: Password must be hashed (bcrypt)
  ✓ AUTH-002: JWT tokens expire after 24h
  ✓ AUTH-003: Failed login attempts rate limited
  ✓ AUTH-004: Sessions stored in database
  All contract tests passed (12/12)

→ E2E tests...
  ✓ J-USER-SIGNUP: User can sign up with valid email
  ✓ J-USER-LOGIN: User can log in with credentials
  ✓ J-USER-LOGOUT: User can log out
  All E2E tests passed (3/3)

✓ Verification complete
```

**Phase 5: Close Issues**

```
📝 Phase 5: Updating GitHub

Issue #42: "Add user authentication"
  → All acceptance criteria met
  → Contract tests passing
  → E2E journeys passing
  → Closing issue with summary...
  ✓ Issue #42 closed

Issue #44: "Add user profile page"
  → All acceptance criteria met
  → Tests passing
  → Closing issue...
  ✓ Issue #44 closed

✓ Wave 1 issues closed (2/2)
```

**Final Summary**

```
🎉 WAVE EXECUTION COMPLETE

Summary:
  Duration: 4m 23s
  Issues processed: 2
  Issues closed: 2
  Contracts created: 2
  Migrations generated: 1
  E2E tests created: 3
  All tests passing: ✓

Next Wave:
  Issue #43 is now unblocked
  Ready to execute when you are

Run "Execute waves" again to continue.
```

---

## Step 3: Review What Was Generated

After the wave completes, check what was created:

### 1. Contracts (docs/contracts/)

```bash
ls docs/contracts/
# feature_authentication.yml
# journey_user_signup.yml
```

**Example contract:**
```yaml
contract_type: feature
feature_name: authentication
description: User authentication with email/password

invariants:
  - id: AUTH-001
    rule: "Passwords MUST be hashed using bcrypt before storage"
    severity: critical
    enforcement: contract_test

  - id: AUTH-002
    rule: "JWT tokens MUST expire after 24 hours"
    severity: critical
    enforcement: e2e_test
```

### 2. Migrations (supabase/migrations/)

```bash
ls supabase/migrations/
# 20260203_create_users_table.sql
```

**Example migration:**
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- RLS policies
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
-- ... policies ...
```

### 3. E2E Tests (tests/e2e/)

```bash
ls tests/e2e/
# journey_user_signup.spec.ts
# journey_user_login.spec.ts
```

**Example test:**
```typescript
test.describe('J-USER-SIGNUP', () => {
  test('User can sign up with valid email', async ({ page }) => {
    // Given: User is on signup page
    await page.goto('/signup')

    // When: User submits form with valid data
    await page.fill('[data-testid="email-input"]', 'test@example.com')
    await page.fill('[data-testid="password-input"]', 'SecurePass123!')
    await page.click('[data-testid="signup-btn"]')

    // Then: User is redirected to dashboard
    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('[data-testid="welcome-msg"]')).toBeVisible()
  })
})
```

### 4. Implementation Code (src/)

```bash
ls src/features/auth/
# SignupPage.tsx
# LoginPage.tsx
# useAuth.ts (React Query hook)
```

**Example component:**
```tsx
export function SignupPage() {
  const { mutate: signup } = useSignup()

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        data-testid="email-input"
        {...register('email')}
      />
      <input
        type="password"
        data-testid="password-input"
        {...register('password')}
      />
      <button type="submit" data-testid="signup-btn">
        Sign Up
      </button>
    </form>
  )
}
```

---

## Time Comparison

Let's be honest about what just happened:

| Task | Manual (Human) | Specflow (Agents) | Time Saved |
|------|----------------|-------------------|------------|
| Write acceptance criteria | 15-20 min | 2 min (auto-extracted) | **18 min** |
| Create contracts | 30-45 min | 1 min (specflow-writer) | **40 min** |
| Design database schema | 20-30 min | 30s (migration-builder) | **25 min** |
| Write migration SQL | 15-20 min | 30s (migration-builder) | **17 min** |
| Implement auth logic | 90-120 min | 2-3 min (frontend-builder + edge-function-builder) | **100 min** |
| Write E2E tests | 45-60 min | 1-2 min (playwright-from-specflow) | **50 min** |
| Manual testing/debugging | 30-45 min | 0 min (tests auto-verify) | **35 min** |
| **Total** | **4-6 hours** | **20-30 minutes** | **~4 hours** |

**That's 8-12x faster.** And the contracts enforce quality automatically.

---

## What Makes This Different?

### Traditional Workflow
```
Write code → Manual review → Hope nothing breaks → Ship and pray
```

**Problems:**
- Review doesn't scale (human bottleneck)
- Violations invisible until production
- No enforcement of architectural rules

### Specflow Workflow
```
Define contracts → Agents generate → Tests enforce → Ship or stop
```

**Advantages:**
- Review is automated (contracts = compiler for architecture)
- Violations caught at test time (before merge)
- Enforcement is continuous (every PR)

**The compiler analogy holds:**
- TypeScript rejects `"hello" + 5` at compile time
- Specflow rejects "missing RLS policy" at test time

---

## Understanding the Output

When waves-controller finishes, you'll see:

### ✅ Success Output

```
🎉 WAVE EXECUTION COMPLETE

All issues closed: ✓
All contracts passing: ✓
All E2E tests passing: ✓
Ready to merge: ✓
```

**Meaning:** Safe to ship. Contracts verified. Architecture intact.

### ⚠️ Partial Success Output

```
⚠️ WAVE PARTIALLY COMPLETE

Issues closed: 2/3
Issues with failures: 1
  - Issue #43: E2E test failing (J-USER-LOGIN)
    Error: Timeout waiting for [data-testid="dashboard"]
    Fix: Verify dashboard route exists

Next steps: Fix failing tests, then re-run wave
```

**Meaning:** Some work complete, some blocked. Fix failures before proceeding.

### ❌ Failure Output

```
❌ WAVE EXECUTION FAILED

Contract violations detected:
  - AUTH-001: Password stored in plaintext (CRITICAL)
    File: src/features/auth/signup.ts:23
    Expected: bcrypt.hash(password)
    Actual: { password: password }

Build failed. Fix violations before proceeding.
```

**Meaning:** Architecture rules violated. Build is blocked. Fix required.

---

## Next Steps

After your first wave:

1. **Review the generated code** — Check `src/`, `tests/`, `docs/contracts/`
2. **Run tests locally** — `npm run test:e2e` to see Playwright in action
3. **Execute Wave 2** — Run "Execute waves" again for blocked issues
4. **[Understand Output](/getting-started/understanding-output/)** — Learn to read contracts and test results

---

## Common Questions

### "Can I edit the generated code?"

**Yes.** Specflow generates starting implementations. You can edit freely. Just ensure:
- Contract tests still pass (`npm test -- contracts`)
- E2E journeys still pass (`npm run test:e2e`)

If you violate a contract, the build will fail and tell you why.

### "What if I don't like what an agent generated?"

**Override or regenerate:**
1. Delete the generated code
2. Update the GitHub issue with more specific acceptance criteria
3. Re-run waves-controller
4. Agents will regenerate based on new spec

### "Can I run just one agent instead of the full wave?"

**Yes.** Use individual agents:
```
Read scripts/agents/specflow-writer.md, then create contracts for issue #42
```

But **waves-controller is faster** for multi-issue work.

### "How do I know if my issue will work?"

**Issues work best when they have:**
- Clear description of what to build
- Acceptance criteria (Gherkin format ideal)
- No dependencies on incomplete issues

waves-controller will analyze and warn if issues are ambiguous.

---

## Troubleshooting

### "No issues found"

**Solution:** Create a GitHub issue first, then run "Execute waves"

### "All issues blocked"

**Solution:** Check dependency chains. Fix circular dependencies or mark issues as independent.

### "Agent timeout"

**Solution:** Issue might be too complex. Break into smaller issues.

### "Tests fail after generation"

**Expected.** Agents generate tests that match spec. If spec is ambiguous, tests may fail. Fix spec, regenerate.

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
