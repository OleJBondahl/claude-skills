---
name: python-coding-and-tooling
description: Use when writing Python code, setting up a new Python repo, configuring pyproject.toml, choosing linters/type checkers/test runners, writing docstrings, or structuring modules into functional core and imperative shell. Covers uv, ruff, ty, pytest, deal, vulture, pyproject.toml layout, frozen dataclasses, and docstring style.
---

# Python Coding and Tooling

## Overview

Opinionated baseline for Python projects in this user's ecosystem. Applies the global Code Style Philosophy (functional core / imperative shell) to Python concretely: which tools, which file layout, which decorators, which commands, which docstring style. Companion skill: `reviewing-ai-generated-python` for review/audit work.

## Mandatory Toolchain

| Concern | Tool | Notes |
|---|---|---|
| Package manager | `uv` | Never `pip`, never `poetry`, never `pipenv`. |
| Runner | `uv run` | Never bare `python`. Never `python -c` (hook-blocked). |
| Python version | `3.14` | Set `requires-python = ">=3.14"`. ruff `target-version = "py313"` until ruff ≥ 0.15.13 fixes the py314 `except (X, Y):` format bug; `ty` infers the runtime version from `requires-python` regardless. |
| Linter + formatter | `ruff` | One tool does both. No `black`, no `flake8`, no `isort`. |
| Type checker | `ty` | Astral's type checker. Not `mypy`, not `pyright`. ty honours `# ty: ignore[<rule>]` (its native syntax), NOT legacy `# type: ignore[<rule>]`. |
| Tests | `pytest` | With `pytest-cov` for coverage. |
| Property tests | `hypothesis` | When the input space is large enough that example tests can't cover it. |
| Mutation tests | `mutmut` | Periodic kill-rate runs on hot modules. |
| Dead-code | `vulture` | Advisory pre-commit hook (`--min-confidence 80`). Catches unused module-level functions/classes that ruff F doesn't. |
| Architecture contracts | `import-linter` | When the project has > 1 layer (e.g. core/shell). |
| Purity enforcement | `deal` + `fp-purity-gate` AST script | `@deal.pure` on every function in a core module. Don't expand `deal` outside core/ — see Forbidden Toolchain below. |

## Forbidden Toolchain

These tools are **redundant with ruff or `ty`** and must NOT be added. Adding them creates duplicate config, slower CI, and conflicting findings.

| Tool | Why forbidden | What replaces it |
|---|---|---|
| `bandit` | ruff's `S` (flake8-bandit) rules cover the same checks, faster. | `select = [..., "S"]` in pyproject. |
| `radon` (cc, mi) | ruff's `C90` + `PLR0911/0912/0913/0915` cover complexity. | `select = [..., "C90", "PLR"]` with thresholds in `[tool.ruff.lint.mccabe]` / `[tool.ruff.lint.pylint]`. |
| `docstr-coverage` | ruff's `D100-D107` flag missing docstrings. The aggregate % is not actionable; per-site flags are. | `select = [..., "D"]` with `convention = "google"`. |
| `interrogate` | Same as `docstr-coverage`. Also has `cairosvg` install issues on Windows. | `select = [..., "D"]`. |
| `darglint` / `darglint2` | Forces `:param:`/`:returns:` agreement on every public function — *opposite* of the docstring style this skill prescribes (see Docstrings below). Upstream is unmaintained. | None. The companion `reviewing-ai-generated-python` skill calls this out as smell #4. |
| `pydoclint` | Same problem as darglint: agreement-checking pushes toward inflated docstrings. ruff is implementing DOC-prefixed rules in preview; do not add pydoclint as a stopgap. | None. Wait for ruff DOC to stabilize. |
| `pylint` | ruff's `PLR`/`PLE`/`PLW` cover the worthwhile checks, 100x faster. | `select = [..., "PLR"]`. |
| `mypy` / `pyright` | Project standardises on `ty`. Mixed checkers fight over the same `# ignore[...]` syntax (ty uses `# ty: ignore`, mypy uses `# type: ignore`). | `ty`. |
| `black` / `autopep8` / `yapf` | `ruff format` is the formatter. | `ruff format`. |
| `isort` | ruff's `I` rules sort imports, with formatter-compatible style. | `select = [..., "I"]`. |
| `flake8` (and plugins) | ruff implements ~all flake8 plugins natively. | `ruff`. |

