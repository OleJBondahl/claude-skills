# SimpleHarness CLI Reference

Complete reference for agents using the `simpleharness` CLI. You should not
need to read source code — everything you need is here.

## Prerequisites

`simpleharness` must be on PATH (installed via `uv tool install`). The
toolbox (roles, workflows, subagents) lives inside the installed package.

## Commands

### `simpleharness init`

Create the `simpleharness/` folder layout in the worksite.

```
simpleharness init [--worksite PATH]
```

Creates: `simpleharness/tasks/`, `simpleharness/memory/`, `simpleharness/logs/`,
and a starter `WORKSITE.md`.

### `simpleharness new`

Scaffold a new task.

```
simpleharness new "<title>" [--workflow=<name>] [--worksite PATH]
```

- `title` — Short imperative title (e.g., "Add retry logic to API client")
- `--workflow` — `universal` (default), `feature-build`, `feature-build-local`,
  or `feature-build-hybrid`
- Creates `simpleharness/tasks/NNN-<slug>/` with template `TASK.md` and `STATE.md`
- **You must overwrite TASK.md** with the fully populated content after scaffolding

### `simpleharness watch`

Run the tick loop (primary mode).

```
simpleharness watch [--once] [--worksite PATH]
```

- `--once` — Do one tick then exit (useful for testing)
- Without `--once`, loops forever with idle sleep between ticks
- Each tick: discover tasks, pick the next active task, load its role, run a
  Claude Code session, update STATE.md

### `simpleharness status` / `simpleharness list`

Show all tasks with current state.

```
simpleharness status [--worksite PATH]
```

Output per task: `slug  status=  phase=  last=  next=  sessions=N/cap`

### `simpleharness show <slug>`

Show details for one task.

```
simpleharness show <slug> [--worksite PATH]
```

### `simpleharness unblock <slug>`

Reset a blocked task back to active.

```
simpleharness unblock <slug> [--worksite PATH]
```

- Accepts exact slug or unique substring
- Clears `blocked_reason` and resets `no_progress_ticks`

### `simpleharness doctor`

Sanity checks: Claude CLI on PATH, toolbox reachable, roles/workflows present,
permission mode valid, bash/uv available (for approver mode).

```
simpleharness doctor [--worksite PATH]
```

## Worksite Directory Structure

After `init` and creating tasks:

```
<worksite>/
  simpleharness/
    config.yaml              # optional per-worksite overrides
    memory/
      WORKSITE.md            # long-term notes every session reads
    tasks/
      001-my-task/
        TASK.md              # task brief (you author this)
        STATE.md             # harness-managed lifecycle state
        .session_prompt.md   # generated per session (ephemeral)
        00-kickoff.md        # phase files written by roles
        01-brainstorm.md
        02-plan.md
        03-develop.md
        FINAL.md
        CORRECTION.md        # user intervention (Ctrl+C during session)
      002-another-task/
        ...
    logs/
      001-my-task/
        00-project-leader.jsonl
        00-project-leader.log
```

## Workflows

### `universal`

Single-phase: **project-leader** runs every session and dynamically dispatches
other roles via `STATE.next_role`. Best for exploratory, unclear, or small tasks.

Roles: project-leader (always), then whatever project-leader dispatches.

### `feature-build`

Structured 5-phase pipeline:

1. **project-leader** — kickoff, reads TASK.md, writes `00-kickoff.md`
2. **brainstormer** — explores intent/requirements, writes `01-brainstorm.md`
3. **plan-writer** — produces `02-plan.md` from brief + brainstorm
4. **developer** — executes plan, writes `03-develop.md`, commits work
5. **project-leader** — wrap up, writes `FINAL.md`, sets `status=done`

Best for features, refactors, and bugfixes where a spec-driven approach fits.

### `feature-build-local`

Same as `feature-build` but uses a **local-worker** role (Ollama/Qwen3.5)
instead of developer for implementation. Good for cost-sensitive work or
benchmarking local models.

### `feature-build-hybrid`

Two-tier workflow: Opus handles planning, local Ollama models execute in a
**quality-gated loop**:

1. **project-leader** — kickoff
2. **brainstormer** — explores requirements
3. **plan-writer** — produces PLAN.md with numbered steps (uses
   `hybrid-plan-writer` skill). Each step has interface contracts, acceptance
   criteria, and a quality wishlist.
4. **loop** (local-builder → local-reviewer → local-critic):
   - **local-builder** implements one step from PLAN.md
   - **local-reviewer** runs tests, writes `REVIEW.md` with `verdict: pass|fail`
   - **local-critic** checks quality wishlist, writes `CRITIQUE.md` with
     `verdict: approved|suggestions`
   - On review fail → builder retries (up to `max_cycles`, default 5)
   - On critique suggestions → builder applies them (up to `max_critic_rounds`,
     default 2)
   - On step complete → advances to next step
   - After all steps → e2e test run → loop done
   - Steps that exhaust retries are flagged for project-leader review
5. **project-leader** — final review, handles flagged steps, wrap-up

Best for: feature work where you want local models doing the bulk of
implementation with automated quality gates. Requires Ollama running locally
with a model like `qwen3.5-nothink`.

**PLAN.md format for hybrid workflow:**
Each step must have:
- `## Step N` heading (harness counts these to set `total_steps`)
- Interface contracts and function signatures
- Acceptance criteria (specific test names/assertions)
- Quality wishlist (FP patterns, complexity targets)

