---
name: novel-write
description: "Write one or more chapters of a novel. Reads the outline, character bible, and style/language settings to produce consistent, on-voice prose. Automatically checks character consistency, adds new characters to the bible, and handles bilingual translation. Use when user says \"写小说\", \"写第N章\", \"write chapter\", \"继续写\", \"continue writing\", or wants to generate novel content."
argument-hint: [chapter-number(s) or range — e.g., "ch01" / "ch03-ch05" / "all"]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob
---

# Novel Write: Chapter-by-Chapter Drafting

Write novel chapters: **$ARGUMENTS**

## Overview

This skill writes one or more chapters of a novel, drawing on:
- `novel/OUTLINE.md` — story structure and chapter beats
- `novel/characters/CHARACTER_BIBLE.md` — character profiles and consistency rules
- `novel/settings/STYLE_SETTING.json` — genre, POV, tense, word count target
- `novel/settings/LANGUAGE_SETTING.json` — primary language, bilingual mode
- `novel/settings/CHAPTER_TEMPLATE.md` — chapter structure guidance

## Prerequisites Check

Before writing, verify required files exist:

```
Required:
- novel/OUTLINE.md             → ⚠️ if missing, ask user to provide outline
- novel/characters/CHARACTER_BIBLE.md → ⚠️ if missing, run /character-design first

Recommended (defaults applied if missing):
- novel/settings/STYLE_SETTING.json   → default: zh, 3000字, general genre
- novel/settings/LANGUAGE_SETTING.json → default: monolingual zh
```

If the outline is missing: "⚠️ No outline found at `novel/OUTLINE.md`. Please provide the story outline (paste text or file path)."

## State Persistence

Writing state is persisted to `novel/NOVEL_STATE.json` after each chapter:

```json
{
  "title": "[Novel title or 'Untitled']",
  "total_chapters": 20,
  "chapters_written": 3,
  "last_chapter": "ch03",
  "current_arc": "Part 1",
  "bilingual": false,
  "status": "in_progress",
  "word_counts": {
    "ch01": 3024,
    "ch02": 2987,
    "ch03": 3156
  },
  "character_updates": [
    {"chapter": "ch02", "character": "林夏", "update": "Revealed backstory"}
  ],
  "new_characters_added": ["ch03: 陈警官 (supporting)"],
  "timestamp": "[ISO timestamp]"
}
```

On invocation, check this file:
- If absent → fresh start from Chapter 1 (or the chapter specified in `$ARGUMENTS`)
- If present → resume from `last_chapter + 1`, or jump to the chapter specified in `$ARGUMENTS`

## Argument Parsing

Parse `$ARGUMENTS` to determine which chapter(s) to write:

| Argument form | Meaning |
|---------------|---------|
| `ch01` / `第一章` / `1` | Write Chapter 1 only |
| `ch03-ch05` / `3-5` | Write Chapters 3 through 5 |
| `all` / `全部` / `继续` / `continue` | Write all remaining unwritten chapters |
| `ch07 --restyle` | Rewrite Chapter 7 with current style settings |
| _(nothing)_ | Write the next unwritten chapter |

## Workflow

### Step 1: Load Context

Read all context files before writing any chapter:

1. `novel/OUTLINE.md` — identify the chapter's plot beats, key events, and emotional arc
2. `novel/characters/CHARACTER_BIBLE.md` — load all character profiles for this chapter's cast
3. `novel/settings/STYLE_SETTING.json` — load genre, POV, tense, word count target, style notes
   - **Custom style file**: if `STYLE_SETTING.json` contains a non-null `custom_style_file` field, read that file now and use its content as the prose guidance for this chapter, replacing `style_notes`. If the file is missing, log "⚠️ Custom style file not found — using genre defaults for this chapter" and continue.
4. `novel/settings/LANGUAGE_SETTING.json` — load language mode
5. `novel/settings/CHAPTER_TEMPLATE.md` — load structural template
6. **Previous chapter file** (if exists) — read the last 200–300 words to ensure continuity of prose rhythm and scene flow
7. `novel/NOVEL_STATE.json` — load current progress state
8. `novel/settings/TIMELINE.md` (if exists) — consult the story timeline to ensure correct in-world dates, event ordering, and time gaps between chapters. This file tracks story-day numbers, key events, and scene-paragraph references for every chapter.

### Step 2: Pre-Writing Analysis

For the target chapter, extract from the outline:

- **Scene goal**: what must happen by the end of this chapter?
- **Characters present**: who appears? Cross-reference with CHARACTER_BIBLE.
- **Emotional arc**: what is the dominant emotional journey of the POV character?
- **New information**: what does the reader (or a character) learn?
- **Forward hook**: what unresolved tension carries into the next chapter?
- **New characters**: does the outline introduce any new named characters in this chapter?

If a new character appears who is not in the CHARACTER_BIBLE:
- Assess role (major / supporting / minor)
- If supporting or major: **create a stub profile before writing their first dialogue** (see Step 2a)
- If minor: note for minor character table

