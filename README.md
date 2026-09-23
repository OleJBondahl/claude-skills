# claude-skills

Claude Code skills for AI-assisted software engineering. Works with [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex](https://openai.com/index/introducing-codex/), [Copilot CLI](https://docs.github.com/en/copilot), and [Gemini CLI](https://github.com/google-gemini/gemini-cli).

The development-workflow skills (brainstorming, plans, TDD, debugging, reviews, worktrees) come from the [superpowers](https://github.com/obra/superpowers) plugin and are no longer kept here.

## Install

**Clone the full collection** into your skills directory:

```bash
git clone https://github.com/OleJBondahl/claude-skills.git ~/.claude/skills
```

**Or copy individual skills** — each skill is a self-contained folder with a `SKILL.md`:

```bash
# Example: grab just the doc-writer skill
cp -r claude-skills/doc-writer ~/.claude/skills/doc-writer
```

## Skills catalog

### Documentation and writing

| Skill | Description |
|---|---|
| [doc-writer](doc-writer/) | Concise technical docs following Diátaxis + Google style standards |
| [humanizer](humanizer/) | Remove AI-generated writing patterns from text |
| [visual-review](visual-review/) | Evaluate visual output (SVG, PDF, HTML) using PNG renders |

### Language tooling

| Skill | Description |
|---|---|
| [python-coding-and-tooling](python-coding-and-tooling/) | Python repos with uv, ruff, ty, pytest, deal, functional core/shell |
| [reviewing-ai-generated-python](reviewing-ai-generated-python/) | Audit AI-authored Python for over-engineering, defensive bloat and shallow tests |
| [typescript-coding-and-tooling](typescript-coding-and-tooling/) | TS repos with strict tsconfig, ESLint, Vitest, neverthrow |
| [skidl](skidl/) | SKiDL code and KiCad netlists from Python |

## License

MIT
