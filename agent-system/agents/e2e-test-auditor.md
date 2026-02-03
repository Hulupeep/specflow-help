# E2E Test Auditor Agent

## Purpose

Automatically analyze Playwright test results to catch silent failures that DOM-only assertions miss. Uses LLM to detect when features appear to work (elements visible) but are actually broken (console errors, API failures, wrong data).

## When to Use

**AUTO-TRIGGER after every E2E test run:**
- `npm run test:e2e` completes
- CI/CD pipeline runs E2E tests
- Developer runs single test file

## What It Does

1. **Captures comprehensive test artifacts:**
   - Console logs (all levels: error, warning, info, debug)
   - Network requests/responses (XHR, Fetch)
   - Failed API calls (4xx, 5xx status codes)
   - Page screenshots on failure
   - Browser storage state (localStorage, cookies)

2. **LLM analysis:**
   - Reads console logs for patterns indicating failure
   - Checks for "Failed to fetch", "404", "500", "undefined is not a function"
   - Verifies API responses match expected schema
   - Detects "silent failures" (test passes but feature broken)

3. **Generates failure report:**
   - Severity: P0 (blocks release), P1 (important), P2 (minor)
   - Root cause hypothesis
   - Links to relevant code files
   - Suggested fixes

## Example: Catching the Demo Mode Bug

**Without LLM auditor:**
```
✅ Test: Demo mode shows scheduler
  ✓ Page loaded
  ✓ "Staff Availability" heading visible
  ✓ Test PASSED
```

**With LLM auditor:**
```
✅ Test: Demo mode shows scheduler (DOM assertions PASSED)
❌ Silent Failure Detected (P0 - Blocks Release)

Console Errors:
  [ERROR] Failed to fetch employees: column employees.contracted_hours does not exist
  [ERROR] Failed to fetch shifts: ...

Root Cause:
  TypeScript build errors blocking Vercel deployment.
  Production running 6-hour-old code with schema mismatch.

Suggested Fix:
  1. Fix TypeScript errors in useShiftsByStaff.ts
  2. Verify build succeeds locally: npm run build
  3. Push and wait for Vercel deployment
  4. Re-test production

Affected Files:
  - src/features/scheduler/hooks/useShiftsByStaff.ts (TypeScript errors)
  - src/features/scheduler/hooks/useStaffAvailability.ts (outdated query)

GitHub Issue: Created #306 with full details
```

## Benefits

1. **Catch silent failures immediately** (would have found #306 in CI before push)
2. **No more "looks good but broken"** false positives
3. **Automatic root cause analysis** (saves 2+ hours debugging)
4. **Links to relevant code** (faster fixes)
5. **Severity triage** (P0 vs P2 classification)

## Implementation Phases

**Phase 1:** Add console capture to all tests
**Phase 2:** LLM analysis on failures
**Phase 3:** Auto-create GitHub issues for P0 failures

See full implementation in docs/specs/e2e_test_auditor.md

---

**Created by:** Claude Sonnet 4.5  
**Based on:** Real bug caught during #306 debugging
