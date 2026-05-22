---
name: contextual-code-review
description: Use when reviewing code changes, pull requests, or evaluating code quality - especially when the codebase has established conventions, frameworks, or architectural patterns that new code must conform to
---

# Contextual Code Review

## Overview

Code review is **judging whether implementation faithfully serves design intent**. Not pattern-matching against a smell catalog.

Three questions drive every review:
1. **Is the design reasonable?** — Does the code's structure match its purpose?
2. **Is the implementation correct?** — Does the code do what its design promises?
3. **Is the expression clear?** — Does the code communicate intent to the next reader?

## When to Use

- Reviewing a PR or diff
- Evaluating code quality of a module
- Reviewing your own code before committing

## The Three-Step Method

### Step 1 — Understand Context (do NOT judge yet)

1. Read the project's conventions (config files, existing patterns, base classes)
2. Identify framework/library idioms in use
3. Understand **what the code is supposed to do** (PR description, commit message, linked issue)
4. Identify the **business domain** — it determines which problems are critical vs. tolerable

### Step 2 — Evaluate Design

Apply core design principles to judge whether the code's **structure** is appropriate for its **purpose**. Do not mechanically check metrics (line count, parameter count) — reason about whether the design serves its intent.

#### Principle 1: Single Responsibility

**Question:** Can you describe this unit's purpose in one sentence without "and"?

A function/module/class should have exactly one reason to change. Violation is not measured by size — a 200-line function that only drives a PWM breathing effect has one responsibility. A 20-line function that validates input, calculates price, and sends a notification has three.

**How to detect:** Name each distinct "section" in the function body. If they serve different stakeholders or change for different reasons, the function mixes responsibilities.

**Context matters:** In embedded code, a hardware init sequence that configures 10 registers is one responsibility ("initialize device"). In a web service, a handler that validates, processes, persists, and notifies is four.

#### Principle 2: Consistent Contracts

**Question:** Does every caller get what they expect, in every case?

- Error signaling must be uniform: if the convention is exceptions, don't return null; if it's error codes, don't mix in exceptions
- Return types must be stable: don't return `{data}` on success and `{error}` on failure from the same function
- Side effects must be declared: a function named `get*` must not mutate state

**How to detect:** Check all exit paths of a function — do they all conform to the same contract? Check if the contract matches the codebase convention (identified in Step 1).

#### Principle 3: Explicit Dependencies

**Question:** Can you understand this code's behavior by reading it alone, without hidden knowledge?

- No hardcoded config (URLs, credentials, thresholds) — these are hidden dependencies on deployment environment
- No magic values — literals whose meaning requires domain knowledge the reader may not have
- No implicit coupling — don't silently depend on global state, call order, or timing

**How to detect:** For each literal, ask: "would a new team member understand why this value?" For each external call, ask: "is this dependency visible in the function signature or class constructor?"

#### Principle 4: Appropriate Abstraction

**Question:** Does the abstraction level match the actual need?

- **Under-abstraction:** Raw dicts/maps passed through 3+ functions when a named type would clarify intent. Copy-pasted code blocks that differ only in values (should be parameterized).
- **Over-abstraction:** Interfaces with one implementation and no concrete plan for more. Patterns applied where a simple function call suffices. (YAGNI)

**How to detect:** For duplication — are there 2+ code blocks >5 lines that are 80%+ identical? For premature abstraction — is there exactly one implementation with no evidence of planned variation?

#### Principle 5: Fail Visibly

**Question:** When something goes wrong, will anyone know?

- Critical operations must not swallow errors silently (`catch: pass`, empty `except`, logging without re-raise)
- Error context must be preserved: log the entity ID, the input values, the operation that failed
- Observability must be consistent across all code paths — not detailed logging on path A and silence on path B

**How to detect:** Trace each error path: does it produce enough information to diagnose the problem without reproducing it?

### Step 3 — Verify Implementation

Once the design is accepted, check whether the implementation is **correct and safe**. These are concrete, verifiable properties:

| Check | What to Look For |
|-------|-----------------|
| **Logic correctness** | Off-by-one errors, boundary conditions, operator precedence, integer overflow/truncation |
| **Concurrency safety** | Shared mutable state without synchronization; lock/unlock asymmetry; TOCTOU (check-then-act without atomicity) |
| **Resource safety** | Unbounded growth (cache/queue without TTL/eviction); unclosed handles; leaked connections |
| **Data integrity** | Float arithmetic on currency; accumulation applied to running total instead of per-item; string concatenation for structured output (CSV/SQL/XML) without escaping |
| **Atomicity** | Multi-step writes without transaction boundary — ask: "if step N fails, are steps 1..N-1 rolled back?" |
| **Performance traps** | Await/query inside a loop (N+1 pattern); blocking I/O on hot path; full table scan where index exists |
| **Dead paths** | Unused imports, uncalled functions, unreachable branches, TODO on live code paths, features behind permanent flags |

