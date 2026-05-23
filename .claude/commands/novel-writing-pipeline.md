---
name: novel-writing-pipeline
description: "Full novel-writing pipeline: outline → language/style config → chapter drafting → export. Also supports fusion mode (expand an existing novel). Orchestrates /novel-outline, /novel-fusion, /language-setting, /novel-style, /novel-write, and /novel-export. Use when user says \"写小说\", \"novel pipeline\", \"小说全流程\", \"帮我写本小说\", \"end-to-end novel\", \"融合写作\", \"扩写小说\", or wants to go from an outline to a finished document."
argument-hint: [outline-file-or-description — optional: language, style, output-format, fusion-source]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, Skill
---

# Novel Writing Pipeline: Outline → Finished Novel

Full novel pipeline for: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Constants

- **AUTO_PROCEED = false** — When `true`, automatically moves through Phase 1–3 (configuration) without waiting for confirmation. When `false` (default), pauses at each gate for user approval. Recommended: keep `false` for first-time runs so you can review the character bible and style settings before writing begins.
- **PIPELINE_MODE = "standard"** — `standard` (outline → write) or `fusion` (source novel + fusion outline → fused write). Auto-detected from arguments: if a source novel file is provided alongside an outline, mode is `fusion`. Override: `fusion`, `融合`, `扩写`.
- **FUSION_SOURCE = null** — Path to the existing novel for fusion mode. Detected from arguments (e.g., `source:novel.docx`, or a `.docx`/`.txt` file when fusion keywords are present).
- **LANGUAGE_MODE = "monolingual"** — `monolingual`, `bilingual`, or `dialogue-bilingual`. Override: `bilingual: true`, `dialogue-bilingual: true`, `台词双语`, or specify two languages (e.g., `中英双语`, `台词日语`).
- **PRIMARY_LANGUAGE = "zh"** — Language the novel is written in. Detected from arguments or outline language. Default: Chinese (`zh`).
- **SECONDARY_LANGUAGE = "en"** — Secondary language for bilingual mode. Default: English (`en`). Only used when `LANGUAGE_MODE = bilingual`.
- **GENRE = "auto"** — Genre/style. Auto-detected from outline if not specified. Override: `genre: romance / thriller / fantasy / literary / light`.
- **CHAPTER_WORD_COUNT = 8000** — Target words per chapter. Override: `每章N字` or `N words per chapter`.
- **OUTPUT_FORMAT = "epub docx"** — Output formats. Options: `txt`, `docx`, `pdf`, `epub`, `all`. Default: epub + docx.
- **AUTO_EXPORT = false** — When `true`, automatically exports after all chapters are written. When `false` (default), user calls `/novel-export` manually.
- **CHAPTER_RANGE = "all"** — Which chapters to write in the writing phase. Default: all chapters from the outline.
- **WRITER_PERSONA = null** — Writer persona name. If specified, looks for `writer_persona/{name}.yaml`. If null, uses the first `.yaml` alphabetically.

> Override any constant via argument: `/novel-writing-pipeline "outline.md — bilingual: zh/en, genre: romance, 每章3000字, output: epub pdf, auto_proceed: true"`

## Overview

This pipeline supports two modes:

### Standard Mode
```
/novel-outline  →  /language-setting  →  /novel-style  →  /novel-write  →  /novel-export
   (Phase 1)           (Phase 2)           (Phase 3)        (Phase 4)       (Phase 5)
 Structured Outline   Language Config     Style Config    Chapter Drafts   Final Output
 + CHARACTER_BIBLE
```

### Fusion Mode
```
/novel-fusion   →  /language-setting  →  /novel-style  →  /novel-write  →  /novel-export
   (Phase 1F)          (Phase 2)           (Phase 3)        (Phase 4)       (Phase 5)
 Source Novel +       Language Config     Style Config    Chapter Drafts   Final Output
 Fusion Outline →                                       (passthrough/new/modify)
 Fused OUTLINE.md
 + CHARACTER_BIBLE
 + CONTINUITY_MAP
```

**Phase 1 (standard)** — `/novel-outline` converts raw source material (PDF, text, notes) into a structured `OUTLINE.md` with per-chapter beats, character lists, and emotional arcs. Also creates `CHARACTER_BIBLE.md` if absent. Both can be skipped if the user provides them pre-made.

**Phase 1F (fusion)** — `/novel-fusion` reads an existing novel + a fusion outline, splits the source into chapters, and generates a fused `OUTLINE.md` with three chapter types (passthrough/modify/new), plus `CHARACTER_BIBLE.md`, `SOURCE_INDEX.md`, and `CONTINUITY_MAP.md`.

