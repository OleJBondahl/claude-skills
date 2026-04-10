# Example: Research task (002-research-streaming-logs)

```yaml
---
title: "Research streaming log viewer approach"
workflow: universal
worksite: .
depends_on: [001-split-core-module]
deliverables:
  - path: docs/research/002-streaming-logs-report.md
    description: "Decision report: approach comparison + recommendation"
  - path: scratch/002-streaming-logs
    description: "Prototype branch demonstrating recommended approach"
refine_on_deps_complete: false
references:
  - docs/architecture.md
  - src/simpleharness/shell.py
---

# Goal

Evaluate three approaches for a real-time streaming log viewer (WebSocket,
SSE, file-tail polling) and produce a written recommendation with a working
prototype of the chosen approach on a scratch branch.

## Success criteria

- [ ] `docs/research/002-streaming-logs-report.md` exists with: problem
      summary, three approaches evaluated against [latency, complexity,
      browser support], a clear recommendation with rationale
- [ ] Branch `scratch/002-streaming-logs` exists with a prototype that
      demonstrates the recommended approach against a mock log stream
- [ ] `tests/research/test_streaming_smoke.py` exists and passes on the
      scratch branch
- [ ] If NO approach meets all three criteria, the report states that
      explicitly and the task blocks for user decision

## Boundaries

- Do not modify production code on `main` — scratch branch only
- Do not add new runtime dependencies without flagging in the report
- Do not implement a full UI — prototype is backend + minimal HTML

## Autonomy

**Pre-authorized (decide and proceed):**
- Weighting of evaluation criteria (prefer simplicity if close)
- Choice of test framework within the project's existing stack
- Naming of prototype files, functions, branches

**Must block (stop and write BLOCKED.md):**
- If all three approaches fail the evaluation criteria
- If a new runtime dependency is required for the recommended approach
- If the prototype reveals a fundamental architecture incompatibility

## Handoff

This task's outputs feed: 003-implement-streaming-logs
Downstream needs: the report's recommendation + the prototype branch as
a starting point for production implementation.

## Notes

The `depends_on` on 001-split-core-module ensures the research starts with
the cleaner module structure. `refine_on_deps_complete: false` means this
task will block after 001 finishes — the user re-briefs with any new context
from the split before research begins.
```