#### Step 2a: New Character Stub (if needed)

Before writing the chapter, add a stub profile to `CHARACTER_BIBLE.md`:

```markdown
## [New Character Name] *(stub — added Ch [N])*

**Role**: Supporting
**First appears**: Chapter [N]
**Status**: Active

### Identity
- Full name: [inferred from context]
- Age: [estimated or unknown]

### Appearance
- [One physical anchor inferred from outline context]

### Personality Core
1. [One trait inferred from their role/function]

### Speech Fingerprint
- Vocabulary: [inferred from role/setting]
- [Will be expanded as more context emerges]

### Relationships
| Character | Relationship | Notes |
|-----------|--------------|-------|
| [Protagonist] | [relationship] | [context from outline] |
```

Then continue to Step 3.

### Step 3: Draft the Chapter

Write the chapter following all loaded settings. Apply them in order of priority:

1. **Plot fidelity**: the chapter must hit the beats specified in the outline
2. **Character voice**: every character's dialogue must match their speech fingerprint
3. **POV discipline**: the specified POV must not break (no "head-hopping" in limited/first-person)
4. **Style settings**: apply genre guidance, prose density, sentence rhythm from STYLE_SETTING.json
5. **Word count target**: aim for ± 10% of the target; ± 20% for pivotal chapters

**Chapter file header:**
```markdown
# [第N章：章节名] / [Chapter N: Title]

> Words: ~[target]  |  POV: [character]  |  Arc: [story arc or part name]
```

The actual prose begins immediately after the header — no meta-commentary.

**Inline image placement:**

Before drafting, check `novel/assets/IMAGE_MAP.json` (if it exists) for any images assigned to this chapter:

```python
import json, pathlib
image_map = json.loads(pathlib.Path("novel/assets/IMAGE_MAP.json").read_text(encoding="utf-8"))
chapter_images = [img for img in image_map if img.get("chapter") == "chNN"]
```

For each image assigned to this chapter, insert a markdown image reference at the appropriate narrative position:

```markdown
![caption text](novel/assets/outline_pXX_imgYY.png)
```

**Placement rules:**
- `"position": "chapter_start"` → insert after the chapter header, before the first paragraph
- `"position": "after_scene_break"` (default) → insert after the `---` that most closely relates to the image content
- `"position": "chapter_end"` → insert after the last paragraph, before the next chapter
- If the image is a **map or world diagram**: place at chapter start or at the first scene where the location is introduced
- If the image is a **character illustration**: place at the character's first significant appearance in this chapter
- If the image is a **setting/concept diagram**: place immediately before or after the passage that describes the concept

These `![](path)` markers are preserved in the markdown and will be embedded automatically by pandoc when exporting to DOCX, PDF, or EPUB. TXT export ignores them.

**Scene break format:** use `---` for time/location/POV shifts.

**POV switch format** (if genre allows multiple POVs):
```markdown
---
*[Character Name]*

[scene content]
```

### Step 4: Self-Consistency Check

After drafting, run the character consistency audit (per `../shared-references/character-consistency.md`):

For each named character in the chapter:
1. Does their physical description match the bible? (If described)
2. Does their dialogue match their speech fingerprint?
3. Does their behavior match their personality core?
4. Do they reference information they shouldn't know yet?

Fix any inconsistencies in the draft before saving.

Also check:
- Continuity with the previous chapter's final scene (location, time of day, character states)
- That plot beats from the outline are all addressed

### Step 5: Save the Chapter

**Monolingual mode:**
```bash
mkdir -p novel/draft
# Save the chapter
```
Save to `novel/draft/chNN.md` (zero-padded: `ch01.md`, `ch02.md`, ..., `ch10.md`, etc.)

**Bilingual mode (translate immediately if `translation_timing: per-chapter`):**
```bash
mkdir -p novel/draft/[primary-lang-code]
mkdir -p novel/draft/[secondary-lang-code]
```
Save primary to `novel/draft/[primary-code]/chNN.md`
Save translation to `novel/draft/[secondary-code]/chNN.md`

For deferred translation (`translation_timing: post-writing`), save primary only and note in state.

### Step 6: Translation (Bilingual Mode)

If bilingual and translating per-chapter:

#### Step 6a: Load Glossary and Style

1. Load `novel/settings/LANGUAGE_SETTING.json` → get `translation_style`
2. **Load `novel/settings/TRANSLATION_GLOSSARY.md`** → build an in-memory lookup table of all confirmed proper nouns

If the glossary file does not exist, warn once and continue (treat all proper nouns as unregistered).

#### Step 6b: Pre-Translation Scan

Before translating, scan the primary-language chapter for proper nouns:
- Character names
- Place names, faction names
- Skill/technique/item names
- Culture-specific terms, honorifics, verbal tics

For each term found:
- **In glossary (confirmed)** → use the registered translation verbatim, no deviation
- **In glossary (待确认 / Pending)** → use the auto-inferred translation from the pending section, mark with `※` in the draft for user review
- **Not in glossary** → auto-infer a reasonable translation, append to the `待确认` section (see Step 6d), mark with `※` in the draft

