---
layout: default
title: "Security & Accessibility Gates"
parent: "Core Concepts"
nav_order: 4
permalink: /core-concepts/security-accessibility/
---

# Security & Accessibility Gates
{: .no_toc }

Default quality gates for security and accessibility, enforced as contract rules.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why Default Gates?

Most projects share the same fundamental security and accessibility requirements. Instead of writing these rules from scratch for every project, Specflow ships default contract templates covering the most common violations.

These defaults are:
- **Ready to use** -- copy the templates and run tests immediately
- **Extensible** -- add project-specific rules alongside the defaults
- **Overridable** -- disable individual rules via `.specflow/config.json`

> Adapted from [forge](https://github.com/ikennaokpala/forge) by Ikenna N. Okpala. The security and accessibility gate patterns were adapted for Specflow's YAML contract format.

---

## Security Rules (SEC-xxx)

### SEC-001: No Hardcoded Secrets

**Rule:** Source code must not contain hardcoded passwords, API keys, tokens, private keys, GitHub PATs, or Slack bot tokens.

```yaml
- id: SEC-001
  title: "No hardcoded secrets"
  scope:
    - "src/**/*.ts"
    - "src/**/*.tsx"
    - "src/**/*.js"
    - "!src/**/*.test.*"
  behavior:
    forbidden_patterns:
      - pattern: /(password|passwd|pwd)\s*[:=]\s*['"][^'"]{4,}['"]/i
        message: "Hardcoded password detected"
      - pattern: /(api[_-]?key|apikey)\s*[:=]\s*['"][^'"]{8,}['"]/i
        message: "Hardcoded API key detected"
      - pattern: /(secret|token)\s*[:=]\s*['"][^'"]{8,}['"]/i
        message: "Hardcoded secret or token detected"
      - pattern: /-----BEGIN\s+(RSA\s+)?PRIVATE\s+KEY-----/
        message: "Private key embedded in source code"
      - pattern: /ghp_[A-Za-z0-9]{36}/
        message: "GitHub personal access token detected"
      - pattern: /xoxb-[0-9]+-[A-Za-z0-9]+/
        message: "Slack bot token detected"
    auto_fix:
      strategy: "replace"
      hint: "Replace with environment variable reference (process.env.VARIABLE_NAME)"
```

### SEC-002: No Raw SQL String Concatenation

**Rule:** SQL queries must not use string concatenation or template literals with user input. Use parameterized queries.

```yaml
- id: SEC-002
  title: "No raw SQL string concatenation"
  scope:
    - "src/**/*.ts"
    - "!src/**/*.test.*"
  behavior:
    forbidden_patterns:
      - pattern: /(query|execute)\s*\(\s*`[^`]*\$\{/
        message: "Template literal in SQL query (SQL injection risk)"
      - pattern: /(query|execute)\s*\(\s*['"][^'"]*['"]\s*\+/
        message: "String concatenation in SQL query (SQL injection risk)"
```

### SEC-003: No innerHTML with Unsanitized Content

**Rule:** Do not use `dangerouslySetInnerHTML` or `.innerHTML` without a sanitization step.

```yaml
- id: SEC-003
  title: "No innerHTML with unsanitized content"
  scope:
    - "src/**/*.tsx"
    - "src/**/*.ts"
  behavior:
    forbidden_patterns:
      - pattern: /dangerouslySetInnerHTML/
        message: "dangerouslySetInnerHTML usage (XSS risk)"
      - pattern: /\.innerHTML\s*=/
        message: "Direct innerHTML assignment (XSS risk)"
    auto_fix:
      strategy: "replace"
      hint: "Use textContent for plain text, or DOMPurify.sanitize() for HTML content"
```

### SEC-004: No eval or Function Constructor

**Rule:** `eval()` and `new Function()` must not be used. They enable arbitrary code execution.

```yaml
- id: SEC-004
  title: "No eval or Function constructor"
  scope:
    - "src/**/*.ts"
    - "src/**/*.tsx"
    - "src/**/*.js"
  behavior:
    forbidden_patterns:
      - pattern: /\beval\s*\(/
        message: "eval() usage (arbitrary code execution risk)"
      - pattern: /new\s+Function\s*\(/
        message: "Function constructor (arbitrary code execution risk)"
```

### SEC-005: No Path Traversal

**Rule:** File system operations (`readFile`, `writeFile`) must use `path.join()` or `path.resolve()` to prevent directory traversal attacks.

```yaml
- id: SEC-005
  title: "No path traversal"
  scope:
    - "src/**/*.ts"
    - "!src/**/*.test.*"
  behavior:
    forbidden_patterns:
      - pattern: /(readFile|writeFile|readFileSync|writeFileSync)\s*\([^)]*\+/
        message: "File operation with string concatenation (path traversal risk)"
    required_patterns:
      - pattern: /path\.(join|resolve)\s*\(/
        message: "File operations must use path.join() or path.resolve()"
```

---

## Accessibility Rules (A11Y-xxx)

### A11Y-001: Images Must Have Alt Text

**Rule:** All `<img>` elements must include an `alt` attribute.

```yaml
- id: A11Y-001
  title: "Images must have alt text"
  scope:
    - "src/**/*.tsx"
  behavior:
    forbidden_patterns:
      - pattern: /<img\s+(?![^>]*\balt\b)[^>]*>/
        message: "Image element missing alt attribute"
```

### A11Y-002: Icon Buttons Must Have aria-label

**Rule:** Buttons that contain only icons (no visible text) must have an `aria-label` attribute.

```yaml
- id: A11Y-002
  title: "Icon buttons must have aria-label"
  scope:
    - "src/**/*.tsx"
  behavior:
    forbidden_patterns:
      - pattern: /<button[^>]*>\s*<(svg|icon|i)\b[^>]*>\s*<\/button>/
        message: "Icon-only button missing aria-label"
```

### A11Y-003: Form Inputs Need Labels

**Rule:** Every `<input>`, `<select>`, and `<textarea>` must have an associated `<label>` or `aria-label`.

```yaml
- id: A11Y-003
  title: "Form inputs need labels"
  scope:
    - "src/**/*.tsx"
  behavior:
    forbidden_patterns:
      - pattern: /<input\s+(?![^>]*\b(aria-label|id)\b)[^>]*>/
        message: "Input element missing label association (add id + label, or aria-label)"
```

### A11Y-004: No Positive tabindex

**Rule:** `tabindex` values must be `0` or `-1`. Positive values break the natural tab order.

```yaml
- id: A11Y-004
  title: "No positive tabindex"
  scope:
    - "src/**/*.tsx"
  behavior:
    forbidden_patterns:
      - pattern: /tabindex\s*=\s*["']?[1-9]/i
        message: "Positive tabindex breaks natural tab order (use 0 or -1)"
      - pattern: /tabIndex\s*=\s*\{[1-9]/
        message: "Positive tabIndex breaks natural tab order (use 0 or -1)"
```

---

## Quick Reference

| ID | Rule | Category |
|----|------|----------|
| SEC-001 | No hardcoded secrets | Security |
| SEC-002 | No raw SQL string concatenation | Security |
| SEC-003 | No innerHTML with unsanitized content | Security |
| SEC-004 | No eval or Function constructor | Security |
| SEC-005 | No path traversal | Security |
| A11Y-001 | Images must have alt text | Accessibility |
| A11Y-002 | Icon buttons must have aria-label | Accessibility |
| A11Y-003 | Form inputs need labels | Accessibility |
| A11Y-004 | No positive tabindex | Accessibility |

---

## Installation

Copy the default contract templates to your project:

```bash
cp Specflow/templates/contracts/security_defaults.yml docs/contracts/
cp Specflow/templates/contracts/accessibility_defaults.yml docs/contracts/
```

Then run the contract tests to verify:

```bash
npm test -- contracts
```

---

## Extending with Project-Specific Rules

Add your own rules alongside the defaults. Use the same `SEC-xxx` or `A11Y-xxx` prefix for consistency, or create a new domain prefix.

```yaml
# docs/contracts/feature_security_custom.yml
contract_meta:
  id: security_custom
  version: 1
  created_from_spec: "docs/specs/security.md"
  covers_reqs:
    - SEC-006
    - SEC-007
  owner: "security-team"

rules:
  non_negotiable:
    - id: SEC-006
      title: "All API responses must strip internal error details"
      scope:
        - "src/routes/**/*.ts"
      behavior:
        forbidden_patterns:
          - pattern: /res\.(json|send)\(\s*\{[^}]*stack/
            message: "Stack trace exposed in API response"

    - id: SEC-007
      title: "CORS must not use wildcard in production"
      scope:
        - "src/config/**/*.ts"
        - "src/middleware/**/*.ts"
      behavior:
        forbidden_patterns:
          - pattern: /origin\s*:\s*['"]\*['"]/
            message: "Wildcard CORS origin in production config"
```

---

## Overriding or Disabling Rules

Disable specific rules in `.specflow/config.json`:

```json
{
  "disabled_rules": ["A11Y-004"],
  "rule_overrides": {
    "SEC-003": {
      "severity": "important",
      "note": "We use DOMPurify globally; innerHTML is safe in our codebase"
    }
  }
}
```

| Override | Effect |
|----------|--------|
| `disabled_rules` | Rule is skipped entirely during contract tests |
| `rule_overrides.severity` | Changes the rule from `critical` to `important` (warns but does not block) |
| `rule_overrides.note` | Documents the reason for the override |

---

## Integration with specflow-writer

When the `specflow-writer` agent generates contracts for new features, it automatically references the SEC and A11Y defaults:

1. If the feature involves user input, SEC-002 (SQL injection) and SEC-003 (XSS) are referenced
2. If the feature has UI components, A11Y-001 through A11Y-004 are checked
3. The generated contract includes `covers_reqs` entries for the relevant default rules

This means new features inherit baseline security and accessibility coverage without any extra configuration.

---

## Related Pages

- [What Are Contracts?](/core-concepts/contracts/) -- How contracts enforce architecture
- [Contract Schema](/reference/contract-schema/) -- Full YAML schema reference
- [Self-Healing Fix Loops](/advanced/self-healing/) -- Auto-fix for SEC/A11Y violations
- [CI/CD Integration](/advanced/ci-integration/) -- Running SEC/A11Y gates in CI

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
