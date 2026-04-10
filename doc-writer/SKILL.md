---
name: doc-writer
description: >
  Use when writing or editing technical documentation: README, API reference,
  how-to guide, tutorial, changelog, usage docs, runbook, compliance doc,
  guidelines, or any .md/.qmd file explaining code, tools, or processes.
  Triggers on: "write docs", "document this", "add a README", "create a guide",
  "write a tutorial", "update the changelog", "write a runbook".
---

# Doc Writer

Write concise technical docs. Identify the doc type, draft to that type's rules, cut the filler, then run humanizer.

**Output:** `.md` or `.qmd` (use `.qmd` with Typst frontmatter for formal/external docs).

**MANDATORY FINAL STEP:** Invoke the `humanizer` skill on every completed draft.

## Step 1 — Pick the doc type

| Type | Answers | Tone | Structure |
|---|---|---|---|
| Tutorial | "How do I learn this?" | Prescriptive, sequential | Numbered steps with outcomes |
| How-to | "How do I do X?" | Direct, skip context | Goal → steps → done |
| Reference | "What are the options?" | Facts only | Tables, no prose |
| Explanation | "Why does this work this way?" | Conceptual | Short paragraphs |
| Runbook | "What do I do when X breaks?" | TL;DR first, then steps | Summary → procedure → recovery targets |
| Compliance | "Does this meet standard Y?" | Formal, versioned | Numbered sections, .qmd with Typst frontmatter |
| Guidelines | "What rules apply here?" | Rules with rationale | Rule → why → enforcement |

Pick one. Do not mix types in a single doc.

## Step 2 — Draft

- [ ] Active voice — "Run the command", not "The command should be run"
- [ ] Second person — "you", not "the user"
- [ ] Present tense — "returns", not "will return"
- [ ] One idea per paragraph
- [ ] Specific verbs — "configure", "run", "delete", not "manage", "handle", "utilize"
- [ ] Tables and lists over prose for structured info
- [ ] Assume competence — skip obvious explanations
- [ ] No heading restatement — if the heading says "Installation", don't open with "To install..."

## Step 3 — Cut 20%

Default AI drafts contain 20-30% filler. Delete before polishing.

**Delete on sight:**

| Pattern | Replace with |
|---|---|
| "In order to" | "To" |
| "It should be noted that" | (state it directly) |
| Moreover, furthermore, additionally, consequently | (delete — let ideas flow) |
| Crucial, significant, comprehensive, pivotal, key | (delete or use a specific adjective) |
| "In today's...", "As you may know..." | (delete) |
| "Let's dive in", "Here's what you need to know" | (delete — start with content) |
| "The future is bright", "Exciting times ahead" | (delete or state a specific plan) |
| "This is a powerful/robust/flexible..." | (delete — show, don't tell) |
| Any sentence repeating earlier information | (delete) |
| Any sentence adding no new information | (delete) |

**Rule:** If removing a sentence loses nothing, remove it.

## Step 4 — Humanizer pass

Invoke the `humanizer` skill on the finished draft. Not optional. Every doc, every time.

## Length guidance

| Doc type | Aim for |
|---|---|
| README | Under 300 words + code examples |
| How-to | Under 500 words |
| Tutorial | 800-1200 words |
| Reference | No limit — one row per item |
| Runbook | Under 400 words |
| Changelog entry | 1-3 sentences per change |
| Explanation | 300-600 words |
| Compliance | As needed — but no padding |
| Guidelines | Under 400 words per topic |

Exceed when content requires it. Never pad to fill.

## Anti-patterns

- **Type mixing** — tutorial + reference in one doc. Split them.
- **Over-contextualizing** — three paragraphs of "why this exists" before the instructions. Lead with the action.
- **Hedged instructions** — "you may want to consider running..." → "run"
- **Schema dumps** — pasting a full schema when a 3-field example suffices. Link to the full schema.
- **Directory trees** — showing the entire project layout when only 3 paths matter.
- **Repeating across sections** — if it's in the intro, cut it from the body.
- **Feature lists restating the how-to** — if "How it works" explains it, don't repeat in "Key features".
- **Verbose command tables** — list the 5 commands people use. Link to `--help` for the rest.

## Before / After

**Before (typical AI draft):**

> ## Getting Started
>
> Getting started with SimpleHarness is straightforward. In order to begin,
> it's important to ensure that you have all the necessary dependencies
> installed. SimpleHarness is a powerful and flexible tool that plays a
> crucial role in managing your Claude Code sessions effectively.

**After:**

> ## Getting started
>
> ```bash
> uv tool install simpleharness
> simpleharness init
> simpleharness new "your task title"
> simpleharness watch
> ```
>
> This scaffolds a `simpleharness/` directory, creates a task, and starts
> the supervision loop.