If a project already has any of these, **drop them** as a separate cleanup commit. Don't run them in parallel with ruff.

## Docstrings

**Default: don't write a docstring.** Identifier names + type hints already say WHAT the code does. Reserve docstrings for non-obvious WHY: invariants, edge cases, units, side-effect ordering, hidden constraints.

When you do write one, it's a single line:

```python
# ❌ BAD: paraphrases the signature, AI-bloat smell
def add_terminal(self, tm_id: str, *, poles: int = 1) -> Terminal:
    """Add a terminal to the circuit.

    Args:
        tm_id: The terminal identifier.
        poles: The number of poles. Defaults to 1.

    Returns:
        The newly created terminal.

    Raises:
        ValueError: If tm_id is empty.
    """
    ...

# ✅ GOOD: silent if the WHY is obvious from the name + types
def add_terminal(self, tm_id: str, *, poles: int = 1) -> Terminal:
    ...

# ✅ GOOD: one line, only when there's a non-obvious WHY
def add_terminal(self, tm_id: str, *, poles: int = 1) -> Terminal:
    """Auto-connects to the previous component in the chain unless `connect_from_previous=False`."""
    ...
```

Forbidden in docstrings:
- Restating the signature (`Args:` block listing every param with type + paraphrase).
- `Returns:` blocks that paraphrase the return type.
- `Raises:` blocks for exceptions the type system or invariant rules out.
- Multi-paragraph "Examples" — write a real test instead.
- Section banners (`# === Setup ===`) in short files.

Allowed:
- One-line WHY for non-obvious behavior.
- `Raises:` only when the exception is part of the public contract AND not obvious from the function name (e.g. a domain error from a builder method that otherwise looks pure).
- A `# unit: mm` style comment for ambiguous numeric returns.

