---
name: novel-writing-pipeline
description: "Full novel-writing pipeline: outline → character design → style/language config → chapter drafting → export. Orchestrates /character-design, /language-setting, /novel-style, /novel-write, and /novel-export into a single end-to-end workflow. Use when user says \"写小说\", \"novel pipeline\", \"小说全流程\", \"帮我写本小说\", \"end-to-end novel\", or wants to go from an outline to a finished document."
argument-hint: [outline-file-or-description — optional: language, style, output-format]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, Skill
---

# Novel Writing Pipeline: Outline → Finished Novel

Full novel pipeline for: **$ARGUMENTS**

## Constants

- **AUTO_PROCEED = false** — When `true`, automatically moves through Phase 1–3 (configuration) without waiting for confirmation. When `false` (default), pauses at each gate for user approval. Recommended: keep `false` for first-time runs so you can review the character bible and style settings before writing begins.
- **LANGUAGE_MODE = "monolingual"** — `monolingual` (default) or `bilingual`. Override: `bilingual: true` or specify two languages (e.g., `中英双语`).
- **PRIMARY_LANGUAGE = "zh"** — Language the novel is written in. Detected from arguments or outline language. Default: Chinese (`zh`).
- **SECONDARY_LANGUAGE = "en"** — Secondary language for bilingual mode. Default: English (`en`). Only used when `LANGUAGE_MODE = bilingual`.
- **GENRE = "auto"** — Genre/style. Auto-detected from outline if not specified. Override: `genre: romance / thriller / fantasy / literary / light`.
- **CHAPTER_WORD_COUNT = 3000** — Target words per chapter. Override: `每章N字` or `N words per chapter`.
- **OUTPUT_FORMAT = "epub docx"** — Output formats. Options: `txt`, `docx`, `pdf`, `epub`, `all`. Default: epub + docx.
- **AUTO_EXPORT = false** — When `true`, automatically exports after all chapters are written. When `false` (default), user calls `/novel-export` manually.
- **CHAPTER_RANGE = "all"** — Which chapters to write in the writing phase. Default: all chapters from the outline.

> Override any constant via argument: `/novel-writing-pipeline "outline.md — bilingual: zh/en, genre: romance, 每章3000字, output: epub pdf, auto_proceed: true"`

## Overview

This pipeline chains five sub-skills into a single novel-writing lifecycle:

```
/character-design  →  /language-setting  →  /novel-style  →  /novel-write  →  /novel-export
     (Phase 1)             (Phase 2)            (Phase 3)        (Phase 4)       (Phase 5)
  Character Bible       Language Config       Style Config     Chapter Drafts   Final Output
```

**Persistent memory files** (live in `novel/` throughout the pipeline):
- `novel/OUTLINE.md` — story structure (input, never modified)
- `novel/characters/CHARACTER_BIBLE.md` — living character reference (grows during writing)
- `novel/settings/LANGUAGE_SETTING.json` — language configuration
- `novel/settings/STYLE_SETTING.json` — style configuration
- `novel/draft/` — chapter files (primary language)
- `novel/draft/[lang-code]/` — chapter files by language (bilingual)
- `novel/NOVEL_STATE.json` — pipeline progress state
- `novel/output/` — final exported files

## State Persistence

Persist pipeline state to `novel/NOVEL_STATE.json` after each phase:

```json
{
  "title": "[Novel title]",
  "phase": 3,
  "auto_proceed": false,
  "language_mode": "bilingual",
  "primary_language": "zh",
  "secondary_language": "en",
  "genre": "romance",
  "chapter_word_count": 3000,
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

1. **Outline source** — detect a file path (`.md`, `.txt`) or inline text
2. **Language settings** — detect bilingual/monolingual, primary/secondary languages
3. **Genre/style hints** — extract if mentioned
4. **Chapter word count** — extract if specified
5. **Output format** — extract if specified
6. **Constant overrides** — update constants accordingly

**Initialize project directory:**

```bash
mkdir -p novel/characters
mkdir -p novel/settings
mkdir -p novel/draft
mkdir -p novel/output
mkdir -p novel/assets
```

**Save the outline:**

If the outline was provided inline (not a file path), write it to `novel/OUTLINE.md`.
If it was provided as a file path, copy to `novel/OUTLINE.md` (keep the original).
If no outline provided: ask immediately. Do not proceed without an outline.

```
No outline provided. Please share the story outline (paste text, or provide a file path).
```

**Extract images from PDF outline (if applicable):**

If the outline source is a `.pdf` file, run the image extraction step before proceeding:

```python
# Requires: pip install pymupdf
import fitz, json, pathlib

