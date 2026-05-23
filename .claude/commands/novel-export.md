---
name: novel-export
description: "Compile all chapter drafts and export the novel to one or more output formats: txt, docx (Word), pdf, epub. Handles bilingual novels by exporting separate files per language. Use when user says \"导出小说\", \"export novel\", \"生成epub\", \"生成word\", \"输出文件\", \"打包小说\", or wants to convert the drafted chapters into a final document."
argument-hint: [format(s) — e.g., "docx epub" / "all" / "pdf"]
allowed-tools: Bash(*), Read, Write, Edit, Glob
---

# Novel Export: Compile and Convert to Final Formats

Export the novel to: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

This skill assembles all chapter drafts from `{project}/draft/` into a single document and exports to the requested output format(s). Bilingual novels produce separate files per language.

Supported formats:

| Format | Extension | Tool | Notes |
|--------|-----------|------|-------|
| Plain text | `.txt` | pandoc or sed | Stripped formatting |
| Word document | `.docx` | pandoc | Styled, with TOC |
| PDF | `.pdf` | pandoc + xelatex | CJK-compatible |
| EPUB | `.epub` | pandoc | E-reader ready |

## Prerequisites Check

Before exporting:

1. Check `{project}/draft/` (monolingual) or `{project}/draft/[lang-code]/` (bilingual) for chapter files
2. Check `{project}/settings/LANGUAGE_SETTING.json` for language mode and file naming
3. Check `{project}/NOVEL_STATE.json` for novel title and chapter count
4. Verify at least one chapter file exists
5. **Check for image assets** (see Step 0 below)

```bash
# Check chapter count
ls {project}/draft/ch*.md 2>/dev/null | wc -l
```

If no chapters found: "⚠️ No chapter files found in `{project}/draft/`. Run `/novel-write` first."

If some chapters are missing (gaps in sequence): warn and ask whether to export what exists or wait.

### Step 0: Image Asset Pre-Processing (Automatic)

Before assembling chapters, check if there are image assets that need to be mapped and placed into chapter files.

**Trigger condition**: `{project}/assets/` directory exists AND contains image files (`.jpeg`, `.png`, `.jpg`).

```python
import pathlib, json

assets_dir = pathlib.Path("{project}/assets")
has_assets = assets_dir.exists() and any(assets_dir.glob("*.jpeg")) or any(assets_dir.glob("*.png"))

if has_assets:
    image_map_path = assets_dir / "IMAGE_MAP.json"
    if image_map_path.exists():
        image_map = json.loads(image_map_path.read_text(encoding="utf-8"))
        placed = [i for i in image_map if i.get("status") == "placed"]
        selected = [i for i in image_map if i.get("status") == "selected"]
        total = len(image_map)
        print(f"Image assets: {total} images, {len(placed)} placed, {len(selected)} selected")
    else:
        placed = []
        selected = []
        print("Image assets found but no IMAGE_MAP.json — needs full scan")
```

**Decision tree:**

| Condition | Action |
|-----------|--------|
| No `{project}/assets/` dir | Skip — no images to process |
| `IMAGE_MAP.json` missing | Run `/asset-map all` (full pipeline: scan → filter → place) |
| All selected images are `placed` | Skip — images already in chapter files, ready for export |
| Selected images exist but NOT `placed` | Run `/asset-map place` (Phase 4 only: copy + insert into chapters) |
| No images are `selected` yet | Run `/asset-map all` (full pipeline) |

**Important**: The asset-map process requires **user confirmation** before inserting images into chapter markdown files (Phase 3→4 gate). If running automatically from export, present the filtering summary and wait for user approval before proceeding. This prevents unwanted modifications to the prose.

After asset-map completes, the chapter files will contain `![caption](path)` references that pandoc embeds automatically in DOCX, PDF, and EPUB outputs. TXT export ignores image references.

## Argument Parsing

Parse `$ARGUMENTS` for requested formats:

| Keyword | Format(s) |
|---------|-----------|
| `txt` / `text` / `纯文本` | txt |
| `docx` / `word` / `Word` / `doc` | docx |
| `pdf` / `PDF` | pdf |
| `epub` / `EPUB` / `电子书` | epub |
| `all` / `全部` | txt + docx + pdf + epub |
| _(nothing)_ | epub + docx (default) |

Multiple formats can be specified: `"docx pdf epub"` → export all three.

## Workflow

### Step 1: Dependency Check

