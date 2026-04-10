# Example: Refactor task (001-split-core-module)

```yaml
---
title: "Split core.py into core_tick.py and core_review.py"
workflow: feature-build
worksite: .
depends_on: []
deliverables:
  - path: src/simpleharness/core_tick.py
    description: "Pure tick planning logic extracted from core.py"
  - path: src/simpleharness/core_review.py
    description: "Pure review planning logic extracted from core.py"
refine_on_deps_complete: false
references:
  - src/simpleharness/core.py
  - CLAUDE.md
---

# Goal

Split `core.py` into two focused modules (`core_tick.py` for tick planning,
`core_review.py` for review planning) while preserving all behavior and
keeping `core.py` as a re-export facade so existing imports don't break.

## Success criteria

- [ ] `uv run pytest` exits 0 with no test changes
- [ ] `uv run ruff check .` clean
- [ ] `uv run ty check` clean
- [ ] `scripts/check_fp_purity.py` passes on all three core files
- [ ] No public API symbols removed from `core.py` (re-exports all names)
- [ ] `core_tick.py` contains only tick-related dataclasses and functions
- [ ] `core_review.py` contains only review-related dataclasses and functions

## Boundaries

- Do not change any function signatures or behavior
- Do not touch `shell.py` imports (the re-export facade handles this)
- Do not add new features or "improve" logic while splitting
- Do not modify tests (they must pass as-is)

## Autonomy

**Pre-authorized (decide and proceed):**
- Which functions belong in tick vs review (use judgment based on call graph)
- Import ordering within the new modules
- Whether to keep shared dataclasses in `core.py` or duplicate

**Must block (stop and write BLOCKED.md):**
- If splitting would require changing any public API signature
- If a circular import arises that can't be resolved without restructuring

## Notes

The FC/IS refactor is complete. All core.py functions are `@deal.pure` with
frozen dataclasses. The split should preserve this invariant.
```
