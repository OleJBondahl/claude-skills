# claude-skills

Claude Code skills for AI-assisted software engineering. Works with [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex](https://openai.com/index/introducing-codex/), [Copilot CLI](https://docs.github.com/en/copilot), and [Gemini CLI](https://github.com/google-gemini/gemini-cli).

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

### Development workflow

| Skill | Description |
|---|---|
| [brainstorming](brainstorming/) | Explore intent, requirements, and design before implementation |
| [writing-plans](writing-plans/) | Write implementation plans with bite-sized TDD-driven tasks |
| [executing-plans](executing-plans/) | Execute plans in separate sessions with review checkpoints |
| [test-driven-development](test-driven-development/) | Red-green-refactor: write test first, watch it fail, write minimal code |
| [systematic-debugging](systematic-debugging/) | Find root cause before attempting fixes using evidence gathering |
| [verification-before-completion](verification-before-completion/) | Run verification commands before claiming success |
| [finishing-a-development-branch](finishing-a-development-branch/) | Present merge/PR/cleanup options when work is done |
| [requesting-code-review](requesting-code-review/) | Dispatch code-reviewer subagent to catch issues before merge |
| [receiving-code-review](receiving-code-review/) | Evaluate review feedback with rigor, not performative agreement |

### Agent orchestration

| Skill | Description |
|---|---|
| [dispatching-parallel-agents](dispatching-parallel-agents/) | Delegate 2+ independent tasks to agents in parallel |
| [subagent-driven-development](subagent-driven-development/) | Execute plans by dispatching one subagent per task with two-stage review |
| [haiku-delegate](haiku-delegate/) | Delegate mechanical tasks to haiku model to preserve context |
| [expert-panel](expert-panel/) | Evaluate complex decisions with adversarial multi-agent panel |
| [using-superpowers](using-superpowers/) | Load and invoke relevant skills before any response |

### Documentation and writing

| Skill | Description |
|---|---|
| [doc-writer](doc-writer/) | Concise technical docs following Diátaxis + Google style standards |
| [humanizer](humanizer/) | Remove AI-generated writing patterns from text |
| [apa-citations](apa-citations/) | APA 7th edition citations using Pandoc + citeproc |
| [writing-skills](writing-skills/) | Author new skills using TDD methodology |
| [visual-review](visual-review/) | Evaluate visual output (SVG, PDF, HTML) using PNG renders |

### Language tooling

| Skill | Description |
|---|---|
| [python-coding-and-tooling](python-coding-and-tooling/) | Python repos with uv, ruff, ty, pytest, deal, functional core/shell |
| [typescript-coding-and-tooling](typescript-coding-and-tooling/) | TS repos with strict tsconfig, ESLint, Vitest, neverthrow |

### Infrastructure

| Skill | Description |
|---|---|
| [deploy-remote](deploy-remote/) | Deploy to remote hosts via SSH with build-transfer-restart pipeline |
| [ssh-remote](ssh-remote/) | Run commands on remote machines or WSL from Windows |
| [using-git-worktrees](using-git-worktrees/) | Create isolated git worktrees with safety verification |

### Codebase knowledge graph

| Skill | Description |
|---|---|
| [codebase-memory-exploring](codebase-memory-exploring/) | Search and explore code via knowledge graph |
| [codebase-memory-quality](codebase-memory-quality/) | Find dead code, unused functions, and complexity hotspots |
| [codebase-memory-reference](codebase-memory-reference/) | MCP reference guide for graph queries and Cypher syntax |
| [codebase-memory-tracing](codebase-memory-tracing/) | Trace call chains and dependencies for impact analysis |
| [updating-memory](updating-memory/) | Keep MCP memory server current when code changes |

### Product strategy

| Skill | Description |
|---|---|
| [jobs-to-be-done](jobs-to-be-done/) | Uncover customer jobs, pains, and gains in JTBD format |
| [opportunity-solution-tree](opportunity-solution-tree/) | Build OST from outcomes to opportunities, solutions, and tests |
| [positioning-statement](positioning-statement/) | Create Geoffrey Moore-style positioning statement |
| [problem-framing-canvas](problem-framing-canvas/) | MITRE Problem Framing Canvas for problem clarity before solutions |
| [product-strategy-session](product-strategy-session/) | End-to-end strategy across positioning, discovery, and roadmap |
| [roadmap-planning](roadmap-planning/) | Plan roadmap across prioritization, epics, and sequencing |

### SimpleHarness

| Skill | Description |
|---|---|
| [simpleharness-task](simpleharness-task/) | Author tasks for [SimpleHarness](https://github.com/OleJBondahl/SimpleHarness) to execute hands-off |

## License

MIT