pdf_path = "XXX.pdf"   # replace with actual path
assets = pathlib.Path("novel/assets")
assets.mkdir(parents=True, exist_ok=True)

doc = fitz.open(pdf_path)
image_map = []   # will be saved as IMAGE_MAP.json

for page_num in range(len(doc)):
    page = doc[page_num]
    images = page.get_images(full=True)
    for img_idx, img in enumerate(images):
        xref = img[0]
        base_img = doc.extract_image(xref)
        img_ext = base_img["ext"]            # png, jpeg, etc.
        img_data = base_img["image"]
        filename = f"outline_p{page_num+1:02d}_img{img_idx+1:02d}.{img_ext}"
        out_path = assets / filename
        out_path.write_bytes(img_data)
        image_map.append({
            "file": f"novel/assets/{filename}",
            "source_page": page_num + 1,
            "chapter": None,        # to be filled by user or inferred from outline
            "position": "after_scene_break",  # default placement
            "caption": ""           # optional caption
        })

(assets / "IMAGE_MAP.json").write_text(
    json.dumps(image_map, ensure_ascii=False, indent=2), encoding="utf-8"
)
print(f"Extracted {len(image_map)} images → novel/assets/IMAGE_MAP.json")
```

**Fallback if pymupdf is not available:**
```bash
# pdfimages (poppler-utils) — saves as PPM/PNG
pdfimages -png XXX.pdf novel/assets/outline
# Then manually rename: outline-000.png → outline_p01_img01.png
```

**After extraction**, read each extracted image visually (the Read tool supports image files) and:
1. Describe the image content in one line
2. Infer which chapter it most likely belongs to based on the outline context
3. Fill in `"chapter": "ch01"` and `"caption": "[description]"` in `IMAGE_MAP.json`

Present to the user for confirmation:

```
Extracted [N] images from outline PDF.

IMAGE_MAP.json preview:
- outline_p03_img01.png → Ch01 "地图：星语之塔位置图" [地图]
- outline_p07_img01.png → Ch03 "艾莲娜·瓦恩 人物立绘" [人物图]
- outline_p12_img01.png → Ch07 "虚构粒子结构示意图" [设定图]

Confirm chapter assignments? (or adjust before writing begins)
```

If the user adjusts assignments, update `IMAGE_MAP.json` accordingly.

**Note:** TXT export ignores images entirely. Images are only embedded in DOCX, PDF, and EPUB outputs (handled in `/novel-export`).

**Present pipeline overview:**

```
Novel Writing Pipeline initialized.

Outline: novel/OUTLINE.md ([N] chapters detected)
Title: [detected title or "Untitled"]

Pipeline configuration:
- Language: [Monolingual zh / Bilingual zh→en]
- Genre: [detected or "to be configured"]
- Chapter length: [N] 字 target
- Output: [formats]

Phases:
1. Character Design   → CHARACTER_BIBLE.md
2. Language Setting   → LANGUAGE_SETTING.json
3. Style Config       → STYLE_SETTING.json
4. Chapter Writing    → novel/draft/ ([N] chapters)
5. Export             → novel/output/ ([formats])

Proceed? (or adjust any setting before starting)
```

If AUTO_PROCEED=false: wait for user confirmation.
If AUTO_PROCEED=true: proceed after presenting the overview.

### Phase 1: Character Design

Invoke `/character-design`:

```
/character-design "novel/OUTLINE.md"
```

This reads the outline and produces `novel/characters/CHARACTER_BIBLE.md` with full profiles for all major/supporting characters.

**🚦 Gate 1 — Character Review:**

After `CHARACTER_BIBLE.md` is generated, present a summary:

```
Character Bible complete.

