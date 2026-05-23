---
name: asset-map
description: "Scan, filter, and position images from the PDF outline into chapter locations for epub/docx export. Visually analyzes each image, cross-references with outline text and chapter content, filters out irrelevant images (icons, UI, duplicates), and copies selected images to an export-ready folder with updated IMAGE_MAP.json. Use when user says \"定位图片\", \"图片映射\", \"map assets\", \"assign images\", \"准备图片导出\", \"image placement\", or wants to prepare images for final novel export."
argument-hint: [scope — e.g., "all" / "ch01-ch05" / "scan-only" / "filter-only" / "place"]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob
---

# Asset Map: Image Filtering & Placement for Novel Export

Process scope: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

This skill takes the raw images extracted from the PDF outline (`{project}/assets/`) and produces an export-ready image set with chapter placements. It is a **pre-processing step** before `/novel-export`.

```
{project}/assets/               → 309 raw images (all from PDF)
        ↓  Phase 1: Visual Scan
        ↓  Phase 2: Cross-Reference with Outline
        ↓  Phase 3: Filter (keep only novel-worthy images)
        ↓  Phase 4: Chapter Placement
{project}/assets/export/         → Curated images with chapter assignments
{project}/assets/IMAGE_MAP.json  → Updated with chapter, position, caption, status
{project}/draft/[lang]/ch*.md    → ![caption](path) references inserted
```

## Prerequisites

Required files:

- `{project}/assets/IMAGE_MAP.json` — image inventory from PDF extraction
- `{project}/assets/*.jpeg|*.png` — extracted images
- `{project}/OUTLINE.md` — source outline with `<!-- Page N -->` markers
- `{project}/draft/[lang]/ch*.md` — chapter files (needed for Phase 4 placement)

## Argument Parsing

| Argument                   | Meaning                                                        |
| -------------------------- | -------------------------------------------------------------- |
| `all` / `全部` / _(empty)_ | Full pipeline: scan → filter → place                           |
| `scan-only` / `扫描`       | Phase 1-2 only: analyze images, update IMAGE_MAP               |
| `filter-only` / `筛选`     | Phase 3 only: filter from already-scanned IMAGE_MAP            |
| `place` / `定位`           | Phase 4 only: place already-filtered images into chapters      |
| `ch01-ch05` / `ch08`       | Full pipeline but only for images mapped to specified chapters |
| `reset`                    | Clear all assignments, start fresh                             |
| `report`                   | Show current IMAGE_MAP status without changes                  |

## State Persistence

All state lives in `{project}/assets/IMAGE_MAP.json`. Each image entry:

```json
{
  "file": "{project}/assets/outline_p032_img01.png",
  "source_page": 32,
  "size_kb": 45.2,
  "chapter": "ch05",
  "position": "after_scene_break",
  "position_detail": "after line 142, ノルド arrives at Truth Eden",
  "caption": "Truth Eden — 剣の都の全景",
  "type": "map",
  "status": "selected",
  "scan_notes": "Panoramic illustration of a floating city with sword-shaped towers"
}
```

**Status values:**

- `unscanned` — not yet analyzed (initial state)
- `scanned` — analyzed, type assigned, awaiting filtering
- `rejected` — filtered out (with reason)
- `selected` — approved for export
- `placed` — inserted into chapter markdown

**Type values:**

- `character` — character illustration / portrait
- `map` — world map, location diagram, floor plan
- `scene` — narrative scene illustration (battle, ceremony, etc.)
- `diagram` — concept diagram, relationship chart, timeline
- `cover` — chapter cover / title card
- `icon` — small icon, UI element, logo (typically rejected)
- `duplicate` — near-duplicate of another image (rejected)
- `text_only` — image containing only text, no illustration (typically rejected)
- `unknown` — couldn't classify

## Pipeline

### Phase 1: Visual Scan

Analyze each image to determine its content and type.

**Strategy: batch by size first, then visual analysis.**

#### Step 1a: Pre-filter by size

```python
import json, pathlib

with open("{project}/assets/IMAGE_MAP.json", "r", encoding="utf-8") as f:
    images = json.load(f)

# Size-based triage
for img in images:
    kb = img.get("size_kb", 0)
    if kb < 5:
        img["type"] = "icon"
        img["status"] = "rejected"
        img["scan_notes"] = f"Auto-rejected: too small ({kb:.1f}KB), likely icon/dot"
    elif kb < 10:
        img["type"] = "icon"
        img["status"] = "scanned"
        img["scan_notes"] = f"Flagged: very small ({kb:.1f}KB), likely icon — needs visual confirm"
    else:
        img["status"] = "unscanned"

# Save
with open("{project}/assets/IMAGE_MAP.json", "w", encoding="utf-8") as f:
    json.dump(images, f, ensure_ascii=False, indent=2)
```

#### Step 1b: Visual analysis (batched)

For images with `status: "unscanned"`, read them visually using the Read tool (which supports image files).

**Batch processing**: process images in groups of 5-10, sorted by source_page. For each image:

