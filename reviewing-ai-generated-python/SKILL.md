---
name: reviewing-ai-generated-python
description: Use when auditing an AI-authored Python codebase for quality, simplicity, inconsistency, and AI-specific code smells — distinct from a generic code review because it targets the patterns that distinguish LLM output from human output (over-engineering, defensive bloat, phantom features, inflated docstrings, inconsistent error contracts, hallucinated APIs, shallow tests, type-annotation theater).
---

# Reviewing AI-Generated Python

## Overview

A focused playbook for reviewing Python code written primarily by AI coding agents (Claude Code, Copilot, Cursor, Codex). Generic code review misses the patterns that are specific to LLM output; this skill lists those patterns with concrete grep/AST signals so a review subagent can produce evidence, not vibes.

**Core principle:** Every finding must come with file path, line number, and a 1-line reason. Reviewers that say "this feels over-engineered" are useless. Reviewers that say `foo.py:47 — SingleMethodClass pattern, called once in bar.py:12, replace with function` are actionable.

## When to Use

Use when:
- Auditing an AI-authored repo for cleanup candidates
- Assessing an alpha project where breaking changes are fine
- Dispatched as a sub-reviewer covering one domain (API consistency, purity, testing, etc.)

Skip when:
- Reviewing a normal human-authored PR (use a standard code review skill)
- Reviewing a production repo where the priority is regression risk, not simplification

## Ground Rules for Subagent Reviewers

1. **Read-only.** Do not edit, do not stage, do not run formatters/fixers. Report only.
2. **Every finding = file:line + 1-line reason + suggested fix.** No vague observations.
3. **Triage into tiers** — `HIGH` (bug / broken abstraction), `MEDIUM` (simplification opportunity), `LOW` (style nit). Don't bury high-severity findings in noise.
4. **Prefer evidence over feelings.** If you claim "over-engineered," quote the symbol and show its 1 callsite.
5. **Budget your output** — 800–1500 words per report, aggressively structured. Long reports go unread.
6. **Do not re-review the same domain twice.** Stay in lane; other agents cover other domains.

## The 12 AI-Code Smells (with concrete signals)

### 1. Over-engineering & speculative generality
**Signals**
- Class with one public method → should be a function
- ABC / `Protocol` / Registry with exactly one concrete implementation
- Dataclass fields that are never read (`rg '<field>' | wc -l` ≈ 1)
- `**kwargs` passthroughs on functions with 1–2 callers
- Strategy pattern gated on a single value

**How to confirm**
`rg -w '<ClassName>' src/` — if the class name appears only at definition + 1 or 2 callsites, it's speculative. Ask: what second implementation would this enable? If none exists and none is planned, collapse it.

**Fix:** Inline to a function or the single caller. Delete the abstraction.

---

### 2. Defensive-programming bloat
**Signals**
- `try/except` around code whose only failure mode is a type error (the type checker should catch it)
- `if x is not None:` on parameters typed `X` (not `X | None`)
- Re-validation of arguments the caller already validated
- `getattr(obj, 'x', default)` on a declared attribute
- Catching an exception to re-raise with no added context

**How to confirm**
Walk one layer up in the call graph. If the guarded condition is impossible (enforced by type, upstream validation, or invariant), it's bloat.

**Fix:** Delete the guard. Let the type system or the invariant hold. Reserve real guards for I/O and external-input boundaries.

---

### 3. Phantom features & speculative API surface
**Signals**
- Public function parameters with defaults never overridden by any caller
- Enum values / `Literal[...]` options never referenced
- Branches whose guard is never true at runtime
- `utils.py` / `helpers.py` created alongside a feature that could live inline
- Exception classes defined but never raised

**How to confirm**
`rg '<param>\s*=' src/ tests/ | grep -v def` — if only default assignments appear, the parameter is dead. For enum values, `rg '<EnumClass>\.<VALUE>'` — 1 hit = only the definition.

**Fix:** Remove the parameter / branch / value / file.

---

### 4. Misleading / inflated docstrings & comments
**Signals**
- Docstrings on trivial functions that restate the signature
- Multi-paragraph `Args/Returns/Raises/Examples` on a 3-line function
- WHAT-not-WHY comments above obvious code
- "Used by X" / "See also Y" comments (rot fast)
- Section banners in short files

**How to confirm**
Skim each docstring: does it state a non-obvious WHY? If removing the docstring loses only a paraphrase of the code, delete.

**Concrete pattern to delete on sight:**

