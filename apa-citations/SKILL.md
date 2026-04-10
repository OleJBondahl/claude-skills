---
name: apa-citations
description: Use when adding APA 7th edition citations and reference lists to academic markdown documents (essays, theses, reports). Wraps Pandoc + citeproc + the official CSL apa.csl style. Fetches BibTeX from DOIs via Crossref content negotiation. Norwegian and English locale supported. Verify before delivering — Pandoc/CSL is correct by construction, but the metadata you feed in is not.
---

# APA 7 citations via Pandoc + citeproc

## When to use this skill

Use when the user is writing an academic markdown document and needs:

- In-text citations in APA 7 (narrative `Stray et al. (2020)` or parenthetical `(Stray et al., 2020)`)
- A correctly formatted reference list at the end
- Norwegian-locale handling (e.g. `lang: nb-NO` for "u.å." instead of "n.d.")

Do **not** use this skill for: footnote-style citations (Chicago notes-bib), arbitrary style guides where the user only needs one or two references and just wants them typed out, or quick informal blog posts.

## Why this stack and not something else

Pandoc + citeproc + `apa.csl` is the de facto academic Markdown stack. The `apa.csl` style file is the canonical APA 7 implementation, maintained by the [Citation Style Language project](https://github.com/citation-style-language/styles) — the same one Zotero and Mendeley use. Output is correct *by construction*; the only thing that can go wrong is the metadata you feed in.

Avoid: hand-written APA from LLM memory (formatting drift), unproven third-party Claude skills, or hobbyist DOI-formatter sites. They all share the same failure mode — wrong punctuation in places no human will notice until a grader does.

## Prerequisites — verify before any work

1. **Pandoc 2.11+** (citeproc became built-in at 2.11; current is 3.x). Check with `pandoc --version`.
2. **`apa.csl`** in the project. Download the latest from the official repo:
   ```bash
   curl -fsSL -o apa.csl https://raw.githubusercontent.com/citation-style-language/styles/master/apa.csl
   ```
3. **A bibliography file** — `refs.bib` (BibTeX) or `refs.json` (CSL-JSON). Both work; BibTeX is easier for humans to inspect.

If Pandoc is missing on Windows: `winget install --id JohnMacFarlane.Pandoc`. The installer puts it at `%LOCALAPPDATA%\Pandoc\pandoc.exe` and adds that directory to the user PATH — but **existing shells will not see it until restarted**. If you must use Pandoc in the current session before a restart, call it by full path.

## Workflow

### 1. Get metadata from DOIs (preferred)

Most academic articles have a DOI. Crossref's content-negotiation API returns publisher-provided BibTeX directly:

```bash
curl -fsSLH 'Accept: application/x-bibtex' 'https://doi.org/10.1109/MS.2018.2875988'
```

Append the result to `refs.bib`. Do not retype author names or fix capitalization — leave the publisher metadata as-is unless it is obviously wrong (e.g. all-caps titles, which BibTeX preserves with `{}` braces around words you want kept verbatim).

For DOI variants: `https://api.crossref.org/works/{doi}/transform/application/x-bibtex` works as an alternative, and `application/vnd.citationstyles.csl+json` returns CSL-JSON if you prefer that input format.

### 2. Hand-build entries for sources without DOIs

Some chapters, books, and Cutter/Amplify-style trade articles lack DOIs. In that case write the BibTeX entry by hand from the original PDF's title page. Common entry types:

- `@book` for full books (Sommerville, Cohn)
- `@incollection` for a chapter in a book with a different editor
- `@article` for journal/magazine articles (use even for IEEE Software, *Communications of the ACM*, etc.)
- `@misc` for white papers, blog posts, or anything that doesn't fit

Required fields by type are documented at the [BibTeX entry types reference](https://www.bibtex.com/e/entry-types/). Pandoc/citeproc is forgiving about extra fields but will silently produce wrong output if a required field is missing — for example, an `@article` without a `journal` field will not show the journal title in the reference list.

### 3. Wire the document

Add YAML frontmatter to the markdown document:

```yaml
---
lang: nb-NO
bibliography: refs.bib
csl: apa.csl
---
```

`lang` controls localization (terms like "n.d.", "ed.", "and"). For English use `en-US` or `en-GB`. For Norwegian use `nb-NO` (Bokmål) or `nn-NO` (Nynorsk).

Then cite inline:

- Narrative: `@Stray_2020` → "Stray et al. (2020)"
- Parenthetical: `[@Stray_2020]` → "(Stray et al., 2020)"
- Multiple: `[@Stray_2020; @Dingsoyr_2022]` → "(Dingsøyr et al., 2022; Stray et al., 2020)"
- With page locator: `[@Stray_2020, p. 73]` → "(Stray et al., 2020, p. 73)"
- Suppress author (for "as Stray et al. showed"): `[-@Stray_2020]` → "(2020)"

Add a heading where the reference list should appear. Pandoc will inject the bibliography immediately *after* this heading:

```markdown
# Litteraturliste
```

### 4. Render

```bash
pandoc --citeproc --from=markdown --to=markdown --wrap=none \
       Individuell_refleksjon.md -o Individuell_refleksjon_rendered.md
```

Flags:
- `--citeproc` — enables the citation processor (this is the only required new flag).
- `--from=markdown --to=markdown` — keep it in markdown rather than producing PDF/HTML/DOCX. Useful when you want to verify before final export.
- `--wrap=none` — don't hard-wrap output lines, so diffs stay clean.

For a final deliverable, swap the output: `-o essay.pdf` or `-o essay.docx`. PDF requires a LaTeX engine (TeX Live is fine — the user already has it at `C:\texlive\2025\`).

### 5. Verify

The processor is correct by construction, but **the metadata is not**. Always eyeball the rendered reference list against the original source. The most common issues:

- Title case wrong because BibTeX dropped the braces. Fix by wrapping proper nouns in `{Braces}` in the `.bib` entry: `title = {Daily Stand-Up Meetings: Start Breaking the Rules}`.
- Author name initials wrong because the publisher's metadata used full names that Crossref couldn't parse. Fix manually in the `.bib` file.
- Wrong year because Crossref returned the *online-first* year, not the official issue year. Check the source PDF.
- Missing DOI URL because the entry has no `doi` field. Add `doi = {10.1109/...}`.

For a 5–10 entry reference list this verification takes about three minutes and catches every realistic problem.

## Reference: APA 7 quick rules to spot-check

- 3+ authors → "Author et al. (Year)" from the **first** citation (this changed from APA 6, which spelled out all authors first time).
- Reference list uses `&` before the last author, not "and": `Stray, V., Moe, N. B., & Sjoberg, D. I. K.`
- Journal title and volume number in italics. Issue number in parens, not italics: `*IEEE Software*, *37*(3)`.
- Page range with en-dash, not hyphen: `70–77` not `70-77`. Pandoc handles this if you write `70--77` in BibTeX.
- DOI as a URL, not as `DOI: 10.x.y/z`: `https://doi.org/10.1109/ms.2018.2875988`.
- Hanging indent on each reference list entry — not visible in markdown output, but applied automatically when rendering to PDF/DOCX.

If the rendered output violates any of these, the bug is in the BibTeX entry, not in apa.csl.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `[@Stray_2020]` appears literally in the output | `--citeproc` flag missing, or bibliography file path wrong | Add `--citeproc`; check `bibliography:` path is relative to the markdown file |
| Reference list is empty | No `# References`/`# Litteraturliste` heading in the document | Add the heading where the bibliography should go |
| Author shows as "Anonymous" | BibTeX entry has no `author` field | Add `author = {Lastname, Firstname}` |
| "n.d." instead of year | BibTeX entry has no `year` field | Add `year = {2020}` |
| Title in lowercase like "daily stand-up meetings" | BibTeX sentence-case conversion stripped capitals | Wrap protected words in braces: `title = {Daily {Stand-Up} Meetings}` |
| Et al. when only 2 authors | apa.csl uses et al. for 3+. With 2 authors it should show both. If it doesn't, the BibTeX `author` field is malformed | Authors in BibTeX must be separated by `and`, not commas: `author = {Smith, J. and Jones, K.}` |
| Citation in unexpected language | Wrong `lang` in frontmatter | Set `lang: nb-NO` for Norwegian Bokmål, `en-US` for US English |

## Updating apa.csl

CSL styles are versioned and updated occasionally as APA publishes corrections. To refresh:

```bash
curl -fsSL -o apa.csl https://raw.githubusercontent.com/citation-style-language/styles/master/apa.csl
```

The file is small (~25 KB) and idempotent to overwrite.