1. **Read the image** with the Read tool
2. **Describe** the visual content in one sentence
3. **Classify** into a type (character / map / scene / diagram / cover / icon / text_only)
4. **Extract text** if the image contains readable text (character names, location labels, etc.)
5. **Note key identifiers**: character names visible, location names, chapter-relevant keywords

**Update IMAGE_MAP.json** after each batch:

```json
{
  "scan_notes": "Full-body illustration of a dragon-person warrior in red armor, holding a large axe",
  "type": "character",
  "status": "scanned",
  "detected_text": ["オッザニア"],
  "detected_keywords": ["龍人", "赤", "戦士"]
}
```

**Efficiency rules:**

- Skip images already scanned (`status != "unscanned"`)
- If an image is clearly an icon/UI element, mark and move on immediately
- If an image is nearly identical to a previous one (same character, slightly different pose on adjacent pages), mark the lower-quality one as `duplicate`
- Process all images from the same source_page together — they share context

#### Step 1c: Cross-reference with Outline

For each scanned image, find the corresponding outline text using the `source_page`:

```python
import re

with open("{project}/OUTLINE.md", "r", encoding="utf-8") as f:
    outline_text = f.read()

# Split outline by page markers
pages = {}
for match in re.finditer(r'<!-- Page (\d+) -->\n(.*?)(?=<!-- Page \d+ -->|$)', outline_text, re.DOTALL):
    page_num = int(match.group(1))
    pages[page_num] = match.group(2).strip()
```

For each image:

1. Get the outline text from `source_page`
2. Determine which chapter the outline page corresponds to
3. Assign `chapter` field in IMAGE_MAP

**Page-to-chapter mapping** should be inferred from the outline structure (chapter headers, page ranges). Build this mapping once, then apply to all images.

### Phase 2: Outline-to-Chapter Mapping

Build a page→chapter lookup table from the outline:

1. Read `{project}/OUTLINE.md`
2. Find chapter boundaries (look for chapter header patterns: `第N章`, `Chapter N`, `第一章`, etc.)
3. Record which page range each chapter covers
4. For each image, assign `chapter` based on its `source_page`

```python
# Build chapter ranges from outline
chapter_ranges = []  # [(start_page, end_page, chapter_id), ...]
current_chapter = None
current_start = None

for page_num in sorted(pages.keys()):
    text = pages[page_num]
    # Detect chapter headers
    ch_match = re.search(r'第(\d+|[一二三四五六七八九十]+)章', text)
    if ch_match:
        if current_chapter:
            chapter_ranges.append((current_start, page_num - 1, current_chapter))
        # Convert to chNN format
        ch_num = ch_match.group(1)
        # Handle both numeric and kanji chapter numbers
        current_chapter = f"ch{int(ch_num):02d}" if ch_num.isdigit() else convert_kanji_num(ch_num)
        current_start = page_num

# Don't forget the last chapter
if current_chapter:
    chapter_ranges.append((current_start, max(pages.keys()), current_chapter))
```

**Edge cases:**

- Interlude chapters (幕間, interlude): assign as `ch{N}_interlude`
- Prologue / epilogue: assign as `ch00` / `ch{last+1}`
- Images on pages between chapters: assign to the nearest chapter

### Phase 3: Filter

Apply filtering rules to decide which images to keep (`selected`) vs reject (`rejected`).

#### Filtering rules (in priority order)

1. **Size filter**: images < 5KB → reject as `icon`
2. **Type filter**: `icon`, `text_only` → reject (unless text_only contains a unique map/diagram)
3. **Duplicate filter**: if two images from adjacent pages show the same subject, keep the larger/clearer one
4. **Relevance filter**: cross-reference image keywords with chapter text
   - Read the target chapter file
   - Check if the character/location/event shown in the image actually appears in the written chapter
   - If the image shows content NOT covered in our novel draft (e.g., a minor subplot we skipped), reject
5. **Density filter**: aim for 1-3 images per chapter maximum
   - If a chapter has 10+ candidate images, rank by relevance and keep top 3
   - Priority: character first appearances > key scene illustrations > maps > diagrams
6. **Quality filter**: if an image is too small to display well (< 200px on any dimension), reject

**For each rejection, record the reason:**

```json
{
  "status": "rejected",
  "reject_reason": "duplicate of outline_p032_img01 (same character, lower quality)"
}
```

#### Interactive confirmation gate

After filtering, present a summary:

```
Image filtering complete.

Selected: [N] images across [N] chapters
Rejected: [N] images

By chapter:
  ch01: [N] selected (character: 2, scene: 1)
  ch02: [N] selected (map: 1)
  ...

Rejected breakdown:
  icon/UI: [N]
  duplicate: [N]
  text_only: [N]
  not in chapter: [N]
  density cap: [N]

Review? Type "show rejected" to see all rejected images, or "confirm" to proceed.
```

Wait for user confirmation before proceeding to Phase 4.

### Phase 4: Placement

For each `selected` image, determine the exact insertion point in the chapter markdown.

#### Step 4a: Determine position

For each selected image:

1. Read the target chapter file
2. Based on image type, determine placement:

| Image type  | Default position                 | Strategy                                        |
| ----------- | -------------------------------- | ----------------------------------------------- |
| `cover`     | `chapter_start`                  | Before first paragraph                          |
| `character` | First significant appearance     | Find first mention of character name in chapter |
| `map`       | Where location is introduced     | Find first mention of location name             |
| `scene`     | Nearest `---` scene break        | Match scene description to outline context      |
| `diagram`   | Before/after explanatory passage | Find passage that describes the concept         |

1. For `character` and `map` types, search the chapter text for keywords from `detected_text` and `detected_keywords`
2. Record the position as a line number or landmark

#### Step 4b: Copy to export folder

```bash
mkdir -p {project}/assets/export
```

Copy selected images to `{project}/assets/export/` with clean filenames:

```python
import shutil, json, pathlib

with open("{project}/assets/IMAGE_MAP.json", "r", encoding="utf-8") as f:
    images = json.load(f)

export_dir = pathlib.Path("{project}/assets/export")
export_dir.mkdir(parents=True, exist_ok=True)

counter = {}
for img in images:
    if img.get("status") != "selected":
        continue
    ch = img["chapter"]
    counter[ch] = counter.get(ch, 0) + 1
    ext = pathlib.Path(img["file"]).suffix
    # Clean filename: ch05_img01.jpeg
    new_name = f"{ch}_img{counter[ch]:02d}{ext}"
    src = pathlib.Path(img["file"])
    dst = export_dir / new_name
    shutil.copy2(src, dst)
    img["export_file"] = f"{project}/assets/export/{new_name}"
    print(f"  {src.name} → {new_name}")

with open("{project}/assets/IMAGE_MAP.json", "w", encoding="utf-8") as f:
    json.dump(images, f, ensure_ascii=False, indent=2)
```

#### Step 4c: Insert references into chapter markdown

For each language version of each chapter:

1. Read the chapter file
2. Find the insertion point (line number from Step 4a)
3. Insert `![caption](relative_path)` at that position
4. Save the file

```markdown
![オッザニア — 龍人の戦士]({project}/assets/export/ch01_img01.jpeg)
```

**Path handling:**

- For chapter files in `{project}/draft/ja/ch01_ja.md`, the image reference path should be relative or absolute depending on pandoc setup
- Use the path format that `novel-export` expects: `{project}/assets/export/chNN_imgNN.ext`

**Insert both language versions** — the same image goes into both `ja` and `zh` chapter files, but captions are language-specific:

- JA caption: from glossary Japanese name
- ZH caption: from glossary Chinese name

#### Step 4d: Update IMAGE_MAP status

```json
{
  "status": "placed",
  "export_file": "{project}/assets/export/ch05_img01.jpeg",
  "inserted_at": {
    "ja": {"file": "{project}/draft/ja/ch05_ja.md", "line": 42},
    "zh": {"file": "{project}/draft/zh/ch05_zh.md", "line": 44}
  }
}
```

### Phase 5: Report

```
Asset Mapping Complete.

Total images scanned: [N] / 309
Selected for export: [N]
Rejected: [N]
Placed in chapters: [N]

Export folder: {project}/assets/export/ ([N] files, [total_size] MB)

By chapter:
| Chapter | Images | Types            | Status |
| ------- | ------ | ---------------- | ------ |
| ch01    | 2      | character, scene | placed |
| ch02    | 1      | map              | placed |
| ...     |        |                  |        |

Files modified:
- {project}/assets/IMAGE_MAP.json (updated)
- {project}/assets/export/ ([N] images)
- {project}/draft/ja/ch*.md ([N] image references inserted)
- {project}/draft/zh/ch*.md ([N] image references inserted)

Ready for: /novel-export "epub docx"
```

## Incremental Mode

When called with a chapter range (e.g., `ch05-ch08`):

- Only process images whose `source_page` maps to the specified chapters
- Skip images already in `placed` or `rejected` status (unless `--force`)
- Useful for adding images to newly written chapters without re-scanning everything

## Reset Mode

When called with `reset`:

1. Remove all `chapter`, `type`, `status`, `scan_notes` fields from IMAGE_MAP.json
2. Delete `{project}/assets/export/` directory
3. Remove all `![...]({project}/assets/export/...)` lines from chapter files
4. Report: "Asset mappings reset. Run `/asset-map all` to start fresh."

## Key Rules

- **Never delete original images** in `{project}/assets/`. The export folder is a copy.
- **Image density**: 1-10 images per chapter is ideal. More than 10 per chapter degrades reading flow.
- **Large file handling**: if IMAGE_MAP.json exceeds Write limits, use Bash heredoc.
- **Idempotent**: running the skill twice should not create duplicate image references in chapters. Check before inserting.
- **Respect linter**: after modifying markdown files, the linter may adjust formatting. Read files fresh before making sequential edits.
- **User approval required before placement**: Phase 3 filtering results must be confirmed by user before images are inserted into chapter files. This prevents unwanted modifications to the prose.