### Step 4 — Check Code Clarity

Design can be sound and implementation correct, but if the code doesn't clearly communicate intent, it becomes expensive to maintain. These checks catch issues that linters cannot — they require understanding meaning.

**Leave to linters/formatters:** indentation, bracket style, import order, trailing whitespace. Do NOT flag these in review.

#### Naming

**Test:** Can a reader understand the variable/function/class purpose without reading its implementation?

- Names should reveal intent: `remainingRetries` not `n`; `isEligibleForDiscount` not `flag`; `fetchActiveOrders` not `getData`
- Names should match scope: single-letter vars acceptable in 3-line loops, unacceptable as function parameters or class fields
- Names should not lie: if a function is called `validate` it must not also transform; if a variable is called `count` it must be a number
- Abbreviations: acceptable if universal in the domain (`ctx`, `req`, `tx` in their respective contexts), unacceptable if ambiguous (`proc`, `mgr`, `tmp` as public API)

#### Abstraction Level Consistency

**Test:** Does every line in this function operate at the same level of detail?

A function should not mix high-level business logic with low-level mechanical operations. If `processOrder()` contains both `validateBusinessRules(order)` and `bytes.concat(header, payload.slice(4, offset + len))`, the reader is forced to context-switch between two mental models.

**How to detect:** Read the function body — if some lines describe "what" (business intent) and others describe "how" (byte manipulation, string parsing, register writes), the abstraction levels are mixed. Extract the low-level parts into named helpers.

**Exception:** In embedded/systems code, hardware interaction often requires mixing levels. Judge by whether the mixing is essential (register sequence that must be atomic) or accidental (business logic tangled with byte formatting).

#### Comments

**Test:** Does every comment explain *why*, not *what*?

- **Unnecessary:** `i++ // increment i` — the code already says this
- **Necessary:** `// Retry 3 times because the sensor needs warm-up after cold boot` — the code cannot express this reasoning
- **Dangerous:** Comments that contradict the code, or describe what the code *used to do*
- **TODO/FIXME on live paths:** If the TODO is on a code path that runs in production, it's not a comment — it's an unfinished feature masquerading as documentation. Flag it

#### Error Handling Separation

**Test:** Can you read the happy path without wading through error handling?

- Error handling code should not obscure the main logic flow
- Prefer early returns / guard clauses over deeply nested if-else error checking
- Error branches should be clearly distinct from normal branches — a reader should never wonder "is this the success path or the error path?"

## Context-Aware Judgment Rules

**Rule 1: Design intent determines correctness.** The same code pattern is correct or wrong depending on what it's trying to achieve. Judge against purpose, not against a universal checklist.

**Rule 2: Codebase convention is law.** If the team throws exceptions for errors, returning null is a bug. If the team uses raw SQL, flagging "should use ORM" is a preference, not a defect.

**Rule 3: Business domain calibrates severity.** `float` for money is critical in fintech, acceptable in a game score. `catch: pass` is critical in payment processing, tolerable in best-effort telemetry.

**Rule 4: Evidence, not labels.** "Violates SRP" is useless. "This function validates input (lines 10-20), calculates price (21-50), persists to DB (51-60), and sends email (61-70) — four distinct responsibilities that change independently" is actionable.

**Rule 5: Diff-focused.** In PR review, focus on what the diff introduces. Pre-existing issues are out of scope (note separately if critical).

## Output Format

Structure by severity, not by topic:

```
## Must Fix (blocks merge)
1. [Design/Implementation] file:line — What's wrong + evidence + fix suggestion

## Should Fix (merge OK, fix before next PR)
1. [Design/Implementation] file:line — What's wrong + evidence + fix suggestion

## Consider (optional improvement)
1. [Design/Implementation] file:line — What's wrong + evidence + fix suggestion
```

**Rules:**
- Tag each finding as **Design** (structural problem, needs rethinking), **Implementation** (code-level bug/issue, needs fixing), or **Clarity** (readability/maintainability issue, needs cleanup)
- Max 10 Must Fix. More than that means suggest redesign, not enumerate issues
- Each finding MUST have: file:line, concrete evidence from the code, specific fix suggestion
- Never say "improve error handling" — specify which error, where, and how

## Common Review Mistakes

| Mistake | Root Cause |
|---------|-----------|
| Flagging line count / parameter count as inherent defects | Confusing proxy metric with actual design problem |
| "Violates SRP" without naming the responsibilities | Using principle label without doing the analysis |
| Flagging pattern that's standard in this codebase | Skipped Step 1 (context gathering) |
| 50 issues with equal weight | Not distinguishing design defects from implementation issues |
| Checking code against universal rules without context | Applying Rule 1 backwards — checking rules before understanding intent |
