# Example: Batch authoring session

This shows the conversation flow when authoring the three related tasks
from the other examples as a batch.

## User opens with

> "I want to split core.py into smaller modules, then research a streaming
> log viewer, then implement it. The research needs the split done first,
> and implementation needs both."

## Skill decomposes into tasks

The skill identifies three tasks and proposes the dep graph:

```
001-split-core-module (no deps)
  |
  v
002-research-streaming-logs (depends on 001)
  |
  v
003-implement-streaming-logs (depends on 001, 002)
```

## Skill walks through each in producer->consumer order

1. **001-split-core-module** — authored first (no deps, no deliverables
   consumed by 002/003 beyond the code itself)
2. **002-research-streaming-logs** — the skill asks about deliverables
   because 003 depends on it. User specifies the report path and prototype
   branch. Skill prompts for `refine_on_deps_complete` on 003 — user
   chooses `true` (upstream may auto-refine 003's brief).
3. **003-implement-streaming-logs** — references include the report from
   002. Success criteria are initially broad because the recommendation
   isn't known yet. The skill notes this is acceptable because
   `refine_on_deps_complete=true` means 002's project-leader will append
   concrete details.

## Cross-task review

The skill verifies:
- `docs/research/002-streaming-logs-report.md` is declared in 002's
  deliverables AND referenced in 003's references. No dangling ref.
- `scratch/002-streaming-logs` is declared in 002's deliverables.
- No circular dependencies (001->002->003 is a DAG).
- 003's `refine_on_deps_complete=true` means its broad criteria are OK.

## Scaffold

The skill runs:
```bash
simpleharness new "Split core.py into core_tick.py and core_review.py" --workflow=feature-build
simpleharness new "Research streaming log viewer approach" --workflow=universal
simpleharness new "Implement streaming log viewer" --workflow=feature-build
```

Then overwrites each TASK.md with the populated content.
