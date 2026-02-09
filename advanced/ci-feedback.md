---
layout: default
title: "CI Feedback Loop"
parent: "Advanced"
nav_order: 7
permalink: /advanced/ci-feedback/
---

# CI Feedback Loop
{: .no_toc }

Automatic CI status reporting after every `git push`.
{: .fs-6 .fw-300 }

> CI feedback loop from [forge](https://github.com/ikennaokpala/forge) by [Ikenna N. Okpala](https://github.com/ikennaokpala).

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What It Does

After a successful `git push`, the `post-push-ci.sh` hook polls GitHub Actions for the latest CI run and reports whether it passed or failed. This closes the feedback loop between local development and remote CI without requiring you to open a browser.

```
git push  -->  Hook fires  -->  Polls gh run list  -->  Reports pass/fail
```

The hook is **advisory only** -- it always exits 0 and never blocks your workflow. If CI fails, it prints actionable next steps.

---

## How It Works

1. The hook is registered as a `PostToolUse` hook in `.claude/settings.json`
2. After any `git push` command succeeds, the hook activates
3. It polls `gh run list --limit 1` to get the latest CI run status
4. It waits up to `MAX_RETRIES * POLL_INTERVAL` seconds for the run to complete
5. Once complete, it reports the result with the run ID and URL

### Sequence

```
PostToolUse event (git push detected)
  |
  v
Pre-flight checks:
  - Is deferral flag set? --> exit 0
  - Is gh CLI installed?  --> warn and exit 0
  - Is gh authenticated?  --> warn and exit 0
  - Is this a git repo?   --> warn and exit 0
  |
  v
Poll loop (up to MAX_RETRIES):
  - Fetch latest run via gh run list
  - If completed --> display result and exit
  - If pending   --> sleep POLL_INTERVAL, retry
  |
  v
Exhausted retries --> print manual check instructions
```

---

## Example Output

### On Success

```
CI: Checking CI status...
CI: Passing (Contract Tests)
```

### On Failure

```
CI: Checking CI status...
CI: Failed - Contract Tests (see: gh run view 12345678)

CI failed. Options:
  1. gh run view 12345678 --log-failed  (view failure details)
  2. Fix and re-push
  3. Continue (CI is advisory)
```

### While Pending

```
CI: Checking CI status...
CI: Pending... (Contract Tests, attempt 1/5, next check in 10s)
CI: Pending... (Contract Tests, attempt 2/5, next check in 10s)
CI: Passing (Contract Tests)
```

### Exhausted Retries

```
CI: Checking CI status...
CI: Pending... (Contract Tests, attempt 1/5, next check in 10s)
...
CI: Still pending after 5 checks (50s). Check manually:
CI:   gh run view 12345678
CI:   https://github.com/org/repo/actions/runs/12345678
```

---

## Installation

### Via install-hooks.sh (Recommended)

The CI feedback hook is included in the standard Specflow hooks installation:

```bash
bash Specflow/install-hooks.sh /path/to/your/project
```

This installs `post-push-ci.sh` alongside the journey verification hooks and configures `.claude/settings.json` automatically.

### Manual Installation

1. Copy the hook script:

```bash
mkdir -p .claude/hooks
cp Specflow/templates/hooks/post-push-ci.sh .claude/hooks/
chmod +x .claude/hooks/post-push-ci.sh
```

2. Add the hook to `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/post-push-ci.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SPECFLOW_CI_POLL_INTERVAL` | `10` | Seconds between status polls |
| `SPECFLOW_CI_MAX_RETRIES` | `5` | Maximum number of poll attempts |

**Total wait time** = `POLL_INTERVAL * MAX_RETRIES` (default: 50 seconds).

Adjust these based on your CI speed:

```bash
# Fast CI (< 2 minutes)
export SPECFLOW_CI_POLL_INTERVAL=10
export SPECFLOW_CI_MAX_RETRIES=5

# Slow CI (5+ minutes)
export SPECFLOW_CI_POLL_INTERVAL=30
export SPECFLOW_CI_MAX_RETRIES=10
```

---

## Deferring CI Checks

To temporarily skip CI status checks:

```bash
# Disable
touch .claude/.defer-ci-check

# Re-enable
rm .claude/.defer-ci-check
```

When deferred, the hook prints a single message and exits:

```
CI: CI check deferred. Remove .claude/.defer-ci-check to re-enable.
```

---

## Requirements

| Requirement | Purpose |
|-------------|---------|
| `gh` CLI | Fetches CI run status via GitHub API |
| `gh auth login` | Authentication for API access |
| Git remote `origin` | Identifies the repository |

If any requirement is missing, the hook prints a warning and exits without blocking:

```
CI: [warn] gh CLI not found. Install with: brew install gh (mac) or apt install gh (linux)
CI: [warn] Skipping CI status check.
```

---

## Advisory Only

The CI feedback hook **always exits 0**. It is strictly informational and will never:

- Block a push
- Prevent further commands
- Cause Claude Code to halt

This is by design. The authoritative CI gate is the GitHub Actions workflow itself (which blocks PR merges). The hook simply brings that feedback into your terminal session so you can act on it sooner.

For blocking enforcement, see [CI/CD Integration](/advanced/ci-integration/) which covers the GitHub Actions pipeline configuration.

---

## Standalone Usage

The hook can also be run directly from the terminal:

```bash
bash .claude/hooks/post-push-ci.sh
```

When run interactively (TTY detected), it skips the PostToolUse command detection and goes straight to polling CI status.

---

## Related Pages

- [Journey Verification Hooks](/advanced/journey-verification-hooks/) -- Build/commit hooks that run targeted E2E tests
- [CI/CD Integration](/advanced/ci-integration/) -- GitHub Actions pipeline configuration
- [Post-Mortem Learning System](/advanced/learning-system/) -- How violations feed back into agent context

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