**Phase 4 architecture** — `/novel-write` uses a three-role pattern:
- Main agent: orchestrator (context loading, brief assembly, quality control)
- Writing subagent: prose generation (delegated via Agent tool)
- Proof-reader: structured review (character consistency, timeline, language quality, contamination, glossary)

This separation ensures the creative pass and the critical pass have independent context, reducing blind-spot errors.

**Persistent memory files** (live in `{project}/` throughout the pipeline):
- `{project}/OUTLINE_RAW.md` — original source text (reference, never modified)
- `{project}/OUTLINE.md` — structured writing outline with per-chapter beats (generated by `/novel-outline`)
- `{project}/characters/CHARACTER_BIBLE.md` — living character reference (grows during writing)
- `{project}/settings/LANGUAGE_SETTING.json` — language configuration
- `{project}/settings/STYLE_SETTING.json` — style configuration
- `{project}/draft/` — chapter files (primary language)
- `{project}/draft/[lang-code]/` — chapter files by language (bilingual)
- `{project}/NOVEL_STATE.json` — pipeline progress state
- `{project}/output/` — final exported files

## State Persistence

Persist pipeline state to `{project}/NOVEL_STATE.json` after each phase:

```json
{
  "title": "[Novel title]",
  "phase": 3,
  "auto_proceed": false,
  "language_mode": "bilingual",
  "primary_language": "zh",
  "secondary_language": "en",
  "genre": "romance",
  "chapter_word_count": 8000,
  "output_format": ["epub", "docx"],
  "total_chapters": 20,
  "chapters_written": 0,
  "status": "in_progress",
  "timestamp": "[ISO timestamp]"
}
```

On invocation, check this file:
- If absent or `status: "completed"` → fresh start
- If `status: "in_progress"` → **resume from saved phase** (read state, skip completed phases)
- Ask user: "Found an in-progress novel project ([title], Phase [N]/5). Resume or start fresh?"

## Pipeline

### Phase 0: Input Parsing & Project Initialization

Parse `$ARGUMENTS` to extract:

1. **Pipeline mode detection** — check for fusion signals:
   - Keywords: `fusion`, `融合`, `扩写`, `补写`, `expand`, `insert chapters`
   - Two file paths where one is labeled as source novel: `source:novel.docx`, `源小说:xxx`
   - Explicit `FUSION_SOURCE` path
   - If detected → set `PIPELINE_MODE = "fusion"`, extract `FUSION_SOURCE` path
2. **Outline source** — detect a file path (`.pdf`, `.md`, `.txt`, `.docx`) or inline text
   - In fusion mode: this is the **fusion outline** (what to add/change), not the source novel
3. **Chapter count** — detect from `chapters:N`, `N章`, or auto-detect
4. **Chapter word count** — detect from `每章N字`, `words:N`, or use CHAPTER_WORD_COUNT constant
5. **Language settings** — detect bilingual/monolingual/dialogue-bilingual
6. **Genre/style hints** — extract if mentioned
7. **Output format** — extract if specified
8. **Writer persona** — detect from `persona:name`, `人格:name`, `writer:name`
9. **Constant overrides** — update constants accordingly

**Initialize project directory:**

```bash
mkdir -p {project}/characters
mkdir -p {project}/settings
mkdir -p {project}/draft
mkdir -p {project}/output
mkdir -p {project}/assets
```

If no outline source provided: ask immediately.
```
No outline provided. Please share the story outline (paste text, file path, or PDF).
```

**Present pipeline overview:**

**Standard mode:**
```
Novel Writing Pipeline initialized.

Source: [file or "inline text"]
Title: [detected title or "to be determined"]

Pipeline configuration:
- Mode: Standard (new novel from outline)
- Chapters: [N or "auto-detect"]
- Words/chapter: [N] 字 target
- Language: [Monolingual zh / Bilingual zh→en / Dialogue-bilingual zh+ja]
- Genre: [detected or "to be configured"]
- Writer persona: [name or "default (first .yaml)"]
- Output: [formats]

Phases:
1. Outline Prep     → OUTLINE.md + CHARACTER_BIBLE.md
2. Language Setting  → LANGUAGE_SETTING.json
3. Style Config      → STYLE_SETTING.json
4. Chapter Writing   → {project}/draft/ (subagent + proof-reader)
5. Export            → {project}/output/ ([formats])

Proceed? (or adjust any setting before starting)
```

