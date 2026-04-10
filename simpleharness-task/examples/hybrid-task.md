# Example: Hybrid workflow task (004-add-backoff-retry)

```yaml
---
title: "Add exponential backoff retry to API client"
workflow: feature-build-hybrid
worksite: .
depends_on: []
deliverables:
  - path: src/api/retry.py
    description: "Retry module with exponential backoff"
  - path: tests/test_retry.py
    description: "Tests for retry logic"
refine_on_deps_complete: false
references:
  - src/api/client.py
  - CLAUDE.md
---

# Goal

The API client retries transient errors (429, 500, 502, 503) with exponential
backoff and jitter. After max retries, it raises a structured error with the
full retry history for debugging.

## Success criteria

- [ ] `uv run pytest tests/test_retry.py` exits 0
- [ ] `uv run ruff check .` clean
- [ ] `uv run ty check` clean
- [ ] `src/api/retry.py` exists as a pure-core module (no I/O)
- [ ] Retries 429/500/502/503 with exponential backoff + jitter
- [ ] Configurable max_retries (default 3) and base_delay (default 1.0s)
- [ ] Raises RetryExhausted with full attempt history after max retries
- [ ] Does not retry 4xx errors other than 429

## Boundaries

- Do not modify the existing `client.py` interface
- Do not add new runtime dependencies
- Do not change test infrastructure

## Autonomy

**Pre-authorized (decide and proceed):**
- Jitter algorithm (full vs decorrelated)
- Internal data structures for retry history
- Test approach (mocking HTTP responses)
- Whether to use frozen dataclass or NamedTuple for attempt records

**Must block (stop and write BLOCKED.md):**
- If retry logic needs to be async (changes client interface)
- If a new dependency is needed for HTTP mocking in tests
- If the existing client.py doesn't have clear request/response boundaries

## Handoff

None (terminal task).

## Notes

The hybrid workflow will have Opus produce a step-by-step PLAN.md, then local
models implement each step in the builder/reviewer/critic loop. Keep the
acceptance criteria specific enough that the local reviewer can verify each
step independently.
```
