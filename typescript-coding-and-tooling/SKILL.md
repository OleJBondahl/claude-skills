---
name: typescript-coding-and-tooling
description: Use when writing TypeScript code, setting up a new TS repo or workspace, configuring tsconfig/eslint/vitest, or structuring code into core/orchestration/shell layers. Covers npm workspaces, strict tsconfig, eslint-plugin-functional, neverthrow, dependency-cruiser, vitest, ESM .js imports, and Fastify handler style.
---

# TypeScript Coding and Tooling

## Overview

Opinionated baseline for TypeScript projects in this user's ecosystem. Applies the global Code Style Philosophy (functional core / imperative shell) to TypeScript concretely, with a three-layer split enforced by lint and architecture rules. Matches the conventions already in place in `Github_zen/fastify-test`.

## Mandatory Toolchain

| Concern | Tool | Notes |
|---|---|---|
| Package manager | `npm` (with workspaces) | Default. `pnpm` acceptable for large monorepos. Not `yarn`, not `bun`. |
| Language | TypeScript, ESM (`"type": "module"`) | `.js` extensions required on relative imports (ESM resolution). |
| Build | `tsc` | No bundlers unless shipping to a browser. |
| Runtime (scripts/tests) | `tsx` | For running TS directly without a build step. |
| Linter | `eslint` + `typescript-eslint` strict-type-checked | With `eslint-plugin-functional`, `eslint-plugin-sonarjs`, `eslint-plugin-jsdoc`. |
| Formatter | `prettier` | 100 col, trailing commas, double quotes. |
| Tests | `vitest` | Globals enabled. Coverage thresholds enforced. |
| Result type | `neverthrow` | `Result<T, E>`. No custom discriminated unions unless there's a strong reason. |
| Architecture boundary | `dependency-cruiser` | Enforces shell → orchestration → core, one direction only. |
| Dead code | `knip` | Run via `npm run quality:deadcode`. |
| Duplication | `jscpd` | Run via `npm run quality:duplication`. |

## Three-Layer Layout

```
packages/<pkg>/
  src/
    core/                 # pure — no I/O, no throw, no try/catch, no let, no loops
    orchestration/        # pure decisions — no throw/try, local-let allowed
    shell/                # I/O, Fastify routes, DB, fs, network. try/catch allowed
    index.ts
  tests/
    core/*.test.ts                # unit tests (vitest)
    orchestration/*.test.ts
    shell/*.integration.test.ts   # integration tests
```

Dependency direction: `shell → orchestration → core`. Never the reverse. Enforced by `dependency-cruiser`.

## tsconfig.json (strict baseline)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*.ts"]
}
```

No `any`. No `as` casts (use type guards). No non-null `!` (narrow instead).

## ESLint Layer Rules

`eslint-plugin-functional` split by directory:

- **`src/core/**`** — `no-throw`, `no-try`, `no-loop`, `no-let`, `immutable-data`. Pure input → output only.
- **`src/orchestration/**`** — `no-throw`, `no-try`. Local `let` allowed inside functions; no mutation of parameters.
- **`src/shell/**`** — side effects, `try/catch`, Fastify plugins, DB clients.
- **All layers** — mandatory JSDoc on every exported function with an explicit return type.
- **Tests** — all FP rules relaxed.

## Core Pattern

```ts
// src/core/discount.ts
import { ok, err, type Result } from "neverthrow";

/**
 * Applies a percentage discount to a price in minor units.
 * @param price   price in minor units (e.g. cents)
 * @param percent integer percent in [0, 100]
 */
export function applyDiscount(
  price: number,
  percent: number,
): Result<number, string> {
  if (percent < 0 || percent > 100) {
    return err(`percent out of range: ${percent}`);
  }
  return ok(price - Math.floor((price * percent) / 100));
}
```

## Shell Pattern (Fastify handler)

```ts
// src/shell/routes/discount.ts
import type { FastifyInstance } from "fastify";
import { applyDiscount } from "../../core/discount.js"; // note .js

/** Registers the POST /discount route. */
export async function registerDiscountRoute(app: FastifyInstance): Promise<void> {
  app.post<{ Body: { price: number; percent: number } }>("/discount", async (req, reply) => {
    const result = applyDiscount(req.body.price, req.body.percent);
    if (result.isErr()) {
      return reply.code(400).send({ error: result.error });
    }
    return reply.send({ price: result.value });
  });
}
```

Handlers are thin: decode → call core → act on `Result`. Business logic does not live in the handler.

## package.json Scripts (baseline)

```json
{
  "type": "module",
  "scripts": {
    "build": "tsc -p .",
    "dev": "tsx watch src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src tests",
    "format": "prettier -w src tests",
    "quality:architecture": "depcruise src",
    "quality:deadcode": "knip",
    "quality:duplication": "jscpd src",
    "quality": "npm run lint && npm run quality:architecture && npm run quality:deadcode && npm run quality:duplication"
  }
}
```

## Import Style

ESM requires explicit extensions on relative imports:

```ts
import { applyDiscount } from "../core/discount.js"; // .js, even though the source is .ts
import { Foo } from "./types.js";
```

No path aliases unless a workspace needs them. Prefer relative imports within a package and workspace imports between packages.

## Common Commands

```bash
npm install                 # install workspace deps
npm run dev                 # tsx watch
npm test                    # vitest run
npm run lint                # eslint
npm run format              # prettier -w
npm run quality             # lint + arch + knip + jscpd
npm run build               # tsc
```

Never chain with `&&` / `;` at the Bash tool level — a hook blocks it. (Inside `package.json` scripts is fine.)

## Common Mistakes

| Mistake | Fix |
|---|---|
| `throw new Error(...)` in `core/` or `orchestration/` | Return `err("...")`. |
| `let x = ...` in `core/` | Use `const`; build new values. |
| `for (...)` or `while (...)` in `core/` | Use `.map`, `.filter`, `.reduce`, recursion. |
| `import { foo } from "./bar"` (no `.js`) | ESM requires `import { foo } from "./bar.js"`. |
| `as unknown as Foo` | Write a type guard: `function isFoo(x: unknown): x is Foo`. |
| `x!` non-null assertion | Narrow with a check and handle the null case. |
| Business logic inside a Fastify handler | Move to `core/` or `orchestration/`, call from the handler. |
| Mocking a core function in a test | Core is pure — call it directly with inputs. |
| Adding `any` to silence an error | Fix the type. `any` is banned by lint. |
| `zod` for internal types | Use plain TS types + hand-written validators at the shell boundary. |

## Testing Notes

- **Core and orchestration tests** are unit tests. Import the function, call it, assert on the returned `Result`. No mocks.
- **Shell tests** are `*.integration.test.ts`. Use `app.inject()` for Fastify routes (no real HTTP server). Use real test DBs where feasible.
- **Coverage**: target 80% lines / 70% branches, enforced in `vitest.config.ts`.
- **Property tests** (`fast-check`) for core functions with algebraic invariants.
- **Factories** (`fishery`) for test fixtures instead of inline literals.

## Red Flags

- `throw` or `try` outside `src/shell/`
- `let` or `for` inside `src/core/`
- A relative import without a `.js` extension
- `any`, `as`, or `!` anywhere in `src/`
- A Fastify handler longer than ~30 lines
- A core function that takes a `FastifyRequest`, a `Pool`, or a `Logger`
- `dependency-cruiser` warnings about layer violations — fix the import, don't suppress the warning