**Fusion mode:**
```
Novel Fusion Pipeline initialized.

Source novel: [file path]
Fusion outline: [file path or "to be provided"]
Title: [detected or "to be determined"]

Pipeline configuration:
- Mode: Fusion (expand existing novel)
- Words/chapter (new chapters): [N] 字 target
- Language: [Monolingual zh / Bilingual zh→en / Dialogue-bilingual zh+ja]
- Genre: [detected or "to be configured"]
- Writer persona: [name or "default (first .yaml)"]
- Output: [formats]

Phases:
1F. Novel Fusion    → Source split + Fused OUTLINE.md + CHARACTER_BIBLE.md + CONTINUITY_MAP.md
2.  Language Setting → LANGUAGE_SETTING.json
3.  Style Config     → STYLE_SETTING.json
4.  Chapter Writing  → {project}/draft/ (passthrough copies + subagent writes + proof-reader)
5.  Export           → {project}/output/ ([formats])

Proceed? (or adjust any setting before starting)
```

If AUTO_PROCEED=false: wait for user confirmation.
If AUTO_PROCEED=true: proceed after presenting the overview.

### Phase 1: Outline Structuring + Character Extraction (Standard Mode)

**If PIPELINE_MODE = "fusion"** → skip to Phase 1F below.

**Skip conditions:** If `{project}/OUTLINE.md` already exists AND is structured (has per-chapter `### Plot Beats` sections), AND `{project}/characters/CHARACTER_BIBLE.md` exists → skip to Phase 2. Report: "Structured outline and character bible found — skipping Phase 1."

Otherwise, invoke `/novel-outline`:

```
/novel-outline "[source-file] chapters:[N] words:[CHAPTER_WORD_COUNT]"
```

This does three things:
1. Reads the raw source material (PDF/text/markdown)
2. Produces a structured `{project}/OUTLINE.md` with per-chapter beats, character lists, emotional arcs
3. Creates `{project}/characters/CHARACTER_BIBLE.md` if absent (character extraction is a byproduct of outline analysis)

If images are extracted from a PDF source, `/novel-outline` also creates `{project}/assets/IMAGE_MAP.json`.

**🚦 Gate 1 — Outline + Character Review:**

After generation, present a summary:

```
Phase 1 complete.

Structured outline: {project}/OUTLINE.md
- Chapters: [N]
- Plot beats per chapter: [avg N]
- Act structure: [summary]

Character bible: {project}/characters/CHARACTER_BIBLE.md
- Major: [N] — [names]
- Supporting: [N] — [names]
- Minor: [N]
[Images: [N] extracted, [N] assigned to chapters]

Please review both files.
Adjust outline beats, chapter divisions, or character profiles?
```

**If AUTO_PROCEED=false:** Wait for user response. Options:
- **"OK" / "go"** → proceed to Phase 2
- **Adjust chapters** → re-run `/novel-outline` with different chapter count
- **Add/modify characters** → call `/character-design "add: [description]"` or edit directly
- **"stop"** → save state, pipeline pauses here

**If AUTO_PROCEED=true:** Present summary, proceed immediately.

**State**: Write `NOVEL_STATE.json` with `phase: 1`.

### Phase 1F: Novel Fusion (Fusion Mode Only)

**Only runs when PIPELINE_MODE = "fusion".** Skip if standard mode.

Invoke `/novel-fusion`:

```
/novel-fusion "[FUSION_SOURCE] [outline-file]"
```

This does:
1. Reads the source novel, splits into chapters → `{project}/source/`
2. Reads the fusion outline (what to add/change)
3. Builds a fusion map (passthrough / modify / new for each final chapter)
4. Generates fused `{project}/OUTLINE.md` with `TYPE:` annotations
5. Creates `{project}/characters/CHARACTER_BIBLE.md` (merged from source novel + fusion outline)
6. Creates `{project}/settings/CONTINUITY_MAP.md`
7. Creates `{project}/source/SOURCE_INDEX.md`
8. Writes `NOVEL_STATE.json` with `"mode": "fusion"`

**🚦 Gate 1F — Fusion Review:**

```
Phase 1F complete.

Source novel: [title] ([N] chapters, ~[N] words)
Fusion result: [N] total chapters
  - [N] passthrough (source as-is)
  - [N] new (original content)
  - [N] modify (source with changes)

Files created:
- {project}/source/          ← [N] source chapter files
- {project}/OUTLINE.md       ← fused outline with TYPE annotations
- {project}/characters/CHARACTER_BIBLE.md
- {project}/settings/CONTINUITY_MAP.md

Please review the outline — especially:
- Continuity anchors for new chapters
- Modification instructions for modify chapters
- Whether any passthrough chapters need upgrading to modify

Adjust? (or proceed to language configuration)
```

