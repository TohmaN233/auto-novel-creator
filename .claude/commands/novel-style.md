---
name: novel-style
description: "Configure writing style, genre, narrative perspective, and chapter length for a novel. Saves style settings that /novel-write uses to maintain consistent tone and structure across all chapters. Use when user says \"写作风格\", \"文风设置\", \"novel style\", \"每章字数\", \"chapter length\", \"风格配置\", or wants to set up writing parameters before drafting."
argument-hint: [genre — style — chapter-word-count — or: 文风指导.md]
allowed-tools: Read, Write, Edit, Bash(*)
---

# Novel Style: Configure Writing Parameters

Configure writing style for: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

This skill establishes and persists the writing style configuration. Settings are saved to `{project}/settings/STYLE_SETTING.json` and read by every call to `/novel-write`. Once set, the style governs tone, pacing, chapter length, and narrative perspective across the entire novel.

## Parsing Arguments

Extract from `$ARGUMENTS`:

### Genre
Detect from keywords — map to a genre code:

| Keywords | Genre code | Default characteristics |
|----------|-----------|------------------------|
| "都市" / "现代" / "contemporary" | `contemporary` | Conversational prose, realistic pacing |
| "言情" / "romance" / "爱情" | `romance` | Emotionally rich, internalized POV |
| "悬疑" / "推理" / "thriller" / "mystery" | `thriller` | Short chapters, high tension, withheld information |
| "玄幻" / "奇幻" / "fantasy" / "修仙" | `fantasy` | World-building descriptions, elevated register |
| "科幻" / "sci-fi" / "science fiction" | `scifi` | Technical precision, speculative extrapolation |
| "历史" / "historical" | `historical` | Period-appropriate register, slower pacing |
| "武侠" | `wuxia` | Poetic descriptions, honor/code themes |
| "纯文学" / "literary fiction" / "文学" | `literary` | Dense prose, psychological depth |
| "轻小说" / "light novel" / "轻松" | `light` | Fast pacing, humor, short chapters |
| None specified | `general` | Neutral, adaptive |

### Narrative Perspective (POV)
| Keywords | POV code |
|----------|----------|
| "第一人称" / "first person" / "我" | `first_person` |
| "第三人称限制" / "limited third" / "close third" | `third_limited` (default) |
| "第三人称全知" / "omniscient" / "上帝视角" | `third_omniscient` |
| "第二人称" / "second person" | `second_person` |

### Tense
| Keywords | Tense |
|----------|-------|
| "过去时" / "past tense" | `past` (default) |
| "现在时" / "present tense" / "现在进行时" | `present` |

### Chapter Word Count
- Parse any number + "字" / "words" / "字数"
- Examples: "3000字" → 3000; "5000 words" → 5000; "每章两千字" → 2000
- Default: 3000 (Chinese) / 2500 (English)
- Range guidance:
  - 1500–2500: short chapters (thriller/light novel pacing)
  - 2500–4000: standard commercial fiction
  - 4000–6000: literary fiction, epic fantasy
  - 6000+: serialized web novel (warn: may strain context window per chapter)

### Prose Density
| Keywords | Density code |
|----------|-------------|
| "精炼" / "简洁" / "concise" | `tight` |
| "细腻" / "详细" / "descriptive" / "细节丰富" | `rich` |
| Default | `balanced` |

### Sentence Rhythm
| Keywords | Rhythm |
|----------|--------|
| "短句" / "short sentences" | `staccato` |
| "长句" / "flowing" / "流畅" | `flowing` |
| Default | `mixed` |

### Dialogue Density
| Keywords | Value |
|----------|-------|
| "对话多" / "dialogue-heavy" | `high` |
| "旁白多" / "narration-heavy" | `low` |
| Default | `balanced` |

