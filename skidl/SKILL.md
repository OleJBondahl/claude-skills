---
name: skidl
description: Use when writing or editing SKiDL code, scaffolding a KiCad-netlist-from-Python project, debugging generate_netlist output, resolving NC/netclass/subcircuit issues, or configuring ruff/ty for a SKiDL codebase.
---

# SKiDL

## Overview

SKiDL (v2.2.3) is a Python DSL for describing circuits and emitting KiCad netlists. It relies heavily on operator overloading (`+=`), runtime-injected builtins (`NC`), and dynamic attributes — so lints and type checkers need calibrated relaxations per file.

## When NOT to use

- Hierarchical schematics with reusable subcircuits — SKiDL's `@subcircuit` / `@package` emit non-`/` sheetpaths that conflict with refdes-based `pcbnew` updates. Use KiCad's schematic editor.
- Bus-heavy designs where named pairs matter — `Bus(name, width)` renames your nets at emit time.
- Projects where the user wants to keep editing in the KiCad schematic editor — netlists from SKiDL are one-way.

## File layout that works

Flat, connector-heavy boards. Generalise package name to `<pkg>`.

```
src/<pkg>/
    __init__.py     # set_default_tool(KICAD) + stop_log_file_output(True)
    __main__.py     # from .generate import main; main()
    templates.py    # SkidlPart templates + footprint string constants
    components.py   # module-level refdes instantiations (F1, J1, K1…)
    nets.py         # module-level Net() declarations and += wiring
    generate.py     # imports nets for side effects, calls generate_netlist
```

Components and nets materialise at import time. No helper functions — flat and imperative.

## Native idioms (copy these)

Inline template with explicit pins (no `.kicad_sym` dependency):

```python
from skidl import Pin, SkidlPart
fuse = SkidlPart(
    name="Fuse", ref_prefix="F",
    pins=[Pin(num=n, name="", func=Pin.types.PASSIVE) for n in ("1", "2")],
)
```

Instantiate — refdes locked at construction time so regeneration is stable:

```python
F1 = fuse(ref="F1", value="4A", footprint=FP_FUSE)
```

Declare then wire (never on one line — see gotchas):

```python
sig = Net("/my_signal")
sig += J7[1], K5[14]
```

Power-class tag (plural property):

```python
power = NetClass("Power", priority=1)
gnd = Net("GND")
gnd.netclasses = power
```

No-connect pins via SKiDL's builtin singleton:

```python
NC += J1[4], J3[6], K2[21], K2[24]
```

Pin addressing — numeric as int, alphanumeric as string:

```python
J2[8]       # int
K1["A1"]    # str — relay coil / multi-pole pins
```

Entry point:

```python
from skidl import generate_netlist
generate_netlist(file_=str(out_path), do_backup=False)
```

## Tooling baseline

`uv` + Python 3.13 + `ruff` + `ty`. No pytest — a SKiDL script that builds one netlist has nothing to unit-test; "it ran with 0 errors" is the contract.

`pyproject.toml` ruff essentials:

```toml
[tool.ruff.lint]
select = ["ALL"]
ignore = ["D", "COM812", "ISC001", "ANN401", "CPY001", "T201"]

[tool.ruff.lint.per-file-ignores]
"src/<pkg>/components.py" = ["N816", "E501"]   # F1/J1/K1 uppercase mirror refdes
"src/<pkg>/nets.py"       = ["F821", "E501"]   # bare NC is a skidl builtin
"src/<pkg>/templates.py"  = ["E501"]
```

ty override to silence `NC` in `nets.py`:

```toml
[[tool.ty.overrides]]
include = ["src/<pkg>/nets.py"]
[tool.ty.overrides.rules]
unresolved-reference = "ignore"
```

## Gotchas

- `Net(…) += pin` is **illegal Python** — augmented assignment can't target a call result. Bind first: `sig = Net(…); sig += pin`.
- `net.netclasses = power` (**plural**) is the real setter. `net.netclass = …` (singular, shown in SKiDL docstrings) silently sets an instance attribute and never reaches the setter — your net stays `Default`.
- `NC` isn't importable. It's injected into `builtins` by `skidl.skidl:59` on first import. Bare `NC` resolves at runtime but trips `F821` / `unresolved-reference` — configure per-file ignores.
- SKiDL opens `{argv[0]}.erc` and `.log` in CWD at import time (`logger.py:428–430`). Call `stop_log_file_output(stop=True)` right after importing `skidl`, ideally in the package `__init__.py`.
- `generate_netlist(do_backup=False)` suppresses the `{script}_sklib.py` backup dump. With `do_backup=False` no `chdir` / CWD management is needed.
- `SkidlPart` is `functools.partial(Part, …)`, not a class. Don't use it as a return-type annotation.
- Pins connected to `NC` are filtered from the emitted netlist (`circuit.py:get_nets` skips `NCNet`). On import, pcbnew assigns those pads to net code 0 and prints one informational "No net found" line per pad — no DRC error, no layout damage.
- `SkidlPart` templates avoid any dependency on `.kicad_sym` files; inline them when the libraries aren't guaranteed on disk.

## Anti-patterns (don't adopt even though docs mention them)

- `@subcircuit` / `@package` — push a hierarchy level and emit non-`/` sheetpaths per component. pcbnew treats a sheetpath change as a structural change on refdes-based update. Skip for flat boards.
- `Bus(name, width)` — expands to `BUS[0]`, `BUS[1]` nets at emit time, renaming your named pairs (`/sig_a`, `/sig_b`). Breaks net labels on the PCB side.
- Data-driven dict/dataclass tables looped through a `build()` helper. Clever but un-idiomatic. Every devbisme example is imperative top-to-bottom, like reading a schematic. Keep it that way.
- Wrapping `SkidlPart(...)` in a custom factory helper. Either inline or `functools.partial` directly — usually the repetition is fine and clearer.

## `SkidlPart` vs `Part(lib, …)`

Use `SkidlPart` with inline `Pin` lists when the `.kicad_sym` library isn't guaranteed on disk — the schematic becomes fully self-contained in Python and emits `(libsource (lib "NO_LIB"))`. Use `Part("LibName", "PartName", ref=…, …)` with `lib_search_paths["kicad9"]` extended when the user has the real symbol libraries and wants richer metadata (keywords, fp filters) in the netlist.

Full API reference: see `claude-tools/skidl_notes.md` in a SKiDL project, or query context7 for `skidl` docs.