```bash
# Check pandoc
pandoc --version 2>/dev/null && echo "pandoc: OK" || echo "pandoc: MISSING"

# Check PDF engines (for PDF export)
xelatex --version 2>/dev/null && echo "xelatex: OK" || \
  pdflatex --version 2>/dev/null && echo "pdflatex: OK" || \
  wkhtmltopdf --version 2>/dev/null && echo "wkhtmltopdf: OK" || \
  echo "PDF engine: NONE (PDF export will be skipped)"
```

**Check image assets** (after Step 0 asset-map pre-processing, if applicable):

```bash
# Verify all images referenced in chapter files actually exist
python3 - <<'PY'
import re, pathlib
draft = pathlib.Path("{project}/draft")
missing = []
found = 0
for md_file in sorted(draft.rglob("ch*.md")):
    text = md_file.read_text(encoding="utf-8")
    for m in re.finditer(r'!\[.*?\]\(({project}/assets/[^)]+)\)', text):
        img_path = pathlib.Path(m.group(1))
        if not img_path.exists():
            missing.append(f"{md_file.name}: {img_path}")
        else:
            found += 1
if missing:
    print(f"MISSING IMAGES ({len(missing)}):")
    for m in missing: print(" ", m)
elif found > 0:
    print(f"All {found} image references verified OK")
else:
    print("No image references in chapter files (text-only export)")
PY
```

If any images are missing, warn and skip embedding them (do not abort the export). Images in `{project}/assets/export/` are the curated copies produced by `/asset-map`; original images stay in `{project}/assets/`.

**Image support by format:**
| Format | Inline images | Notes |
|--------|--------------|-------|
| DOCX | ✅ embedded | pandoc auto-embeds `![](path)` references |
| PDF (xelatex) | ✅ embedded | rendered via `\includegraphics` |
| EPUB | ✅ embedded | images bundled inside the `.epub` zip |
| TXT | ✗ ignored | image references stripped from plain text output |

If pandoc is missing:
- For txt export: use bash sed fallback
- For all other formats: inform user and skip those formats
  "⚠️ `pandoc` not found. Installing: `winget install JohnMacFarlane.Pandoc` (Windows) / `brew install pandoc` (macOS) / `apt install pandoc` (Linux). Proceeding with txt-only export."

If PDF requested and no engine found:
- Skip PDF, warn: "⚠️ No LaTeX/wkhtmltopdf engine for PDF. Export other formats and retry after installing xelatex."

### Step 2: Load Metadata

Read from `{project}/NOVEL_STATE.json` and `{project}/settings/LANGUAGE_SETTING.json`:

```json
{
  "title": "[Novel title]",
  "author": "[Author or 'Anonymous']",
  "primary_language": "zh",
  "bilingual": true,
  "secondary_language": "en",
  "output_filenames": {
    "primary": "novel_zh",
    "secondary": "novel_en"
  }
}
```

If title is missing from the state file, ask: "What is the novel's title? (Used for output filenames and EPUB metadata)"

### Step 3: Assemble Chapter Files

**Monolingual:**

```bash
python3 - <<'PY'
import re, pathlib

draft = pathlib.Path("{project}/draft")
chapters = sorted(
    [p for p in draft.glob("ch*.md") if p.is_file()],
    key=lambda p: int(re.search(r"\d+", p.stem).group())
)
if not chapters:
    print("ERROR: no chapter files found")
    raise SystemExit(1)

separator = "\n\n---\n\n"
full = separator.join(p.read_text(encoding="utf-8") for p in chapters)
out = draft / "FULL_NOVEL.md"
out.write_text(full, encoding="utf-8")
total_words = sum(len(p.read_text(encoding="utf-8").split()) for p in chapters)
print(f"Assembled {len(chapters)} chapters | ~{total_words} words → {out}")
PY
```

**Bilingual:** run the same script twice, once for each language directory:

```bash
# Primary language
python3 -c "
import re, pathlib
lang = 'zh'  # replace with actual primary lang code
draft = pathlib.Path(f'{project}/draft/{lang}')
chapters = sorted(draft.glob('ch*.md'), key=lambda p: int(__import__('re').search(r'\d+', p.stem).group()))
full = '\n\n---\n\n'.join(p.read_text(encoding='utf-8') for p in chapters)
(draft.parent / f'FULL_NOVEL_{lang}.md').write_text(full, encoding='utf-8')
print(f'{lang}: {len(chapters)} chapters assembled')
"

# Secondary language
python3 -c "
import re, pathlib
lang = 'en'  # replace with actual secondary lang code
draft = pathlib.Path(f'{project}/draft/{lang}')
chapters = sorted(draft.glob('ch*.md'), key=lambda p: int(__import__('re').search(r'\d+', p.stem).group()))
full = '\n\n---\n\n'.join(p.read_text(encoding='utf-8') for p in chapters)
(draft.parent / f'FULL_NOVEL_{lang}.md').write_text(full, encoding='utf-8')
print(f'{lang}: {len(chapters)} chapters assembled')
"
```