**If AUTO_PROCEED=false:** Wait for user response.
**If AUTO_PROCEED=true:** Present summary, proceed immediately.

**State**: Write `NOVEL_STATE.json` with `phase: 1`.

### Phase 2: Language Configuration

Invoke `/language-setting`:

```
/language-setting "[PRIMARY_LANGUAGE] [— SECONDARY_LANGUAGE if bilingual]"
```

This produces `{project}/settings/LANGUAGE_SETTING.json` and creates draft directories.

**🚦 Gate 2 — Language Confirmation:**

```
Language configuration:
- Mode: [Monolingual / Bilingual / Dialogue-bilingual]
- Primary: [language name]
[- Secondary: [language name] (translation: [style], [timing])]             (bilingual)
[- Writing language: [name], Dialogue language: [name] (style: [style])]    (dialogue-bilingual)

Confirm? (or adjust)
```

**If AUTO_PROCEED=false:** Wait for user confirmation.
**If AUTO_PROCEED=true:** Proceed immediately.

**State**: Write `NOVEL_STATE.json` with `phase: 2`.

### Phase 3: Style Configuration

Invoke `/novel-style`:

```
/novel-style "[GENRE] — [CHAPTER_WORD_COUNT]字"
```

This produces `{project}/settings/STYLE_SETTING.json` and `{project}/settings/CHAPTER_TEMPLATE.md`.

**🚦 Gate 3 — Style Confirmation:**

```
Style configuration:
- Genre: [genre name]
- POV: [perspective]
- Tense: [past/present]
- Chapter length: [N] 字 target
- Prose: [density] | Rhythm: [rhythm]

Confirm? (or adjust)
```

**If AUTO_PROCEED=false:** Wait for user confirmation. This is the last gate before autonomous writing begins.
**If AUTO_PROCEED=true:** Proceed immediately.

> ⚠️ **After Gate 3, writing begins.** Phases 4 and 5 run autonomously. If you want to review chapter-by-chapter, set AUTO_PROCEED=false and run /novel-write one chapter at a time after the pipeline finishes Phase 3.

**State**: Write `NOVEL_STATE.json` with `phase: 3`.

### Phase 4: Chapter Writing

Invoke `/novel-write`:

```
/novel-write "all"
```

This writes all chapters using the orchestrator + subagent + proof-reader pattern:

For each chapter:
1. Main agent loads context and assembles a writing brief
2. Writing subagent generates the chapter draft
3. Main agent runs proof-reader checks (character bible, timeline, language quality, contamination, glossary)
4. If NEEDS_REVISION: fix and re-review (max 2 cycles)
5. Save final draft, update state

- Handles monolingual, bilingual, and dialogue-bilingual modes
- Adds new characters to the bible as they appear (stubs created before writing)
- Saves each chapter to the appropriate path per language mode

**Fusion mode behavior**: `/novel-write` reads the `TYPE:` annotation on each chapter and routes accordingly:
- **passthrough** → copy source file to draft (fast, no subagent)
- **new** → full write with continuity anchors from adjacent source chapters
- **modify** → rewrite source chapter with modification instructions, matching source voice

See `/novel-write` Fusion Mode section for details.

**During writing, the pipeline updates `NOVEL_STATE.json` after each chapter.**

Progress updates (presented after each chapter):
```
Chapter [N]/[total] written — [N] 字
Proof-reader: [PASS / PASS_WITH_NOTES / NEEDS_REVISION→fixed]
[New character added: [name] (stub created)]
[Language mode: dialogue-bilingual (ja→zh inline)]
```

**If a chapter requires a new supporting/major character not in the bible:**
- `/novel-write` automatically creates a stub profile in the CHARACTER_BIBLE
- Logs: `"New character '[Name]' added to CHARACTER_BIBLE (Ch [N])"`
- Does NOT pause the pipeline for approval (stubs are clearly marked)

**State**: Write `NOVEL_STATE.json` with `phase: 4, chapters_written: N` after each chapter.

### Phase 5: Export

**If AUTO_EXPORT=false (default):** Present export options:

```
Writing complete!

Novel: [Title]
Total chapters: [N] | Total words: ~[N]
Draft location: {project}/draft/

To export:
/novel-export "[OUTPUT_FORMAT]"

Quick options:
/novel-export "epub docx"    → e-reader + Word
/novel-export "all"          → txt + docx + pdf + epub
/novel-export "pdf"          → PDF only
```

**If AUTO_EXPORT=true:** Automatically invoke:

```
/novel-export "[OUTPUT_FORMAT]"
```

**State**: Write `NOVEL_STATE.json` with `phase: 5, status: "completed"` after export.

### Final Report

After the full pipeline completes:

```markdown
# Novel Writing Pipeline Complete

**Title**: [Novel title]
**Date**: [start] → [end]

## What Was Created

| File | Description |
|------|-------------|
| {project}/OUTLINE.md | Source outline |
| {project}/characters/CHARACTER_BIBLE.md | [N] characters profiled |
| {project}/settings/STYLE_SETTING.json | Genre: [genre], [N] 字/chapter |
| {project}/settings/LANGUAGE_SETTING.json | [Mode], Primary: [lang] |
| {project}/draft/chNN.md (×N) | [N] chapters, ~[total] words |
| {project}/output/novel.[ext] | [formats] |

## Novel Statistics
- Chapters: [N]
- Total words: ~[N] ([primary lang])
[- Translation words: ~[N] ([secondary lang])]
- New characters added during writing: [N]
- Average chapter length: [N] 字

## Output Files
[Table of output files with sizes]

## Character Bible Summary
- Major characters: [N]
- Supporting characters: [N]
- Characters added mid-writing: [list or "none"]
```

## Output Protocols

- **Output versioning**: write timestamped copies before overwriting any existing file (CHARACTER_BIBLE, NOVEL_STATE, chapter files if restyling)
- **Large file handling**: if Write fails, retry with Bash heredoc silently
- **Manifest**: log every output file to `{project}/MANIFEST.md`

```bash
# MANIFEST.md format
| Timestamp | Skill | File | Description |
|-----------|-------|------|-------------|
| [time] | /character-design | {project}/characters/CHARACTER_BIBLE.md | [N] characters |
| [time] | /novel-write | {project}/draft/ch01.md | Chapter 1, [N] words |
```

## Key Rules

- **Never start Phase 4 without a complete outline and character bible.** If either is missing, fix it first.
- **Character bible is the source of truth.** If a chapter contradicts the bible, fix the chapter — never silently update the bible to match the contradiction.
- **Do not write past the outline.** If the outline ends at Chapter 20, do not add a Chapter 21 unless the user requests it.
- **Bilingual translation quality**: literary translation by default — prioritize emotional equivalence, not word count matching.
- **Fail gracefully**: if any phase fails, report clearly and suggest the specific sub-skill to retry (`/character-design`, `/language-setting`, `/novel-style`, `/novel-write`, `/novel-export`).

## Composing with Sub-Skills

Each sub-skill can be called independently if you want to skip the full pipeline:

```
User has an outline and wants to start writing directly:
{project}/OUTLINE.md + CHARACTER_BIBLE.md exist → /novel-write "all"

User wants to expand an existing novel with new content:
/novel-fusion "source_novel.docx fusion_outline.md"
→ then /language-setting, /novel-style, /novel-write "all"

User wants to add a new character mid-story:
/character-design "add: [character description] — appears in Ch 8"

User wants to re-export in a different format:
/novel-export "pdf"

User wants to change the style and restyle existing chapters:
/novel-style "literary, 4000字" → then /novel-write "ch01-ch05 --restyle"

User wants to review existing chapters for quality:
/proof-reader "ch01-ch05"

User wants to switch to dialogue-bilingual mode:
/language-setting "dialogue-bilingual zh台词ja"

User wants to restructure the outline for a new source:
/novel-outline "new_source.pdf chapters:15 words:5000"
```

## Typical Timeline

| Phase | Duration | Can proceed autonomously? |
|-------|----------|--------------------------|
| 0. Init + Outline | 5 min | Yes |
| 1. Character Design | 10–20 min | After Gate 1 |
| 2. Language Config | 2 min | After Gate 2 |
| 3. Style Config | 3 min | After Gate 3 |
| 4. Chapter Writing | 5–15 min/chapter (subagent write + proof-reader review) | Yes ✅ |
| 5. Export | 2–5 min | Yes if AUTO_EXPORT=true ✅ |

**Sweet spot**: configure and confirm Gates 1–3 in an afternoon session, run `/novel-write all` overnight, wake up to a finished draft ready to export.