## Available Roles

| Role | Provider | Purpose |
|------|----------|---------|
| `project-leader` | Claude | Orchestrates: kickoff, review, wrap-up, dispatches other roles |
| `brainstormer` | Claude | Explores requirements, asks questions, identifies risks |
| `plan-writer` | Claude | Produces implementation plans from briefs |
| `developer` | Claude | Implements code, runs tests, commits |
| `local-builder` | Ollama | Implements one plan step in the hybrid loop |
| `local-reviewer` | Ollama | Pass/fail review against acceptance criteria, writes REVIEW.md |
| `local-critic` | Ollama | Quality critique against wishlist, writes CRITIQUE.md |
| `local-worker` | Ollama | General local implementation (feature-build-local workflow) |
| `approver` | Claude | PreToolUse hook reviewer (approver permission mode only) |

## TASK.md Schema

### Frontmatter (YAML)

```yaml
---
title: "Human-readable imperative title"
workflow: feature-build          # or: universal
worksite: .                      # relative path to worksite root
depends_on: []                   # list of task slugs this depends on
deliverables:                    # files/artifacts this task produces
  - path: src/foo/bar.py
    description: "New module for bar logic"
refine_on_deps_complete: false   # true = upstream auto-refines this brief
references:                      # authoritative files for this work
  - docs/architecture.md
  - CLAUDE.md
---
```

**Field details:**

- `depends_on` — Task slugs (e.g., `001-split-core`). Task won't run until
  all deps have `status=done`.
- `deliverables` — Each has `path` (relative to worksite) and `description`.
  The harness verifies these exist before allowing `status=done`.
- `refine_on_deps_complete` — When `true`, upstream's project-leader appends
  concrete details when the dep finishes. When `false`, this task blocks for
  manual rebriefing.
- `references` — Files the agent should read for context. Be specific (paths,
  not "the codebase").

### Body Sections

```markdown
# Goal

<One paragraph: the state of the world when this task is done. Not steps.>

## Success criteria

- [ ] `uv run pytest` exits 0
- [ ] `uv run ruff check .` clean
- [ ] `src/foo/bar.py` exists with functions X, Y, Z
- [ ] <each criterion is objectively testable>

## Boundaries

- Do not modify <adjacent code/system>
- Do not change <existing behavior>

## Autonomy

**Pre-authorized (decide and proceed):**
- Internal naming, structure, test approach
- <decisions the agent can make>

**Must block (stop and write BLOCKED.md):**
- New runtime dependencies
- Public API changes
- Scope expansion beyond what's specified

## Handoff

<What downstream tasks consume from this task's output>

## Notes

<Optional context, background, or constraints>
```

## STATE.md Fields (Harness-Managed)

Agents generally don't edit STATE.md directly — the harness manages it. But
understanding the fields helps when checking task status:

| Field | Values | Notes |
|-------|--------|-------|
| `status` | `active`, `blocked`, `done`, `paused` | Lifecycle state |
| `phase` | `kickoff`, role name | Current workflow phase |
| `next_role` | role name or empty | Override for next session's role |
| `last_role` | role name | Who ran last |
| `total_sessions` | int | How many sessions have run |
| `session_cap` | int (default 20) | Max sessions before blocking |
| `blocked_reason` | string or empty | Why the task is blocked |
| `no_progress_ticks` | int | Consecutive ticks with no STATE.md changes |
| `loop_state` | dict or empty | Loop phase tracking (hybrid workflow only) |
| `loop_state.current_step` | int | 0-indexed step in PLAN.md |
| `loop_state.total_steps` | int | Total `## Step` headings in PLAN.md |
| `loop_state.cycle` | int | Review retry count for current step |
| `loop_state.critic_rounds` | int | Critic suggestion rounds for current step |
| `loop_state.inner_phase` | string | `building`, `reviewing`, `critiquing`, `e2e_testing`, `done` |
| `loop_state.flagged_steps` | list[int] | Steps that exhausted retries |

## Common Agent Workflows

### Create a task and start it

```bash
simpleharness init                                          # first time only
simpleharness new "Add retry logic to API client" --workflow=feature-build
# Edit the generated TASK.md with full content
simpleharness watch --once                                  # run one tick
```

### Check what's running

```bash
simpleharness status
```

### Unblock a stuck task

```bash
simpleharness unblock 001-add-retry     # exact slug or substring
```

### Verify setup

```bash
simpleharness doctor
```

## Workflow Choice Guide

| Task type | Workflow | Why |
|-----------|----------|-----|
| Feature / bugfix / refactor | `feature-build` | Structured phases: spec → plan → implement → review |
| Feature with local models | `feature-build-hybrid` | Opus plans, local models execute in quality loop |
| Cost-sensitive implementation | `feature-build-local` | All implementation on local Ollama |
| Research / spike / exploration | `universal` | Project-leader dispatches dynamically |
| Small / trivial fix | `universal` | Full pipeline would be overkill |
| Unclear scope | `universal` | Project-leader can pivot freely |

### When to use `feature-build-hybrid`

Choose hybrid when:
- The task is well-specifiable (clear acceptance criteria per step)
- You want to minimize Opus API cost (Opus only plans + reviews)
- You have Ollama running locally with a capable model
- The work is decomposable into independent implementation steps

Avoid hybrid when:
- The task requires creative architectural decisions throughout
- Steps have heavy inter-dependencies that need judgment
- The local model struggles with the project's language/framework
