---
name: haiku-delegate
description: Use when performing search, file reading, command execution, or any mechanical/information-gathering task — delegates to haiku model to preserve parent context
---

# Haiku Delegate

Spawn haiku subagents for mechanical tasks instead of doing them inline. This preserves the parent agent's context window for decision-making, code writing, and synthesis.

**Core rule:** If a task is about *finding* or *reading* information rather than *deciding* or *writing*, it goes to haiku.

## Strict Delegation Rules

These tasks MUST be delegated to a haiku subagent. Do not perform them inline.

| Task | Example |
|------|---------|
| **File search** | Finding files by name, pattern, or content (glob, grep) |
| **Code reading for extraction** | "What does function X return?", "What args does Y accept?" |
| **Codebase exploration** | Understanding unfamiliar code areas, tracing call chains |
| **Command execution with verbose output** | Build logs, test output, dependency trees, git log |
| **Dependency/import tracing** | "What imports this module?", "What does this package export?" |
| **Listing and counting** | "How many test files?", "List all routes", "Show all env vars" |
| **Config/schema lookup** | Reading tsconfig, package.json, docker-compose for specific values |
| **Error log reading** | Parsing build failures, test output, runtime errors |

## Never Delegate

These tasks stay on the current model. Never send them to haiku.

- Writing or editing code
- Architectural decisions
- Code review and quality assessment
- Synthesizing findings into plans or summaries
- Choosing between approaches
- Commit message writing
- User-facing communication

## How to Dispatch

Use the Agent tool with `model: "haiku"`. Write focused prompts that tell haiku exactly what to find and how to report back.

**Prompt structure:**

```
[What to find/do]
[Where to look — specific paths or patterns]
[What to return — be explicit about output format]
[Constraints — what NOT to do]
```

**Example — finding a function:**

```
Find the definition of the `encodeHeartbeat` function.

Search in app/packages/shared/src/ and app/packages/ship/src/.

Return: file path, line number, function signature, and a 1-sentence description of what it does.

Do not read entire files — just locate the function and report back.
```

**Example — running a verbose command:**

```
Run `npm test` in the app/ directory.

Return: pass/fail summary, count of passing/failing tests, and the full error output for any failing tests only. Do not include passing test output.
```

**Example — exploring code:**

```
I need to understand how the data lane sends batches in the ship agent.

Start from app/packages/ship/src/shell/server.ts and trace the data sending flow.

Return: the chain of function calls from the server entry point to the actual HTTP request, with file paths and line numbers for each step.
```

## Rules for the Parent Agent

1. **Before grepping or globbing inline** — stop. Spawn haiku instead.
2. **Before reading a file just to find one thing** — stop. Spawn haiku instead.
3. **Before running a command whose output you'll scan and discard** — stop. Spawn haiku instead.
4. **Multiple searches needed?** Spawn multiple haiku agents in parallel (one per search).
5. **Trust haiku's results.** If results seem wrong, re-prompt haiku with better instructions. Do not re-do the search inline.
6. **Act on the summary.** Haiku returns information; you make decisions and write code based on it.

## Anti-Patterns

| Wrong | Right |
|-------|-------|
| Grep for a function, read 3 files, then decide | Spawn haiku to find and summarize, then decide |
| Run `npm test`, read 200 lines of output | Spawn haiku to run test and return pass/fail summary |
| Read 5 files to understand a code path | Spawn haiku to trace the path and report the chain |
| Do a quick glob "just to check" | Even quick searches add context — delegate them |
| Spawn haiku to write a fix | Haiku finds; you write |
