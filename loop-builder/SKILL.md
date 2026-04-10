---
name: loop-builder
description: Use when implementing a plan step inside the hybrid workflow loop. Triggers on session prompt containing "Step N of M" or "PLAN.md". Covers read-step, implement, test, lint, commit cycle.
---

# Loop Builder

Focused workflow for implementing one plan step at a time inside the hybrid Opus/local loop.

## Workflow

1. **Read your step** from PLAN.md — the session prompt tells you which step number.
2. **Read the interface contract** — inputs, outputs, types, constraints.
3. **Write a failing test** matching the acceptance criteria (if tests don't already exist).
4. **Implement** the minimal code to satisfy the contract and pass the tests.
5. **Run the step's tests** — not the full suite, just the relevant ones.
6. **Run linter** on changed files: `uv run ruff check <file>` (or project equivalent).
7. **Commit** with a message referencing the step: `feat: implement step N — <title>`.
8. **Update STATE.md** `phase` field to describe what you completed.

## Rules

- Read only the lines you need (`offset`/`limit`). Never read whole files.
- One shell command per Bash call. Never chain with `&&` or `;`.
- Do NOT read or implement other steps — only your assigned step.
- Do NOT refactor code outside your step's scope.

## If stuck

- Set `STATE.status: blocked` and `STATE.blocked_reason: <why>`.
- If the task needs stronger reasoning, set `STATE.next_role: developer` to escalate.

## If retrying after review failure

The session prompt says "Retry N". Read REVIEW.md for what failed. Fix only the failing criteria.

## If applying critic suggestions

The session prompt says "Critic round N". Read CRITIQUE.md for suggestions. Apply them to your step's code.
