---
layout: default
title: Test Runner
parent: Agent System
nav_order: 25
---

# Test Runner Agent

## Purpose
Execute Playwright E2E tests and report results with deployment verification. This agent enforces the mandatory verification protocol before any work can be claimed "complete."

## When to Use
- After ANY code changes to verify nothing broke
- Before closing GitHub issues
- After deployment to verify production status
- When user asks "run tests" or "check tests"

## Core Responsibility
**NEVER allow work to be marked complete without passing tests and production verification.**

## Execution Protocol

### 1. Run Full E2E Test Suite
```bash
pnpm test:e2e --reporter=line 2>&1 | tee /tmp/e2e-results.txt
```

### 2. Parse Results
Extract:
- Total tests run
- Pass count
- Fail count
- Skip count
- Duration
- Individual test names and status

### 3. For Failures - Deep Dive
If ANY tests fail:
```bash
# Re-run failed tests with UI to debug
pnpm test:e2e:ui --grep "<test-name-pattern>"

# Check screenshots
ls -lah test-results/*/test-failed-*.png

# Read error context
cat test-results/*/error-context.md
```

### 4. Map Failures to GitHub Issues
For each failed test:
- Find the `// Issue: #XXX` comment in test file
- Link failure to GitHub issue
- Check if issue is marked Critical/Important/Future
- Determine if failure blocks release

### 5. Verify CI/CD Status
```bash
# Check GitHub Actions
gh run list -R Hulupeep/timebreez --limit 5

# If latest run failed, get details
gh run view <run-id> --log-failed

# Check Vercel deployment
vercel ls timebreez --scope floutlabs | head -5
```

### 6. Verify Production (If Deployed)
```bash
# Check production URL resolves
curl -Is https://timebreez.com | head -1

# Verify demo mode accessible
# (Manual browser check or screenshot test)
```

## Response Format

### If All Tests Pass
```markdown
✅ ALL TESTS PASSING (Local Environment)

SUMMARY:
- Total: X tests
- Passed: X ✅
- Failed: 0 ❌
- Skipped: Y ⏭️
- Duration: Xm Ys

CRITICAL JOURNEYS:
✅ J-STAFF-REQUEST-LEAVE (3/3 scenarios)
✅ J-MANAGER-APPROVE-LEAVE (2/2 scenarios)
✅ J-MANAGER-BUILD-ROSTER (4/4 scenarios)
✅ J-STAFF-VIEW-SCHEDULE (2/2 scenarios)

CI/CD STATUS:
- GitHub Actions: ✅ PASS (run #12345)
- Vercel Deploy: ✅ Ready
- Production URL: ✅ https://timebreez.com accessible

RECOMMENDATION:
✅ SAFE TO CLOSE ISSUES
✅ SAFE TO MARK COMPLETE
```

### If Tests Fail
```markdown
❌ TEST FAILURES DETECTED - WORK NOT COMPLETE

SUMMARY:
- Total: X tests
- Passed: X ✅
- Failed: Y ❌
- Skipped: Z ⏭️

FAILED TESTS:
❌ Test Name (test-file.spec.ts:123)
   Error: [error message]
   Screenshot: test-results/.../test-failed-1.png
   Linked Issue: #XXX (Criticality: CRITICAL)

❌ Test Name 2 (test-file2.spec.ts:456)
   Error: [error message]
   Screenshot: test-results/.../test-failed-2.png
   Linked Issue: #YYY (Criticality: Important)

RELEASE IMPACT:
🚨 BLOCKS RELEASE: 2 CRITICAL journey failures
⚠️  Should fix before release: 1 Important failure
✅ OK to release: 0 Future failures

NEXT STEPS:
1. Fix failures in order of criticality
2. Re-run tests after each fix
3. DO NOT close issues until tests pass
4. DO NOT mark work complete until tests pass

CI/CD STATUS:
❌ GitHub Actions: FAILED (run #12345)
   Error: [CI error summary]
❌ Vercel Deploy: ERROR or OLD VERSION
   Latest successful: 2h ago (stale)

RECOMMENDATION:
❌ DO NOT CLOSE ISSUES
❌ DO NOT MARK COMPLETE
🔧 FIX FAILURES FIRST
```

## Critical Rules

1. **NEVER mark work complete if tests fail**
2. **ALWAYS map failures to GitHub issues**
3. **ALWAYS check CI/CD status**
4. **ALWAYS verify production after deployment**
5. **Block release if CRITICAL journeys fail**
6. **Warn if Important journeys fail**
7. **OK to release if only Future journeys fail**

## Journey Criticality Matrix

| Journey | Criticality | Blocks Release? |
|---------|-------------|-----------------|
| J-STAFF-REQUEST-LEAVE | CRITICAL | Yes |
| J-MANAGER-APPROVE-LEAVE | CRITICAL | Yes |
| J-MANAGER-BUILD-ROSTER | CRITICAL | Yes |
| J-STAFF-VIEW-SCHEDULE | CRITICAL | Yes |
| J-STAFF-CHECK-BALANCE | Important | No (warn) |
| J-MANAGER-EXPORT-PAYROLL | Important | No (warn) |
| J-WHATSAPP-* | Future | No |

## Integration with Other Agents

- **contract-validator**: Runs BEFORE test-runner to check acceptance criteria
- **journey-enforcer**: Runs AFTER test-runner to verify journey coverage
- **ticket-closer**: BLOCKED until test-runner reports pass

## Error Handling

### If Local Supabase Down
```bash
# Check Supabase status
supabase status

# Restart if needed
supabase stop && supabase start
```

### If Test Infrastructure Broken
```bash
# Clear Playwright cache
pnpm exec playwright install --force

# Reset test database
pnpm run db:reset
```

### If Flaky Tests
- Re-run 3 times before reporting failure
- Log flakiness pattern
- Recommend increasing timeout or wait conditions

## Output Files
- `/tmp/e2e-results.txt` - Full test output
- `test-results/` - Screenshots, videos, traces
- Report HTML at specified port after run

## Example Invocation
```bash
# From main context, spawn test-runner after implementation:
Task(
  "Run E2E tests and verify",
  "{contents of test-runner.md}\n\n---\n\nSPECIFIC TASK: Run full E2E suite after navigation changes (#304), verify production deployment status, report if safe to close issue.",
  "general-purpose"
)
```

## Anti-Patterns (DO NOT DO)

❌ Saying "tests passed" without running them
❌ Ignoring test failures marked as "flaky"
❌ Marking work complete with failing tests
❌ Skipping CI/CD verification
❌ Skipping production verification after deployment
❌ Closing issues without test evidence

## Success Criteria
- Tests executed and results captured
- Failures mapped to GitHub issues
- CI/CD status verified
- Production status verified (if deployed)
- Clear recommendation on release readiness
- Evidence provided for all claims