### Custom Style File
If `$ARGUMENTS` contains a `.md` file path (any token ending in `.md`, or a path containing `/` or `\`), treat it as a **custom style guide file**:

| Signal | Example |
|--------|---------|
| Any `*.md` path | `文风指导.md`, `styles/my-style.md` |
| Keywords + path | `"参考文风 文风指导.md"`, `"style file: xxx.md"` |
| Inline keyword | `"文风文件"`, `"custom style"` followed by a path |

Multiple inputs are allowed: `"romance — 文风指导.md — 3000字"` → genre `romance` + custom style file + word count.

When a custom style file is detected, it is loaded in **Step 1.5** and its content becomes the primary prose guidance, overriding the genre-specific defaults in Step 2.

## Workflow

### Step 1: Parse and Present Detected Settings

```
Writing style detected:

Genre:            [genre name]
POV:              [perspective]
Tense:            [past / present]
Chapter length:   [N] 字 / words (target)
Prose density:    [tight / balanced / rich]
Sentence rhythm:  [staccato / mixed / flowing]
Dialogue density: [high / balanced / low]

Confirm? (or adjust any setting)
```

If genre was not specified, ask: "What genre is this novel? (e.g., romance / thriller / fantasy / literary)"

### Step 1.5: Load Custom Style File (if provided)

Skip this step if no `.md` file path was detected in `$ARGUMENTS`.

**1. Locate the file** — try these paths in order until one resolves:
   1. Exact path as given (absolute or relative to CWD)
   2. `{project}/[path]`
   3. Project root `[path]`

**2. If found** — read the full file content. This becomes `custom_style_content` and is used as the primary `style_notes` in Step 3, replacing the genre-generated guidance. Show a one-line confirmation:
   ```
   Custom style guide loaded: [filename] ([N] lines)
   ```

**3. If not found** — stop and ask:
   ```
   ⚠️ Style guide file "[path]" not found.
   Options:
   a) Provide the correct file path
   b) Paste the style guide content directly
   c) Press Enter to use genre defaults instead
   ```
   Wait for user response before continuing.

**Key behavior:**
- The custom file's content **replaces** the genre-specific prose guidance generated in Step 2, but does not affect structural settings (POV, tense, word count, chapter ending rules — those still come from arguments/defaults).
- The resolved file path is saved to `STYLE_SETTING.json` as `custom_style_file`. `/novel-write` **re-reads the file at each chapter write**, so you can edit the style guide mid-novel without re-running `/novel-style`.
- If the file is empty, fall back to genre defaults and warn: "⚠️ Style file is empty — using genre defaults."

### Step 2: Generate Style Guide

**If a custom style file was successfully loaded in Step 1.5**: skip generating genre-specific guidance. The custom file content is already `custom_style_content` and will be used directly as `style_notes` in Step 3. Proceed to Step 3.

**Otherwise**: generate a brief prose style guide based on the genre. This generated text becomes `style_notes` in Step 3.

Based on the settings, generate a brief prose style guide that `/novel-write` will reference:

**Genre-specific guidance examples:**

*Romance* (`romance`):
- Prioritize the protagonist's inner emotional state over external action
- Slow the narrative at moments of emotional significance
- Sensory details should anchor to physical sensation and emotional response
- Dialogue should carry subtext — what is not said matters as much as what is

*Thriller* (`thriller`):
- End every chapter on a beat that compels the reader forward
- Withhold one key piece of information per scene — the reader should always be slightly behind the character
- Action sequences: short sentences. No introspection mid-chase.
- Interiority is reserved for aftermath, not action

*Fantasy* (`fantasy`):
- World-building should be revealed through action and consequence, not exposition dumps
- Magic system rules must be applied consistently — establish costs and limits early
- Elevated register for formal settings; vernacular for common characters
- Landscape descriptions should reflect the emotional state of the POV character

*Literary* (`literary`):
- Each paragraph does two jobs: advances plot AND deepens character or theme
- Avoid adverbs and clichéd similes
- Subtext over text: show the drift before the fight, not the fight itself
- Free indirect discourse encouraged for third-person POV

*Light novel* (`light`):
- Fast scene-cuts, minimal lingering description
- Humor through character reaction contrast
- Short chapters encourage re-read momentum
- Dialogue is primary vehicle for character personality

### Step 3: Write STYLE_SETTING.json

```json
{
  "genre": "romance",
  "genre_name": "言情",
  "pov": "third_limited",
  "pov_description": "第三人称限制视角（主角视角）",
  "tense": "past",
  "chapter_word_count_target": 3000,
  "chapter_word_count_range": [2700, 3300],
  "prose_density": "rich",
  "sentence_rhythm": "mixed",
  "dialogue_density": "balanced",
  "style_notes": "[generated prose guidance — or full content of custom style file]",
  "custom_style_file": "[resolved path to .md file, or null if not using custom style]",
  "chapter_ending_rule": "End each chapter with a forward hook or unresolved tension.",
  "configured_at": "[ISO timestamp]"
}
```

Save to `{project}/settings/STYLE_SETTING.json`.

### Step 4: Chapter Template

Based on the style, generate a structural template for each chapter that `/novel-write` will follow:

```markdown
## Chapter Template ([Genre])