### Step 4: Run Export Commands

Read `../shared-references/novel-output-formats.md` for the exact pandoc commands for each format.

Apply the commands with the novel's actual metadata:

**For each requested format, run the appropriate pandoc command from the shared reference**, substituting:
- Source file: `{project}/draft/FULL_NOVEL.md` (or `FULL_NOVEL_[lang].md` for bilingual)
- Output file: `{project}/output/[title_slugified].[ext]` (or `[title_slugified]_[lang].[ext]` for bilingual)
- Title metadata: novel title from state
- Author metadata: from state (or "Anonymous")
- Language code: from LANGUAGE_SETTING.json

Example resolved command for EPUB (Chinese novel):
```bash
pandoc {project}/draft/FULL_NOVEL.md \
  -o "{project}/output/novel_zh.epub" \
  --toc --toc-depth=1 \
  --metadata title="[Title]" \
  --metadata author="[Author]" \
  --metadata lang="zh"
```

Create the output directory first:
```bash
mkdir -p {project}/output
```

**Run exports sequentially** (each format is independent but let one finish before starting the next to avoid I/O conflicts).

### Step 5: Post-Export Verification

For each generated file:

```bash
# Verify file exists and has content
ls -lh {project}/output/novel*.* 2>/dev/null
```

- TXT: check file size > 1KB
- DOCX: check file size > 10KB
- PDF: check file size > 50KB; count pages with `pdfinfo {project}/output/novel.pdf | grep Pages`
- EPUB: check file size > 5KB; optionally validate with `epubcheck {project}/output/novel.epub` if available

If a file is suspiciously small (< 1KB), re-run that format's export with verbose output to diagnose the issue.

### Step 6: Deferred Translation (Bilingual, post-writing mode)

If `LANGUAGE_SETTING.json` has `"translation_timing": "post-writing"` and this is the first export:

Check `{project}/NOVEL_STATE.json` for chapters missing their secondary-language translation:

```python
# Chapters present in primary but not secondary language
primary_chapters = set(p.stem for p in Path("{project}/draft/zh").glob("ch*.md"))
secondary_chapters = set(p.stem for p in Path("{project}/draft/en").glob("ch*.md"))
missing = sorted(primary_chapters - secondary_chapters)
```

If any chapters lack a translation:
1. Translate each missing chapter using the style from `LANGUAGE_SETTING.json`
2. Save to `{project}/draft/[secondary-code]/chNN.md`
3. After all translations are done, re-run Step 3 to reassemble the secondary-language full novel
4. Then export the secondary-language files

### Step 7: Export Report

```
Export complete.

Novel: [Title]
Author: [Author]
Total chapters: [N] | Total words: ~[N] ([primary lang])
[Bilingual: [N] words in [secondary lang]]

Output files:
```

Present a table:

| Format | File | Size | Status |
|--------|------|------|--------|
| TXT | {project}/output/novel.txt | 150 KB | ✅ |
| DOCX | {project}/output/novel.docx | 380 KB | ✅ |
| PDF | {project}/output/novel.pdf | 1.2 MB | ✅ (42 pages) |
| EPUB | {project}/output/novel.epub | 210 KB | ✅ |

```
[For bilingual:]
Primary ([lang]): {project}/output/novel_zh.*
Secondary ([lang]): {project}/output/novel_en.*
```

## Partial Export

To export specific chapters only (e.g., for a preview):

```bash
/novel-export "ch01-ch05 epub"
```

Assemble only the specified chapters, export to `{project}/output/preview_ch01-ch05.epub`.

## Cover Image

If `{project}/assets/cover.jpg` (or `.png`) exists, it is automatically used as the EPUB cover.

To generate a simple placeholder cover (text-only, no external tool required):
```python
# Generate minimal cover HTML → convert to image is complex; instead note it as TODO
```

If no cover image exists: note in the report "No cover image found. Add `{project}/assets/cover.jpg` for EPUB cover."

## Key Rules

- **Large file handling**: If Write fails, retry with Bash heredoc. Do not ask for permission.
- **Never overwrite without backing up**: if `{project}/output/` already has files from a previous export, back them up to `{project}/output/backup-[timestamp]/` before re-exporting.
- **Export is read-only** with respect to the chapter drafts — never modify `{project}/draft/` files during export.
- **Missing secondary language chapters**: if translation_timing is `post-writing`, do the translations as part of this export rather than blocking.
- **Graceful degradation**: if one format fails, complete the others and report the failure clearly. Do not abort the entire export.