Characters designed:
- Major: [N] — [names]
- Supporting: [N] — [names]
- Minor: [N]

Key relationships:
- [Brief summary]

Please review novel/characters/CHARACTER_BIBLE.md.
Any characters to add, remove, or adjust?
```

**If AUTO_PROCEED=false:** Wait for user response. Options:
- **"OK" / "go"** → proceed to Phase 2
- **Add a character** → call `/character-design "add: [description]"`, update the bible
- **Modify a character** → edit the bible directly or describe changes
- **"stop"** → save state, pipeline pauses here

**If AUTO_PROCEED=true:** Present summary, proceed immediately.

**State**: Write `NOVEL_STATE.json` with `phase: 1`.

### Phase 2: Language Configuration

Invoke `/language-setting`:

```
/language-setting "[PRIMARY_LANGUAGE] [— SECONDARY_LANGUAGE if bilingual]"
```

This produces `novel/settings/LANGUAGE_SETTING.json` and creates draft directories.

**🚦 Gate 2 — Language Confirmation:**

```
Language configuration:
- Mode: [Monolingual / Bilingual]
- Primary: [language name]
[- Secondary: [language name] (translation: [style], [timing])]

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

This produces `novel/settings/STYLE_SETTING.json` and `novel/settings/CHAPTER_TEMPLATE.md`.

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

This writes all chapters from the outline sequentially:
- Reads each chapter's outline beats
- Maintains character consistency against `CHARACTER_BIBLE.md`
- Adds new characters to the bible as they appear
- Applies style settings from `STYLE_SETTING.json`
- Handles bilingual translation if configured
- Saves each chapter to `novel/draft/chNN.md`

**During writing, the pipeline updates `NOVEL_STATE.json` after each chapter.**

Progress updates (presented after each chapter):
```
Chapter [N]/[total] written — [N] 字
[New character added: [name] (stub created)]
[Translation: complete]
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
Draft location: novel/draft/

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
| novel/OUTLINE.md | Source outline |
| novel/characters/CHARACTER_BIBLE.md | [N] characters profiled |
| novel/settings/STYLE_SETTING.json | Genre: [genre], [N] 字/chapter |
| novel/settings/LANGUAGE_SETTING.json | [Mode], Primary: [lang] |
| novel/draft/chNN.md (×N) | [N] chapters, ~[total] words |
| novel/output/novel.[ext] | [formats] |

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
- **Manifest**: log every output file to `novel/MANIFEST.md`

```bash
# MANIFEST.md format
| Timestamp | Skill | File | Description |
|-----------|-------|------|-------------|
| [time] | /character-design | novel/characters/CHARACTER_BIBLE.md | [N] characters |
| [time] | /novel-write | novel/draft/ch01.md | Chapter 1, [N] words |
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
novel/OUTLINE.md + CHARACTER_BIBLE.md exist → /novel-write "all"

User wants to add a new character mid-story:
/character-design "add: [character description] — appears in Ch 8"

User wants to re-export in a different format:
/novel-export "pdf"

User wants to change the style and restyle existing chapters:
/novel-style "literary, 4000字" → then /novel-write "ch01-ch05 --restyle"
```

## Typical Timeline

| Phase | Duration | Can proceed autonomously? |
|-------|----------|--------------------------|
| 0. Init + Outline | 5 min | Yes |
| 1. Character Design | 10–20 min | After Gate 1 |
| 2. Language Config | 2 min | After Gate 2 |
| 3. Style Config | 3 min | After Gate 3 |
| 4. Chapter Writing | 5–15 min/chapter | Yes ✅ |
| 5. Export | 2–5 min | Yes if AUTO_EXPORT=true ✅ |

**Sweet spot**: configure and confirm Gates 1–3 in an afternoon session, run `/novel-write all` overnight, wake up to a finished draft ready to export.