```python
# ❌ AI-bloat — the signature already says all of this
def add_terminal(self, tm_id: str, *, poles: int = 1) -> Terminal:
    """Add a terminal to the circuit.

    Args:
        tm_id: The terminal identifier.
        poles: The number of poles. Defaults to 1.

    Returns:
        The newly created terminal.
    """
    ...

# ✅ Either silent, or one line stating the non-obvious WHY
def add_terminal(self, tm_id: str, *, poles: int = 1) -> Terminal:
    """Auto-connects to the previous component in the chain unless `connect_from_previous=False`."""
    ...
```

**Fix:** Replace with a 1-line WHY-only docstring, or delete entirely. Reserve docstrings for invariants, edge cases, units, side-effect ordering, or non-obvious rationale. Companion: see `python-coding-and-tooling` skill's "Docstrings" section.

**Do NOT recommend** adding `darglint` / `darglint2` / `pydoclint` to enforce signature-docstring agreement. Those tools enforce the inflated-docstring pattern. The right fix is *deletion*, not enforcement.

---

### 5. Inconsistency (same concept, multiple expressions)
**Signals**
- Mixed error conventions: one function raises, a sibling returns `None`, another returns a `Result`
- Synonym proliferation: `get_user`, `fetch_user`, `load_user`, `retrieve_user`
- Mixed path handling: `os.path.join` + `pathlib.Path` + f-strings
- Import path drift: `from pkg import X` vs `from pkg.mod import X`
- Case-style drift

**How to confirm**
For each public surface (module or class), walk every function's return contract. If siblings disagree (raise vs Result vs None vs bool), flag.

**Fix:** Pick one convention per module. Document it in CLAUDE.md. Migrate the outliers.

---

### 6. Hallucinated APIs, ghost imports, wrong signatures
**Signals**
- Imports of packages not in `pyproject.toml`
- Method calls that look plausible but don't exist
- Wrong-version API (pydantic v1 on v2, etc.)
- Keyword arguments that don't exist in the callee
- `str.includes`, `list.find`, `dict.get_or_default` (JS/Ruby/etc. transfer)

**How to confirm**
Run `ty check` / `mypy`. Any `attr-defined` / `call-arg` / `import-not-found` error is a candidate. For external libs, verify signatures against current docs (use `mcp__plugin_context7_context7__resolve-library-id` then `query-docs`).

**Fix:** Replace with the real API.

---

### 7. Shallow / implementation-mirroring tests
**Signals**
- Tests that only exercise the happy path — no boundary / empty / failure cases
- Assertions equal fixture values constructed from the same inputs (tautology)
- Snapshot tests with no semantic assertion
- Tests that mock every collaborator (mock-everything)
- Test names mirror method names (`test_get_user`) with no behavioral suffix
- Test count disproportionate to behavior count

**How to confirm**
Pick 5 random tests. Mentally flip one operator in the production code under test (`+` → `-`, `<` → `<=`, `and` → `or`). Would the test still pass? If yes, the test is structural not behavioral. Also: `rg 'def test_' | wc -l` vs behavior count — if ratio is >2:1, suspect structural bloat.

**Fix:** Add boundary / failure / empty-input cases. Delete tests that only assert construction succeeded.

---

### 8. Duplication disguised as abstraction
**Signals**
- `utils.py` / `helpers.py` grab-bags of unrelated functions
- Single-use private helpers (`_do_x`, `_handle_y`) called once immediately below
- Two near-identical functions differing only in a string constant
- Wrappers that forward all args and add nothing

**How to confirm**
`rg -c '<helper_name>'` — 1 definition + 1 call = inline it. Human rule of three: two similar blocks = fine, three = extract.

**Fix:** Inline single-use helpers. Keep duplication until there are three instances.

---

### 9. Type-annotation theater
**Signals**
- `Any`, `object`, `dict[str, Any]` for structures that have a real shape
- `Optional[X] = None` on args that are never passed `None`
- `# type: ignore` without a specific error code
- `cast(X, y)` to silence an error that could be fixed
- Over-narrow `Literal[...]` on open sets
- Mixed `typing.List` / `list[...]` in the same repo on Py3.10+

**How to confirm**
Run `ty check` / `mypy --strict`. Each `Any` and unqualified `type: ignore` is a candidate. For each `Optional[X]`, grep for `None` in its call sites.

**Fix:** Narrow the type, remove the `Optional`, remove the `type: ignore`.

---

### 10. Dead code accumulation
**Signals**
- Commented-out alternative implementations
- `# TODO` / `# FIXME` with no ticket
- Unused imports, unused `__all__` entries
- Obsolete fallback paths after a refactor made them mandatory

**How to confirm**
`ruff check --select F401,F841` for unused imports/vars. `vulture` for dead code. `git blame` on comments — if they predate a refactor, they're stale.

**Fix:** Delete commented-out code; VCS is the memory. Delete TODOs without tickets. Remove unused symbols.

---

