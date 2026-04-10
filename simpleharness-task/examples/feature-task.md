# Example: Feature task (003-implement-streaming-logs)

```yaml
---
title: "Implement streaming log viewer"
workflow: feature-build
worksite: .
depends_on: [001-split-core-module, 002-research-streaming-logs]
deliverables:
  - path: src/simpleharness/log_viewer.py
    description: "Streaming log viewer module"
refine_on_deps_complete: true
references:
  - docs/research/002-streaming-logs-report.md
  - docs/architecture.md
  - CLAUDE.md
---

# Goal

Implement a production streaming log viewer using the approach recommended
in the research report from task 002. The viewer allows a user to tail
task session logs in real time from a browser while `simpleharness watch`
is running.

## Success criteria

- [ ] `uv run pytest` exits 0 (including new tests for log_viewer)
- [ ] `uv run ruff check .` clean
- [ ] `uv run ty check` clean
- [ ] `src/simpleharness/log_viewer.py` exists and follows FC/IS split
- [ ] `simpleharness watch` serves a log viewer endpoint on localhost
- [ ] Manual test: open browser to viewer URL while watch is running,
      see live log output within 2 seconds of a session writing to its
      phase file

## Boundaries

- Do not modify the core tick loop behavior
- Do not add frontend build tooling (plain HTML/JS only)
- Do not change the CLI interface beyond adding a `--viewer-port` flag
- Do not touch the approver subsystem

## Autonomy

**Pre-authorized (decide and proceed):**
- Port number default (suggest 8384)
- HTML/CSS styling choices
- Whether to use threading or asyncio for the server
- Internal module structure within log_viewer.py

**Must block (stop and write BLOCKED.md):**
- If the recommended approach from the research report is not viable in
  production (e.g., performance issue found during implementation)
- If a new runtime dependency is needed
- If the viewer requires changes to how sessions write phase files

## Handoff

This task's outputs feed: (none — terminal task in this batch)

## Notes

`refine_on_deps_complete: true` means when task 002 finishes, the
project-leader will append concrete details from the research report
(recommended approach, prototype location) to this TASK.md. The original
brief above stays intact.
```
