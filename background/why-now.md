---
layout: default
title: Why Now?
parent: Background
nav_order: 2
permalink: /background/why-now/
---

# Why Now? The LLM Shift
{: .no_toc }

LLMs changed the failure mode. Old tools became insufficient.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The Shift

### Before LLMs (Pre-2023)

| Aspect | Reality | Implication |
|--------|---------|-------------|
| **Generation** | Slow (50-100 lines/day) | Humans are the bottleneck |
| **Execution** | Deterministic | Same inputs → same outputs |
| **Review** | Scales poorly | But manageable (you review what you generate) |
| **Violations** | Rare bugs | Exceptions, not the norm |
| **Architecture drift** | Visible | Gradual, detectable in review |

**Old tools worked:**
- Code review (manual verification)
- TDD (write tests first)
- Linters (catch syntax issues)
- DbC (optional, heavyweight)

**Why they worked:** Review scaled. Humans could keep up with humans.

---

### After LLMs (2023+)

| Aspect | Reality | Implication |
|--------|---------|-------------|
| **Generation** | Infinite (1000s of lines/hour) | **Review is the bottleneck** |
| **Execution** | Probabilistic | Same prompt → different outputs |
| **Review** | Doesn't scale | **Humans review 10x what they generate** |
| **Violations** | Normal behavior | Drift, hallucination, misalignment |
| **Architecture drift** | Invisible | Happens instantly, undetectable until too late |

