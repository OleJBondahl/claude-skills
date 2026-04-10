---
name: python-coding-and-tooling
description: Use when writing Python code, setting up a new Python repo, configuring pyproject.toml, choosing linters/type checkers/test runners, or structuring modules into functional core and imperative shell. Covers uv, ruff, ty, pytest, deal, pyproject.toml layout, and frozen dataclass patterns.
---

# Python Coding and Tooling

## Overview

Opinionated baseline for Python projects in this user's ecosystem. Applies the global Code Style Philosophy (functional core / imperative shell) to Python concretely: which tools, which file layout, which decorators, which commands.

## Mandatory Toolchain

| Concern | Tool | Notes |
|---|---|---|
| Package manager | `uv` | Never `pip`, never `poetry`, never `pipenv`. |
| Runner | `uv run` | Never bare `python`. Never `python -c` (hook-blocked). |
| Python version | `3.13` | Set `requires-python = ">=3.13"` and `target-version = "py313"`. |
| Linter + formatter | `ruff` | One tool does both. No `black`, no `flake8`, no `isort`. |
| Type checker | `ty` | Astral's type checker. Not `mypy`, not `pyright`. |
| Tests | `pytest` | With `pytest-cov` for coverage. |
| Purity enforcement | `deal` + `fp-purity-gate` AST script | `@deal.pure` on every function in a core module. |

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
requires-python = ">=3.13"
dependencies = ["deal"]

[dependency-groups]
dev = ["pytest", "pytest-cov", "ruff", "ty"]

[tool.ruff]
line-length = 100
target-version = "py313"

[tool.ruff.lint]
extend-select = ["I", "B", "UP", "SIM", "RUF"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers"
```

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

Never chain with `&&` / `;` — a hook blocks it. Run each command separately.

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
