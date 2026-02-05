---
layout: default
title: Journey Verification Hooks
parent: Advanced
nav_order: 4
---

# Journey Verification Hooks

Automatically run E2E tests at the right moments without being asked.

## Why Hooks Exist

There are two ways to make Claude run tests:

### Option A: You Ask Every Time

```
User: "Run the playwright tests"
Claude: [runs tests]
```

**Problem:** You'll forget to ask. Production breaks.

### Option B: Hooks Make It Automatic

```
[Claude runs build]
[HOOK fires automatically]
Claude: "Build passed. Running journey tests..."
```

**Solution:** You can't forget because hooks do it for you.

---

## The Real Problem

Without hooks, this happens:

1. You implement a feature
2. `pnpm build` passes ✅
3. You say "done"
4. Code deploys to production
5. **Production is broken** 💥
6. You discover it hours later

With hooks:

1. You implement a feature
2. `pnpm build` passes
3. **[HOOK]** Claude automatically runs E2E tests
4. Tests fail - Claude reports the issue
5. You fix it BEFORE deploying
6. **Production works** ✅

---

## Local vs Production Testing

This is critical: **tests must run in the right environment**.

### The Deploy Pipeline

```
git push → GitHub → Vercel/Netlify auto-deploys → PRODUCTION CHANGES
                                                   ↑
                                             Tests MUST run here
```

### When to Test Where

| Trigger | Environment | URL |
|---------|-------------|-----|
| PRE-BUILD | Local | `localhost:3000` |
| POST-BUILD | Local | `localhost:3000` |
| POST-COMMIT | **Production** | `https://yourapp.com` |
| POST-MIGRATION | **Production** | `https://yourapp.com` |

**After `git push`, you MUST test against production**, not localhost.

---

## Mandatory Reporting

Claude must ALWAYS report:

### 1. WHERE Tests Ran

```
❌ BAD:  "Tests passed"
✅ GOOD: "Tests passed against PRODUCTION (https://www.yourapp.com)"
```

### 2. WHICH Tests Ran

```
❌ BAD:  "E2E tests passed"
✅ GOOD: "Ran: signup.spec.ts, login.spec.ts, checkout.spec.ts"
```

### 3. HOW MANY Tests

```
❌ BAD:  "Tests passed"
✅ GOOD: "12/12 tests passed (0 failed, 0 skipped)"
```

### 4. What SKIPPED Means

**Skipped ≠ Passed.** Skipped tests didn't run.

```
❌ BAD:  "10/12 passed, 2 skipped" (sounds fine!)

✅ GOOD: "10/12 passed, 0 failed, 2 SKIPPED
         SKIPPED TESTS:
         - signup.spec.ts:45 - @skip tag (TODO: remove)
         - checkout.spec.ts:78 - missing STRIPE_KEY env var"
```

Every skip needs an explanation. "Skipped" often hides problems.

---

## Report Template

After every test run:

```
══════════════════════════════════════════════════════════════
E2E TEST REPORT
══════════════════════════════════════════════════════════════

ENVIRONMENT: PRODUCTION (https://www.yourapp.com)

TESTS RUN:
  - tests/e2e/auth/signup.spec.ts
  - tests/e2e/auth/login.spec.ts

RESULTS: 18/20 passed, 1 failed, 1 skipped

FAILURES:
  ✗ signup.spec.ts:67 "should create user"
    Error: API returned 400

SKIPPED (with reasons):
  ⊘ checkout.spec.ts:120 - @skip: Stripe not configured

CONSOLE ERRORS:
  - [HTTP 400] POST /api/signup

══════════════════════════════════════════════════════════════
```

---

## Installation

```bash
# From Specflow repo
bash install-hooks.sh /path/to/your/project
```

This installs:
- `.claude/settings.json` - Hook triggers
- `.claude/hooks/journey-verification.md` - Behavior spec

Then add configuration to your `CLAUDE.md`:

```markdown
## Test Configuration

- **Package Manager:** pnpm
- **Test Command:** `pnpm test:e2e`
- **Local URL:** `http://localhost:5173`
- **Production URL:** `https://www.yourapp.com`
- **Deploy Platform:** Vercel
- **Deploy Wait:** 90 seconds
```

---

## Anti-Patterns

### People-Pleasing Reports

```
❌ "Tests mostly passed with a few minor skips"
✅ "12/15 passed, 2 failed, 1 skipped. Failures: ..."
```

### Hiding Behind "Skipped"

```
❌ "All tests passed (5 skipped)"
✅ "10/15 passed. 5 SKIPPED - here's why each one..."
```

### Vague Environment

```
❌ "E2E tests passed"
✅ "E2E tests passed against PRODUCTION"
```

### Missing Counts

```
❌ "Journey tests passed"
✅ "8/8 journey tests passed"
```

---

## Summary

| Without Hooks | With Hooks |
|---------------|------------|
| You ask "run tests" | Automatic |
| You forget → prod breaks | Can't forget |
| "Tests passed" | WHERE/WHAT/HOW MANY |
| Skipped = ignored | Skipped = explained |
| Local only | Local + Production |