**Old tools became insufficient:**
- Code review (can't review 10x output)
- TDD (who writes the tests? LLM or human?)
- Linters (only catch syntax, not architecture)
- DbC (assumes aligned execution)

**New tool needed:** **Specflow** (automated containment).

---

## The Bottleneck Reversal

### Pre-LLM Workflow

```
[SLOW] Human writes code (4-6 hours)
  ↓
[FAST] Code review (30 min)
  ↓
Ship
```

**Bottleneck:** Generation (human typing)

---

### Post-LLM Workflow (Without Specflow)

```
[FAST] LLM generates code (5 min)
  ↓
[SLOW] Human reviews 1000s of lines (2-3 hours)
  ↓
[SLOWER] Human fixes violations (1-2 hours)
  ↓
[REPEAT] LLM regenerates, human re-reviews...
  ↓
Ship (or give up)
```

**Bottleneck:** Review (human reading)

**Problem:** Review doesn't scale with infinite generation.

---

### Post-LLM Workflow (With Specflow)

```
[INSTANT] Human defines contracts (10 min)
  ↓
[FAST] LLM generates code (5 min)
  ↓
[INSTANT] Tests enforce contracts (2 min)
  ↓
Pass → Ship
Fail → Stop (LLM regenerates or human intervenes)
```

**Bottleneck:** Eliminated (automated verification)

**Time:** 20-30 min (vs 4-6 hours manual)

---

## The New Failure Mode

### Old Failure Mode: Bugs

**Example:**
```python
def divide(a, b):
  return a / b  # Bug: No check for b == 0
```

**How it fails:** Crashes at runtime when `b = 0`

**How old tools catch it:**
- Unit test: `assert divide(10, 0) raises ZeroDivisionError`
- Code review: Reviewer notices missing check

**Characteristics:**
- Deterministic (always fails on `b = 0`)
- Rare (edge case)
- Visible (crashes loudly)

---

### New Failure Mode: Drift

**Example:**
```yaml
# Contract: AUTH-001
rule: "Passwords MUST be hashed using bcrypt before storage"
```

**LLM generates (first attempt):**
```python
def signup(email, password):
  user = User(email=email, password=hash_password(password))
  db.save(user)

def hash_password(pw):
  return hashlib.sha256(pw.encode()).hexdigest()  # ❌ SHA256, not bcrypt
```

**How it fails:** Silent drift (SHA256 is weaker than bcrypt, but still "works")

**How old tools miss it:**
- Unit tests pass (SHA256 is a valid hash)
- Code review misses it (looks reasonable)
- App "works" (no crashes)

**How Specflow catches it:**
```typescript
// Contract test
it('AUTH-001: Passwords hashed with bcrypt', () => {
  const code = readFileSync('signup.py')
  if (!code.includes('bcrypt')) {
    throw new Error('CONTRACT VIOLATION: AUTH-001')
  }
})
```

**Test fails:** `❌ AUTH-001: Expected bcrypt, found SHA256`

**Characteristics:**
- Probabilistic (LLM might use bcrypt next time)
- Common (LLMs hallucinate alternatives)
- Invisible (no runtime error, just weaker security)

---

## Probabilistic vs Deterministic Execution

### Deterministic (Human Code)

```python
# Same function, same inputs → always same output
def add(a, b):
  return a + b

add(2, 3)  # Always 5
add(2, 3)  # Always 5
add(2, 3)  # Always 5
```

**Testing strategy:** Write test once. If it passes, it always passes.

---

### Probabilistic (LLM Code)

```
Prompt: "Create a login function with bcrypt hashing"

Run 1:
  def login(email, password):
    hashed = bcrypt.hash(password)  # ✅ Correct

Run 2:
  def login(email, password):
    hashed = hashlib.sha256(password)  # ❌ Wrong

Run 3:
  def login(email, password):
    hashed = pbkdf2_hmac('sha256', password)  # ⚠️ Also wrong
```

**Same prompt, different outputs.**

**Testing strategy:** Tests must enforce **contracts**, not just "works".

---

## Why Review Doesn't Scale

### Manual Review (Pre-LLM)

**Scenario:** Human writes 100 lines of code

**Review time:**
- Read 100 lines: 10 min
- Identify issues: 5 min
- Comment: 5 min
- **Total:** 20 min

**Reviewer throughput:** ~300 lines/hour

**Bottleneck:** None (generation is slower than review)

---

### Manual Review (Post-LLM)

**Scenario:** LLM writes 1000 lines of code in 5 minutes

**Review time:**
- Read 1000 lines: 100 min
- Identify issues: 30 min
- Comment: 20 min
- **Total:** 150 min (2.5 hours)

**Reviewer throughput:** ~400 lines/hour

**Bottleneck:** **Review** (generation is 12x faster than review)

**Problem:** Humans can't keep up with LLMs.

---

### Automated Review (Specflow)

**Scenario:** LLM writes 1000 lines of code in 5 minutes

**Verification time:**
- Contract tests: 30s (scan for patterns)
- E2E tests: 2 min (run journeys)
- **Total:** 2.5 min

**Throughput:** ~24,000 lines/hour (automated)

**Bottleneck:** Eliminated

**Result:** Ship in 8 min instead of 3 hours.

---

## The Trust Reversal

### Pre-LLM: Trust, Then Verify (Maybe)

```
Human writes code → Assume it's correct → Ship
                    ↓
           (Optional: Add tests later)
```

**Mindset:** "I trust myself. Tests are for edge cases."

**Failure mode:** Bugs slip through. But rare.

---

### Post-LLM: Verify, Then Trust

```
LLM writes code → Assume it's wrong → Verify via tests → Ship or reject
```

**Mindset:** "I don't trust the LLM. Tests are the boundary."

**Failure mode:** Violations are normal. Tests catch them.

**Specflow enforces this mindset.**

---

## The Economic Shift

### Before: Cost of Generation >> Cost of Review

| Task | Cost (Time) |
|------|-------------|
| Write 1000 lines | 2-3 days (human) |
| Review 1000 lines | 2-3 hours (human) |

**Optimization:** Reduce generation cost (IDEs, autocomplete, snippets)

**Tools:** IntelliSense, Copilot (pre-Specflow), Tabnine

---

### After: Cost of Review >> Cost of Generation

| Task | Cost (Time) |
|------|-------------|
| Write 1000 lines | 5 min (LLM) |
| Review 1000 lines | 2-3 hours (human) |

**Optimization:** Reduce review cost (automated verification)

**Tools:** Specflow, contracts, E2E tests

**The bottleneck flipped.**

---

## Why Old Tools Failed

### Code Review

**Problem:** Doesn't scale with infinite generation

**Example:** Reviewing 1000 lines generated in 5 min takes 2-3 hours

**Specflow solution:** Automated contract verification (2-3 min)

---

### TDD

**Problem:** Who writes the tests? LLM or human?

**If human writes tests:**
- Slower than LLM generation (defeats the purpose)

**If LLM writes tests:**
- Tests might be wrong too (who verifies the tests?)

**Specflow solution:** **Contracts define what to test.** Agents generate tests from contracts.

---

### Linters (ESLint, etc.)

**Problem:** Only catch syntax and patterns, not architecture

**Example:** Linter catches `unused variable`, but not "missing RLS policy"

**Specflow solution:** Custom contract tests for architecture (YAML rules → TypeScript tests)

---

### Design by Contract (DbC)

**Problem:** Assumes author = executor (aligned execution)

**Example:** DbC postcondition: `balance = old_balance + amount`

**If LLM violates:** Assertion fails at runtime (too late)

**Specflow solution:** Verify at test time, before deployment

---

## The Forgotten Lesson

> **"We already knew how to control untrusted execution. We just forgot."**

### 1980s: Untrusted Executors Were Common

**Examples:**
- Student code submissions (grading systems needed verification)
- Compiler bugs (needed static analysis)
- Concurrent systems (race conditions, formal methods)

**Solution:** Contracts, invariants, automated verification

---

### 1990-2020: Execution Became Trusted

**Why:**
- IDEs caught syntax errors
- Compilers were deterministic
- Humans wrote code (aligned with intent)

**Result:** Contracts became "optional" or "heavyweight"

**DbC faded. Formal methods stayed academic.**

---

### 2023+: Untrusted Execution Returns

**Why:**
- LLMs generate code (probabilistic, not aligned)
- Violations are normal (drift, hallucination)
- Review doesn't scale (humans can't keep up)

**Solution:** **Rediscover contracts.** Specflow applies them to LLMs.

---

## The Timing

### Why Not 2020? (Pre-LLM Era)

**Code generation existed (Codex, early Copilot), but:**
- Limited scope (autocomplete, not full features)
- Humans still bottleneck (slow typing)
- Review still scaled (you review what you generate)

**Contracts not urgent.**

---

### Why Now? (2023+)

**LLMs crossed the threshold:**
- Full features in minutes (not just autocomplete)
- Infinite generation (faster than review)
- Violations are normal (not rare bugs)

**Contracts became urgent.**

**Specflow fills the gap.**

---

## Next Steps

- **[Origins & Academic Foundation](/background/origins/)** — Where Specflow came from (DbC, QuickCheck, TDD)
- **[Getting Started](/getting-started/)** — Start using Specflow today
- **[Core Concepts](/core-concepts/)** — Understand contracts, agents, journeys

---

## Compiler Analogy Reminder

> **Pre-TypeScript:** JavaScript developers trusted their types. Bugs happened.
>
> **Post-TypeScript:** TypeScript enforces types. Violations blocked at compile time.

> **Pre-Specflow:** Developers trusted LLM outputs. Drift happened.
>
> **Post-Specflow:** Specflow enforces architecture. Violations blocked at test time.

**Same pattern. Different era.**

**The bottleneck shifted. Tools must shift too.**
