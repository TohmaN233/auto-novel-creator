---
name: novel-outline
description: "Convert a raw outline or source material (PDF, text, markdown) into a structured OUTLINE.md optimized for chapter-by-chapter writing. Also creates CHARACTER_BIBLE.md if not already present. Use when user says \"整理大纲\", \"结构化大纲\", \"outline prep\", \"prepare outline\", \"转换大纲\", \"分章\", or provides raw source material for a new novel project."
argument-hint: [source-file-path — chapters:N words-per-chapter:N]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent
---

# Novel Outline: Raw Source → Structured Writing Outline

Process: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

This skill transforms raw source material — a PDF, text file, unstructured notes, or any narrative outline — into a structured `OUTLINE.md` that the writing workflow (`/novel-write`) can consume directly. It also creates `CHARACTER_BIBLE.md` if one doesn't already exist, since character extraction is a natural byproduct of outline analysis.

**Input**: any-format source material + chapter count + word-count target
**Output**: `{project}/OUTLINE.md` (structured) + `{project}/characters/CHARACTER_BIBLE.md` (if absent)

If the user already has a well-structured OUTLINE.md and/or CHARACTER_BIBLE.md, this step can be skipped entirely — `/novel-write` just needs the files to exist.

## Argument Parsing

Extract from `$ARGUMENTS`:

| Parameter | Detection | Default |
|-----------|-----------|---------|
| **Source file** | File path (`.pdf`, `.md`, `.txt`, `.docx`) or "inline" if text is pasted directly | Required — ask if missing |
| **Chapter count** | `chapters:N` / `N章` / `分N章` / `N chapters` | Auto-detect from source structure |
| **Words per chapter** | `words:N` / `每章N字` / `N字/章` / `N words` | 3000 (or from existing STYLE_SETTING.json) |
| **Skip character bible** | `no-bible` / `跳过角色` / `skip-characters` | Create bible if `CHARACTER_BIBLE.md` doesn't exist |

Examples:
```
/novel-outline "my_outline.pdf chapters:12 words:8000"
/novel-outline "outline.txt 分20章 每章3000字"
/novel-outline "raw_notes.md chapters:8"        ← auto word count
/novel-outline "story.pdf"                       ← auto chapter count + word count
```

## Workflow

### Step 0: Environment Setup

```bash
mkdir -p {project}/characters
mkdir -p {project}/settings
mkdir -p {project}/draft
mkdir -p {project}/assets
```

### Step 1: Read Source Material

**By file type:**

| Type | Method |
|------|--------|
| `.pdf` | Extract text with pymupdf (`fitz`). Also extract images to `{project}/assets/` and create `IMAGE_MAP.json` stubs. |
| `.md` / `.txt` | Read directly |
| `.docx` | Convert with `python-docx` or `pandoc -t markdown` |
| Inline text | Accept from `$ARGUMENTS` or user message |

**For PDF extraction:**
```python
import fitz, json, pathlib

pdf_path = "[source path]"
doc = fitz.open(pdf_path)

full_text = []
image_map = []
assets = pathlib.Path("{project}/assets")
assets.mkdir(parents=True, exist_ok=True)

for page_num in range(len(doc)):
    page = doc[page_num]
    full_text.append(f"<!-- Page {page_num+1} -->\n{page.get_text()}")
    
    for img_idx, img in enumerate(page.get_images(full=True)):
        xref = img[0]
        base_img = doc.extract_image(xref)
        filename = f"outline_p{page_num+1:03d}_img{img_idx+1:02d}.{base_img['ext']}"
        (assets / filename).write_bytes(base_img["image"])
        image_map.append({
            "file": f"{project}/assets/{filename}",
            "source_page": page_num + 1,
            "chapter": None,
            "position": "after_scene_break",
            "caption": ""
        })

(assets / "IMAGE_MAP.json").write_text(
    json.dumps(image_map, ensure_ascii=False, indent=2), encoding="utf-8"
)
```

Save the raw extracted text as a working copy (this is NOT the final OUTLINE.md):
```
{project}/OUTLINE_RAW.md
```

### Step 2: Analyze Source Material

Read the full source text and identify:

**2a: Story-level analysis**
- **Title** — detect from heading, cover page, or metadata
- **Genre** — infer from content (fantasy, sci-fi, romance, thriller, literary, etc.)
- **Setting** — world, time period, key locations
- **Premise** — 2-3 sentence story summary
- **Central conflict** — the main dramatic question
- **Thematic threads** — recurring themes, motifs

**2b: Plot structure analysis**
- Identify all major plot events in chronological order
- Identify narrative arcs (setup → confrontation → resolution)
- Note existing chapter/section breaks in the source material
- Identify act structure if detectable (3-act, 4-act, kishotenketsu, etc.)

**2c: Character extraction**
- List every named character
- Classify: major (protagonist, antagonist) / supporting / minor
- Extract any appearance descriptions, personality traits, speech patterns
- Map relationships between characters
- Note first appearance (page/section in source)

**2d: Chapter division strategy**

If chapter count is specified by user, use it. If auto-detecting:

1. Check if source material has explicit chapter/section markers → use those
2. If not, divide based on:
   - Natural plot-event clusters (each chapter = one dramatic unit)
   - Word count targets (total estimated words ÷ target per chapter)
   - Narrative pacing (opening chapters shorter/faster, climax chapters longer)
   - POV transitions (if multiple POVs, chapter breaks at POV switches)

**Division principles:**
- Each chapter must have a clear dramatic question or goal
- Each chapter should end with a hook or shift that pulls into the next
- Avoid splitting a single scene across chapters unless the split itself is dramatic
- Cluster related events — don't spread a battle across 5 chapters of 1000 words each; consolidate
- The first chapter must establish the world and protagonist efficiently
- The final chapter must resolve the central conflict and provide denouement

Present the proposed division to the user:
```
Source material analyzed:
- Title: [detected title]
- Genre: [genre]
- Total source length: ~[N] characters
- Named characters found: [N] (major: [N], supporting: [N], minor: [N])
- Plot events identified: [N]

Proposed chapter division: [N] chapters, ~[word target] words each

Chapter   | Title (working)          | Key Events                    | POV
----------|--------------------------|-------------------------------|----------
Ch01      | [suggested title]        | [1-line summary]              | [character]
Ch02      | [suggested title]        | [1-line summary]              | [character]
...       | ...                      | ...                           | ...
Ch[N]     | [suggested title]        | [1-line summary]              | [character]

Confirm? (or adjust chapter count / division points)
```

Wait for user confirmation or adjustment.

### Step 3: Generate Structured OUTLINE.md

After confirmation, produce the structured outline. This is the primary deliverable.

**Target format:**

```markdown
# [Novel Title]

> Source: [source file] | Chapters: [N] | Target: ~[N] words/chapter
> Generated: [ISO timestamp]

## Story Overview

**Premise**: [2-3 sentences]
**Genre**: [genre]
**Setting**: [world, time, key locations]
**Central Conflict**: [the dramatic question driving the story]
**Thematic Threads**: [key themes]

## Act Structure

| Act | Chapters | Purpose |
|-----|----------|---------|
| I — Setup | Ch01–Ch[N] | [establish world, characters, inciting incident] |
| II — Confrontation | Ch[N]–Ch[N] | [rising action, complications, midpoint shift] |
| III — Resolution | Ch[N]–Ch[N] | [climax, falling action, resolution] |

---

## Chapter 1: [Title]

**POV**: [character name]
**Setting**: [location(s), time of day/story-day]
**Characters**: [list of named characters appearing]
**Word target**: ~[N]

### Plot Beats
1. [Specific scene or event — actionable, not vague. "Nord arrives at the council hall and reports P-gauge readings" not "Introduction happens"]
2. [Beat 2 — each beat is one scene or one dramatic unit within a scene]
3. [Beat 3]
4. [Beat N — typically 3-6 beats per chapter]

### Emotional Arc
[POV character's emotional journey: starts feeling X → event Y shifts them to Z → chapter ends with them feeling W]

### Key Revelations
- [What the reader learns that they didn't know before]
- [What specific characters learn — note which character]

### New Characters Introduced
- [Name] — [role] — [one-line description from source]

### Forward Hook
[The unresolved tension or question that pulls the reader into the next chapter]

### Source Reference
[Page numbers or section markers from the raw source that this chapter draws from]

---

## Chapter 2: [Title]
...

---

## Appendix: Unplaced Material

[Any source material that didn't fit naturally into any chapter — world-building details, side stories, background lore that might be woven in as exposition or saved for future use]
```

**Quality standards for beats:**