The companion skill `reviewing-ai-generated-python` (smell #4) treats inflated docstrings as a deletion candidate. If you see one, delete it.

### Why no docstring-agreement enforcement?

Tools like `darglint`, `darglint2`, `pydoclint` enforce that every signature parameter appears in a `:param:` block in the docstring. That assumes the docstring SHOULD restate the signature. This skill's stance is the opposite: docstrings should NOT restate the signature. Adding agreement-checking tools is therefore a category mistake — they enforce the AI-bloat pattern, not avoid it.

## Repo Layout

```
<repo>/
  pyproject.toml
  src/<package>/
    core.py           # pure — no I/O, no globals, no clock reads
    shell.py          # impure — subprocess, fs, network, datetime.now()
    __init__.py
  tests/
    test_core.py      # unit tests — deterministic, no mocks
    test_shell.py     # integration tests — real fs/process, no mocks of core
  claude-tools/       # ad-hoc scripts (gitignored)
```

One-way dependency: `shell → core`, **never** `core → shell`. If `core.py` needs to import `os`, `pathlib`, `subprocess`, `requests`, or anything from `shell.py`, you've put the logic in the wrong file.

## pyproject.toml Template

```toml
[project]
name = "<name>"
requires-python = ">=3.14"
dependencies = ["deal"]

[dependency-groups]
dev = ["pytest", "pytest-cov", "ruff", "ty", "vulture"]

[tool.ruff]
line-length = 100
target-version = "py313"  # bump to py314 once ruff fixes the except-tuple bug

[tool.ruff.lint]
select = [
    "E", "W", "F", "I",          # pycodestyle + pyflakes + isort
    "B", "UP", "SIM", "RUF",     # bugbear, pyupgrade, simplify, ruff-specific
    "N", "D",                    # naming + docstrings
    "ARG", "PLR", "PT", "RET",   # unused args, pylint-refactor, pytest, return
    "C90", "PERF", "PIE", "TC",  # complexity, perf, pie, type-checking
    "T20", "LOG", "G", "Q",      # print, logging, logging-format, quotes
    "BLE", "RSE", "TRY",         # blind-except, raise, tryceratops
    "S", "DTZ", "PTH", "ERA",    # security, datetimez, pathlib, eradicate
    "FBT", "EM", "TID", "ANN",   # bool-trap, errmsg, tidy-imports, annotations
    "ICN", "ISC",                # import-conventions, implicit-str-concat
]
ignore = ["TRY003"]              # domain exceptions cover this; per-message subclasses are over-eng

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["ANN", "PLR2004", "S101"]  # tests don't need return types, magic numbers ok, asserts are pytest's mechanism
"examples/**" = ["ANN"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers"
```

The `select` list above is the bar for an established project. For a fresh repo, it's fine to start with a smaller set and grow — but always use `select`, never `extend-select` only, so the rule list is explicit.

## Core Module Rules

Every function in a `core*.py` file:

1. **Is decorated `@deal.pure`** — enforced by `fp-purity-gate` AST check in pre-commit.
2. **Takes all inputs as parameters**, including `now: datetime` and `env: Mapping[str, str]`. Never reads `datetime.now()`, `os.environ`, `random.random()`, or any file.
3. **Returns a value** — no mutation of arguments, no global state writes.
4. **Operates on `@dataclass(frozen=True)` types**, not dicts. Rebuild with `dataclasses.replace(obj, field=new)`.
5. **Signals failure via a Result-like return** — a frozen dataclass union like `Ok[T] | Err[E]`, not by raising. Raising is reserved for programmer errors (invariant violations).

```python
from dataclasses import dataclass, replace
from datetime import datetime
import deal

@dataclass(frozen=True)
class Ok[T]:
    value: T

@dataclass(frozen=True)
class Err:
    message: str

@deal.pure
def apply_discount(price: int, percent: int) -> Ok[int] | Err:
    if not 0 <= percent <= 100:
        return Err(f"percent out of range: {percent}")
    return Ok(price - (price * percent // 100))
```

## Shell Module Rules

Shell functions are thin. They decode input, call one or more pure core functions, and act on the returned value. All I/O, subprocess, network, and clock reads live here. Shell functions are **not** `@deal.pure`.

```python
from datetime import datetime, UTC
from pathlib import Path
from . import core

def run_price_update(input_path: Path, output_path: Path) -> None:
    raw = input_path.read_text(encoding="utf-8")
    now = datetime.now(UTC)
    result = core.parse_and_discount(raw, now)
    match result:
        case core.Ok(value):
            output_path.write_text(value, encoding="utf-8")
        case core.Err(message):
            raise SystemExit(f"update failed: {message}")
```

## Common Commands

```bash
uv sync                              # install + lock
uv add <pkg>                         # add runtime dep
uv add --dev <pkg>                   # add dev dep
uv run pytest                        # run tests
uv run pytest --cov=src              # tests with coverage
uv run ruff check .                  # lint
uv run ruff format .                 # format
uv run ty check                      # type check
uv run python -m <package>           # run a module
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| `datetime.now()` inside `core.py` | Pass `now: datetime` as a parameter from shell. |
| Regular `@dataclass` in core | Use `@dataclass(frozen=True)` and `dataclasses.replace`. |
| `raise ValueError(...)` in core | Return `Err("...")` instead. |
| Importing `subprocess` in core | Move the logic to shell and call core with decoded input. |
| Running `python foo.py` | Always `uv run python foo.py` (or `uv run python -m pkg`). |
| Inline `python -c "..."` | Blocked by hook. Write to `claude-tools/<name>.py`, run via `uv run`. |
| Adding `mypy` or `black` | Use `ty` and `ruff format`. |
| Module-level `_cache = {}` in core | Move state into shell, or pass as parameter and return next state. |

## Testing Notes

- **Core tests are deterministic unit tests.** No mocks, no fixtures that touch the network or filesystem. Feed pure inputs, assert on returned values.
- **Shell tests are integration tests.** Use real tmp dirs (`tmp_path` fixture), real subprocesses where feasible. Mock only external services you don't own.
- **Never mock a pure function** — if a core function is hard to use in a test, it's not actually pure or not actually small enough.

## Red Flags

- `import deal` missing from a file named `core*.py`
- Any `open(...)`, `Path(...).read_*`, `requests.`, `httpx.`, `subprocess.`, `os.environ`, `datetime.now()` inside `core*.py`
- A function in `core*.py` without `@deal.pure`
- `@dataclass` without `frozen=True` in core
- `try: ... except Exception: ...` in core (exceptions are a shell concern)

## After Implementing a Feature

After a substantive feature commit (anything that adds a new module, function family, or non-trivial behavior), dispatch the `code-simplifier:code-simplifier` subagent on the changed files before moving on. It catches over-abstraction, single-use helpers, and redundant logic that the write-time "Simplicity first" rule didn't prevent. Skip for one-line fixes, doc/config edits, and pure refactors.
