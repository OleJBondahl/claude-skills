---
name: loop-critic
description: Use when critiquing implementation quality against a plan step's wishlist inside the hybrid workflow loop. Triggers on local-critic role. Produces CRITIQUE.md with structured verdict.
---

# Loop Critic

Quality critique of a plan step's implementation against its quality wishlist.

## Workflow

1. **Read your step** from PLAN.md — the session prompt tells you which step number.
2. **Read the quality wishlist** for that step.
3. **Read the implementation code** for the step.
4. **Check each wishlist item** — does the code meet it?
5. **Write CRITIQUE.md** with the verdict (see format below).
6. **Update STATE.md** `phase` to `critiqued-step-N`.

## CRITIQUE.md Format

**Approved (meets wishlist):**
```
---
verdict: approved
---
Implementation meets quality wishlist.
- [x] Frozen dataclass for error type
- [x] @deal.pure decorated
- [x] Complexity under 5
```

**Suggestions (needs improvement):**
```
---
verdict: suggestions
---
Improvements needed:
- Use frozen dataclass for RetryExhausted instead of plain class (retry.py:15)
- Extract backoff calculation to pure function (retry.py:28-35)
- Complexity is 7, wishlist targets < 5 — simplify the retry loop
```

## Rules

- Do NOT modify source code. Critique only.
- The `verdict` field MUST be exactly `approved` or `suggestions`.
- Be specific: cite file:line numbers and suggest concrete changes.
- Focus ONLY on wishlist items. Do not invent standards beyond what the plan specifies.
- Keep suggestions actionable — tell the builder exactly what to change.