BAD beats (too vague — the writer subagent can't act on these):
```
1. The team assembles
2. They discuss the plan
3. Something goes wrong
4. They resolve it
```

GOOD beats (specific, actionable, scene-level):
```
1. Nord presents P-gauge readings at the All-Species Council — measurements show the inverted continent's energy matches no known database
2. The dragon clan leader reports 30% of defensive forces destroyed in 3 days — proposes forming a six-member investigation team
3. Six representatives are nominated (one per species) — debate over selection criteria, particularly whether to include the poet warrior whose loyalties are questioned
4. Nord accepts the captain role reluctantly — inner conflict: his analytical nature vs. the political weight of leading a cross-species mission
```

Each beat should contain:
- **WHO** does/experiences it
- **WHAT** happens (concrete action or event)
- **WHY** it matters (emotional or plot significance)

### Step 4: Generate CHARACTER_BIBLE.md (if needed)

Check if `{project}/characters/CHARACTER_BIBLE.md` already exists.

**If it exists**: skip this step. Report: "CHARACTER_BIBLE.md already exists — skipping character generation. Run /character-design to update."

**If it doesn't exist**: generate it from the character data extracted in Step 2c.

Use the same format as `/character-design` produces:

For each **major** and **supporting** character, create a full profile:
- Identity (name, age, species/background)
- Appearance (physical anchors — from source descriptions)
- Personality Core (3 traits)
- Speech Fingerprint (vocabulary, sentence style, pet phrases, avoidance words, emotional register)
- Backstory (story-relevant only)
- Relationships (table)
- Story Arc (goal, internal conflict, key turning points by chapter)
- Knowledge State (what they know at story start)

For **minor** characters, create a summary table:

```markdown
## Minor Characters

| Name | Role | First Appears | Description | Notes |
|------|------|---------------|-------------|-------|
```

**Speech fingerprint sourcing**: if the raw source contains dialogue samples for a character, analyze those samples to derive:
- Register (formal/casual/archaic/dialect)
- Sentence length tendency
- Verbal tics or catchphrases
- Emotional expression patterns

If the source lacks dialogue samples, infer from role/personality and mark as `[inferred — refine after Ch01]`.

Save to `{project}/characters/CHARACTER_BIBLE.md`.

### Step 5: Image Mapping (if PDF source)

If images were extracted in Step 1:

1. Read each extracted image visually
2. Based on the now-structured chapter outline, assign each image to a chapter
3. Set appropriate `position` and `caption` in `IMAGE_MAP.json`
4. Present assignments to user for confirmation

### Step 6: Update State

Write or update `{project}/NOVEL_STATE.json`:

```json
{
  "title": "[Novel title]",
  "phase": 0,
  "total_chapters": [N],
  "chapter_word_count_target": [N],
  "chapters_written": 0,
  "status": "outline_complete",
  "outline_source": "[source file]",
  "outline_pages": [N],
  "outline_chars": [N],
  "images_extracted": [N],
  "characters_created": true,
  "timestamp": "[ISO timestamp]"
}
```

### Step 7: Report

```
Outline structured successfully.

Title: [Novel title]
Chapters: [N] (target: ~[word count] words each)
Characters extracted: [N] major, [N] supporting, [N] minor
[Images: [N] extracted, [N] assigned to chapters]

Files created:
- {project}/OUTLINE.md              ← structured writing outline
- {project}/OUTLINE_RAW.md          ← original source text (reference)
[- {project}/characters/CHARACTER_BIBLE.md  ← [N] character profiles]
[- {project}/assets/IMAGE_MAP.json  ← [N] images mapped]

Next steps:
- Review {project}/OUTLINE.md — adjust any chapter divisions or beats
- Review CHARACTER_BIBLE.md — refine speech fingerprints, add details
- /language-setting  → configure language mode
- /novel-style       → configure writing style
- /novel-write       → start writing
- Or: /novel-writing-pipeline to continue the full pipeline
```

## Handling Edge Cases

### Source material is too short for requested chapter count
If the source has ~5000 characters but user wants 20 chapters: warn that each chapter would have very few beats. Suggest a smaller chapter count or note that the writer will need to expand significantly.

```
⚠️ Source material (~5000 chars) is thin for 20 chapters.
Each chapter would average ~250 chars of source beats — heavy expansion needed.
Recommended: [N] chapters, or confirm 20 with the understanding that the writer will elaborate substantially.
```

### Source material has no clear narrative structure
If the source is a world-building document, a setting bible, or scattered notes without a plot:

1. Extract all available information (world, characters, concepts)
2. Ask the user for plot direction: "The source material describes a world but not a specific story. What's the central conflict / protagonist's journey?"
3. Once given a direction, propose a chapter structure that uses the source as the world foundation

### Source material is in a different language than the writing language
The outline should be structured in the **writing language** (from LANGUAGE_SETTING.json if it exists, or from user specification). If the source is in Japanese but the writing language is Chinese, translate the outline content during structuring. Proper nouns should be preserved in both forms:

```markdown
### Plot Beats
1. ノルド(诺德) presents P-gauge readings at the All-Species Council (全种族评议会)...
```

### Source material already has chapter structure
If the source has explicit chapters that the user wants to keep:
- Respect the existing division
- Still restructure the content of each chapter into the beats format
- Report: "Source has [N] chapters — preserving original division. Restructuring content into beats format."

If the user specifies a different chapter count than the source has, subdivide or merge as needed.

### Resuming after interruption
If `OUTLINE_RAW.md` exists but `OUTLINE.md` is unstructured (still raw), the skill can resume from Step 2 without re-extracting.

## Key Rules

- **The structured outline is opinionated.** It commits to POV characters, beat ordering, and chapter boundaries. The user can adjust, but the default should be a clear, writable plan — not a list of options.
- **Beats must be specific enough to write from.** A beat that says "they discuss the situation" is useless. A beat that says "Nord reports measurements; dragon elder reveals casualty numbers; council votes to form investigation team" is writable.
- **Preserve source fidelity.** Every major plot event in the source must appear in a chapter's beats. If something is cut, move it to the Appendix with a note.
- **Keep OUTLINE_RAW.md.** The writer subagent or the user may need to reference the original source for details not captured in the structured outline.
- **Character bible is a bonus, not a shortcut.** If the source material is rich in character detail, the auto-generated bible will be good. If it's sparse, mark profiles as `[sparse — needs expansion]` rather than inventing details.
- **Don't over-structure minor scenes.** A transitional scene ("they travel from A to B") can be one beat. A climactic battle might be 6 beats. Match granularity to narrative weight.
