---
name: loop-reviewer
description: Use when reviewing a plan step for pass/fail acceptance inside the hybrid workflow loop. Triggers on local-reviewer role. Produces REVIEW.md with structured verdict.
---

# Loop Reviewer

Binary pass/fail verification of a plan step against its acceptance criteria.

## Workflow

1. **Read your step** from PLAN.md — the session prompt tells you which step number.
2. **Read the acceptance criteria** for that step.
3. **Run the step's tests**: `uv run pytest <test_file> -v` (or project equivalent).
4. **Check each criterion** — does the implementation satisfy it?
5. **Write REVIEW.md** with the verdict (see format below).
6. **Update STATE.md** `phase` to `reviewed-step-N`.

## REVIEW.md Format

**Pass:**
```
---
verdict: pass
---
All acceptance criteria met.
- [x] test_exact_retry_count passes
- [x] No regressions in existing tests
```

**Fail:**
```
---
verdict: fail
---
Failing criteria:
- [ ] test_exact_retry_count — AssertionError: expected 3, got 2
- [x] No regressions in existing tests
```

## Rules

- Do NOT modify source code. Review only.
- Do NOT write vague verdicts. Cite specific test output or file:line references.
- The `verdict` field MUST be exactly `pass` or `fail` — no other values.
- If tests can't run (missing deps, syntax error), verdict is `fail` with the error message.
