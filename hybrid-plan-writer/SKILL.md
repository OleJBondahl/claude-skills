---
name: hybrid-plan-writer
description: Use when writing PLAN.md for the hybrid Opus/local workflow. Triggers on feature-build-hybrid workflow or when plan output will be executed by local models in a loop.
---

# Hybrid Plan Writer

Write PLAN.md files that local models (Qwen3.5 9B, ~8K context) can execute step-by-step in a quality-gated loop.

## PLAN.md Format

Each step is a numbered `## Step N:` heading. The local builder reads ONE step at a time.

```markdown
## Step 1: <short title>

**Build:** `<file(s)>` -- <what to create or modify>

**Interface contract:**
- Input/output types and signatures
- Purity and side-effect constraints
- Dependencies allowed

**Acceptance criteria:**
- [ ] Specific test(s) that must pass (name the test function)
- [ ] Behavioral requirements
- [ ] No regressions in existing tests

**Quality wishlist:**
- Design pattern preferences (FP, immutability, etc.)
- Complexity targets (e.g., McCabe < 5)
- Code style requirements
```

## Key Principles

- **Define WHAT and WHY, not HOW.** Interfaces and tests, not implementation details.
- **One concern per step.** Each step should touch 1-2 files max.
- **Small enough for 8K context.** The builder reads one step + the relevant source files. Keep steps focused.
- **Tests must be specific.** Name the test function and file. Don't say "write tests" — say which test must pass.
- **Wishlist is aspirational.** The critic uses it to push quality. Items should be concrete and verifiable.

## Step Sizing

A step should be completable by a 9B model in one session (~20 tool calls):
- Add/modify one function or class
- Create one test file with 3-5 test functions
- Fix one bug with a clear test

Too large: "implement the entire retry module with tests"
Right size: "implement `retry_request()` matching the interface contract"

## End-to-End Section

After all steps, add:

```markdown
## End-to-End Verification

**Run:** `uv run pytest` (full test suite)
**Expected:** All tests pass, no regressions.
```