### 11. Inconsistent / silent error handling
**Signals**
- `except Exception:` catching everything
- `logger.error(e)` then return, with no re-raise or contextual info
- Mixed `raise ValueError` / `return None` / `return False` / `return {}` for similar failures
- Custom exception classes defined but never raised
- F-string error messages with no prose (`f"{x} {y}"`)

**How to confirm**
`rg 'except Exception' src/` — each occurrence needs justification. For each custom exception, `rg '<Exception>(' src/` — if only definition + `except` appear but no `raise`, it's dead.

**Fix:** Narrow the `except`. Pick one failure convention per module. Delete unraised exceptions.

---

### 12. Python-specific AI tells
- `list(map(lambda x: f(x), xs))` → `[f(x) for x in xs]`
- `str.format()` in a codebase that uses f-strings elsewhere
- `typing.List` / `typing.Dict` on Python 3.10+ (use `list[...]`, `dict[...]`)
- `if not isinstance(x, list): x = [x]` over-normalization
- `functools.reduce` for simple sums / products
- Redundant `return None` at function end
- `@staticmethod` on methods that should be module-level functions

---

### 13. Tool-stack bloat
**Signals**
- `bandit` listed in dev deps alongside `select = [..., "S"]` in ruff config
- `radon` listed alongside `select = [..., "C90", "PLR"]`
- `docstr-coverage` or `interrogate` alongside `select = [..., "D"]`
- `darglint` / `darglint2` / `pydoclint` (these enforce inflated docstrings — see smell #4)
- `pylint` alongside ruff
- `mypy` or `pyright` alongside `ty` (also produces dueling `# ignore[...]` syntax)
- `black`, `isort`, `flake8` alongside `ruff` (each replaced by ruff)

**How to confirm**
`grep -E '^\s*"(bandit|radon|docstr-coverage|interrogate|darglint|pydoclint|pylint|mypy|pyright|black|isort|flake8)' pyproject.toml`. Each match is a candidate. Cross-check `.pre-commit-config.yaml` for hooks invoking these.

**Fix:** Drop the redundant tool. Migrate its findings to the corresponding ruff rule set or `ty` if needed. Document the drop in `SUPPRESSIONS.md` / `CHANGELOG.md` so future contributors don't re-add it.

For darglint/pydoclint specifically: these enforce signature/docstring agreement, which contradicts smell #4's "minimize docstrings" recommendation. *Drop without replacement* — ruff's `D` rules cover docstring presence + format, and the rest is judgment work for human/AI review.

---

## Quick Reference — Invocation Checklist

When dispatched to review a Schematika module, run in order:

1. `scc <module>` — size/complexity baseline
2. `uv run ty check <module> 2>&1 | head -40` — type errors = smells #6, #9
3. `uv run ruff check <module>` — catches #10 (unused) and some of #12
4. `rg '# type: ignore|# noqa' <module>` — catches #9, inconsistency
5. `rg 'except Exception|except:' <module>` — catches #11
6. `rg 'TODO|FIXME|XXX|HACK' <module>` — catches #10
7. `rg -l '\bany\b|Any' <module>` — catches #9
8. Spot-read 3 files: the largest, the newest, and one random → catches #1, #3, #4, #8
9. For test review: mutate one operator mentally in each module and predict test behavior → catches #7

## Output Format (required)

Produce a markdown report with exactly these sections:

```
# Review: <module or topic>

## Summary
<3–5 sentences: state of the module, headline finding, overall tier>

## HIGH-severity findings
- <file:line> — <1-line description>. Fix: <concrete action>
- ...

## MEDIUM-severity findings
- ...

## LOW-severity findings
- ...

## Metrics
- LoC: <n> / tests: <n> / ty errors: <n> / ruff errors: <n>
- Specific counters relevant to this review

## Recommendations (prioritized)
1. <action>
2. <action>
```

No prose that isn't in one of those sections. No "overall I think..." paragraphs. Evidence or nothing.

## Red Flags in Your Own Report

If your review has any of these, rewrite it:
- Findings without file:line
- "This seems" / "This might" / "Possibly" without evidence
- Recommending "consider refactoring" without saying what to refactor to
- Reviewing code outside your assigned domain
- Summary longer than 5 sentences
- Any finding that could be "both over-engineered and under-engineered" — pick one

## Sources

Pattern catalog derived from:
- thoughtbot, "How to review AI-generated PRs"
- CodeRabbit, "State of AI vs Human Code Generation Report"
- Simon Willison, "Hallucinations in code are the least dangerous"
- Martin Fowler, "Patterns for Reducing Friction in AI-Assisted Development"
- arXiv papers on LLM code hallucination and over-mocked tests
- Adventures in Claude, "Two Weeks of Stomping Slop"