Each chapter should contain:
1. **Opening hook** (1–2 paragraphs): drop the reader into the scene already in motion
2. **Scene development** ([N]% of word count): primary action / dialogue / internal state
3. **Midpoint beat** (optional): a revelation, reversal, or decision point
4. **Closing hook**: either resolve the chapter's tension or introduce a new one
   - [Genre-specific ending rule, e.g., "Romance: end on an emotional note, not an action beat"]

Scene transitions:
- Use section breaks (---) to shift time, location, or POV
- Each section should have a clear purpose; delete any section that could be removed without the reader noticing
```

Save chapter template to `{project}/settings/CHAPTER_TEMPLATE.md`.

### Step 5: Confirm Output

```
Style configuration saved.

Genre: [genre] | POV: [pov] | Tense: [tense]
Chapter length: [N] 字 target ([N-range] acceptable)
Prose: [density] | Rhythm: [rhythm] | Dialogue: [density]
[Custom style: [filename] (live — edits take effect immediately)]   ← only if custom file used

Files:
- {project}/settings/STYLE_SETTING.json
- {project}/settings/CHAPTER_TEMPLATE.md
[- [custom_style_file path] (referenced, not copied)]               ← only if custom file used

Next steps:
- /language-setting  → configure language / bilingual mode
- /character-design  → design characters from the outline
- /novel-write       → start writing
```

## Re-configuring Mid-Novel

If called after chapters have already been written, warn:

```
⚠️ Style settings changed after [N] chapters were written.
New settings apply to all future chapters. Existing chapters will not be retroactively rewritten.
If you want to restyle existing chapters, run /novel-write [ch01-chNN] --restyle.
```

## Key Rules

- **Large file handling**: If the Write tool fails, retry with Bash heredoc.
- **Never silently apply defaults** for genre — always confirm, as genre determines the entire prose guidance.
- **Word count is a target, not a hard limit**: chapters may be 10% shorter if the scene naturally ends, or 20% longer for pivotal scenes. Do not pad to hit the target.
- **POV consistency**: once set, the POV should not change mid-novel unless the outline explicitly calls for a POV switch between chapters. Flag POV switches in the chapter header.
- **Custom style file is live**: the file path is stored in `STYLE_SETTING.json`, not the content. `/novel-write` reads the file fresh at every chapter — edit `文风指导.md` (or whatever file you chose) at any time between chapters and the new guidance applies immediately. No need to re-run `/novel-style`.
- **Custom overrides genre guidance only**: genre code, POV, tense, word count, and chapter-ending rules are always set from arguments regardless of the custom style file. The file governs prose texture — not story structure.
- **Validate on every chapter write**: if `/novel-write` cannot find the registered `custom_style_file` at write time, it falls back to the genre defaults and logs a warning: "⚠️ Custom style file missing — using genre defaults for this chapter."