#### Step 6c: Translate the Chapter

Apply the translation style from `LANGUAGE_SETTING.json`:

- **Literary (意译)**: idiomatic, natural-sounding prose — equivalent emotional impact, not word-for-word
- **Literal (直译)**: preserve source structure; restructure only where grammaticality requires
- **Faithful (忠实原文)**: precise meaning with natural phrasing; keep culture-specific elements with brief contextual notes

**Glossary terms are non-negotiable** — once a translation is confirmed in the glossary, it cannot be varied for style or to avoid repetition (The Banana Rule: "エレナ" is always "艾莲娜", never "艾莲" or "埃勒纳").

**Speech fingerprint in translation:**
- Each character must sound distinct in the target language
- If a character uses a formal/archaic register in the primary language (e.g., classical Japanese 候文), find the equivalent register in the target language (e.g., classical Chinese wényán cadence, or formal literary register)
- Verbal tics registered in the Honorifics table of the glossary must be applied consistently

**Culture-specific elements:**
- Honorifics: use the glossary's registered translation; if unregistered, keep original + parenthetical note on first occurrence per chapter
- Idioms/yojijukugo (四字熟語): translate by meaning, not literally; register in the terminology table if novel-specific
- Setting-specific proper nouns: preserve as-is using glossary form; do not domesticate world-specific terms

#### Step 6d: Append New Terms to Glossary

After translation, collect all terms marked `※` (auto-inferred, unregistered) and append them to the `待确认 / Pending` section of `TRANSLATION_GLOSSARY.md`:

```markdown
| Ch[NN] | [原文] | [自动译名] | [人名/地名/术语/称谓] | 待确认 |
```

Do not modify the confirmed sections of the glossary — only append to Pending.

Save updated glossary to `novel/settings/TRANSLATION_GLOSSARY.md`.

Save translation to `novel/draft/[secondary-code]/chNN.md`.

### Step 7: Update State

Update `novel/NOVEL_STATE.json`:
- Increment `chapters_written`
- Update `last_chapter`
- Record word count for this chapter
- Log any new characters added
- Log any character knowledge state updates

Update `novel/settings/TIMELINE.md` (if it exists):
- Append a new section for this chapter with a table of story-day numbers, key events, scene paragraph references, and notes
- Ensure continuity with the previous chapter's ending day
- If TIMELINE.md does not exist yet, create it with the header format and this chapter's entries

### Step 8: Progress Report

After each chapter (or batch):

```
Chapter [N] complete.

Title: [chapter title]
Words: [actual count] / [target]  ([+/- %])
POV: [character]
New characters added: [names or "none"]
Character consistency: [PASS / N issues fixed]
[Translation: complete (ja→zh)] (bilingual only)
[Glossary: [N] new terms appended to 待确认 — review novel/settings/TRANSLATION_GLOSSARY.md]

Files:
- novel/draft/chNN.md
[- novel/draft/en/chNN.md]  (bilingual)

Progress: [N] / [total] chapters written
Estimated total words so far: [N]

Next: Chapter [N+1] — [brief beat from outline]
Run /novel-write to continue, or /novel-export when ready.
```

## Batch Writing Mode

When writing multiple chapters (`all` or a range):

- Write chapters sequentially, not in parallel (each chapter depends on the previous)
- After every chapter, run the consistency check and save state
- If a chapter requires a new character → create the stub, write the chapter, then continue
- Present a brief progress update after each chapter
- At the end of the batch, present the full batch summary

**Context window management for long novels**: when writing chapter 10+, the full character bible and all previous chapters may not fit in context. Use this strategy:
- Always load the full CHARACTER_BIBLE (compact reference is better than none)
- Load only the previous chapter's final scene (last 300 words), not the full chapter
- Load only the relevant outline section, not the full outline

## Restyle Mode (`--restyle`)

When called with `--restyle`:
1. Read the existing chapter file
2. Apply current STYLE_SETTING.json (may have been changed since original write)
3. Preserve all plot events and character information
4. Rewrite the prose to match the new style
5. Save timestamped backup before overwriting: `novel/draft/chNN_backup_YYYYMMDD_HHmmss.md`

## Key Rules

- **Large file handling**: If Write fails, retry with Bash heredoc. Do not ask for permission.
- **Never invent plot events** not in the outline without flagging them: "I added [X] to bridge the outline's gap at Chapter N — confirm this fits the story."
- **Never skip the consistency check**. A chapter with a character contradiction is worse than a slightly shorter chapter.
- **Word count is a target**: do not pad prose to hit the number. End the chapter when the scene is complete.
- **Speech fingerprints are non-negotiable**: if a character's voice feels wrong, rewrite the dialogue before saving. Do not explain the problem to the user — just fix it.
- **Bilingual translation happens after the primary draft is complete** — never translate a half-written chapter.
