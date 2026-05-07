# Claude Code Novel-Writing Pipeline

A set of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash-command skills that turn a story outline into a fully drafted, bilingual, illustrated novel — exported as EPUB and DOCX.

## A detailed chapter-by-chapter outline of your novel is necessary

AI-Agent cannot produce a consistent story without a carefully written outline.

## Demo

This pipeline was used to produce a **12-chapter bilingual novel** (JA primary, ZH literary translation) from a Battle Spirits card game lore PDF, complete with 98 embedded card illustrations. The full process — from PDF ingestion to final EPUB — run inside a single Claude Code session, and completed with around 120000 Chinese characters.

## Inspiration & Credits

This project was inspired by [wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) and conversations in the SillyTavern community about pushing Claude Code beyond software engineering into creative workflows. The core idea came directly from those discussions.

## How It Works

The pipeline chains seven slash-command skills into an end-to-end novel-writing lifecycle:

```
  PDF / text outline
        |
        v
  /novel-writing-pipeline  (orchestrator)
        |
        +---> /character-design    Phase 1: Build CHARACTER_BIBLE.md
        +---> /language-setting    Phase 2: Configure mono/bilingual mode
        +---> /novel-style         Phase 3: Genre, POV, chapter length, custom style
        +---> /novel-write         Phase 4: Draft chapters (+ translate if bilingual)
        +---> /asset-map           Phase 4.5: Scan, filter, place images into chapters
        +---> /novel-export        Phase 5: Assemble & export to EPUB / DOCX / PDF / TXT
```

### Persistent State

All intermediate state lives on disk — the pipeline can be interrupted and resumed at any point:

```
novel/
  OUTLINE.md                    # Source outline (input, never modified)
  NOVEL_STATE.json              # Pipeline progress tracker
  characters/
    CHARACTER_BIBLE.md          # Living character reference
  settings/
    LANGUAGE_SETTING.json       # Mono/bilingual config
    STYLE_SETTING.json          # Genre, POV, word-count targets
    CHAPTER_TEMPLATE.md         # Per-genre structural template
    TRANSLATION_GLOSSARY.md     # Proper noun translation table (bilingual)
  draft/
    ja/ch01_ja.md ... ch12_ja.md    # Primary language chapters
    zh/ch01_zh.md ... ch12_zh.md    # Secondary language chapters
    FULL_NOVEL_ja.md                # Assembled full text
    FULL_NOVEL_zh.md
  assets/
    IMAGE_MAP.json              # Image inventory with status tracking
    *.jpeg / *.png              # Raw images (from PDF or user-provided)
    export/                     # Curated images for embedding
  output/
    novel_ja.epub / .docx       # Final exports
    novel_zh.epub / .docx
```

## Skills Reference

### `/novel-writing-pipeline` — Orchestrator

The top-level entry point. Parses your outline source and configuration, then drives all other skills in sequence.

```
/novel-writing-pipeline "my_outline.pdf -bilingual ja->zh -genre: fantasy -chapter_words: 8000"
```

**Key parameters** (all optional, with sane defaults):

| Parameter      | Example                                   | Default                 |
| -------------- | ----------------------------------------- | ----------------------- |
| Outline source | `outline.md`, `story.pdf`, or inline text | *(required)*            |
| Language mode  | `-bilingual ja->zh`, `中英双语`           | monolingual zh          |
| Genre          | `-genre: romance`, `玄幻`                 | auto-detected           |
| Chapter length | `-chapter_words: 5000`, `每章3000字`      | 3000                    |
| POV            | `-pov: first_person`, `第一人称`          | third_limited           |
| Output format  | `-output: epub pdf`                       | epub + docx             |
| Auto-proceed   | `-auto-proceed: true`                     | false (pauses at gates) |
| Custom style   | `-custom_style_file: "style_guide.md"`    | none                    |

**Gates**: By default the pipeline pauses after Phase 1 (character review), Phase 2 (language confirm), and Phase 3 (style confirm) for user approval before autonomous writing begins.

### `/character-design` — Character Bible Generator

Reads the outline and produces `CHARACTER_BIBLE.md` with structured profiles: appearance anchors, personality cores, speech fingerprints, relationship maps, and knowledge boundaries.

```
/character-design "novel/OUTLINE.md"
/character-design "add: a rival swordsman who appears in Chapter 8"
```

Characters are categorized as major / supporting / minor. The bible is a living document — `/novel-write` automatically creates stub profiles for new characters as they appear.

### `/language-setting` — Language Configuration

Configures monolingual or bilingual mode. In bilingual mode, each chapter is first written in the primary language, then translated with glossary enforcement.

```
/language-setting "ja -> zh, literary translation, per-chapter"
/language-setting "zh"  # monolingual Chinese
```

**Translation features**:

- Per-chapter or post-writing translation timing
- Literary / literal / faithful translation styles
- `TRANSLATION_GLOSSARY.md` for consistent proper noun translation
- Speech fingerprint preservation across languages

### `/novel-style` — Style Configuration

Sets genre, POV, tense, chapter word-count target, prose density, and optionally loads a custom style guide file.

```
/novel-style "fantasy-scifi, third-person limited, 8000 words per chapter"
/novel-style "文风指导.md"  # load a custom style file
```

Produces `STYLE_SETTING.json` and `CHAPTER_TEMPLATE.md` (a structural template tailored to the genre).

### `/novel-write` — Chapter Drafting Engine

The core writing skill. Reads the outline, character bible, and style settings, then drafts chapters one by one with self-consistency checks.

```
/novel-write "ch01"           # Write one chapter
/novel-write "ch03-ch05"      # Write a range
/novel-write "all"            # Write all remaining chapters
/novel-write "ch07 --restyle" # Rewrite with current style settings
```

**Per-chapter workflow**:

1. Load outline beats, character profiles, and previous chapter's final scene
2. Pre-writing analysis (scene goals, emotional arc, new characters)
3. Draft prose following all style constraints
4. Self-consistency check against CHARACTER_BIBLE
5. Save primary language draft
6. Translate (if bilingual + per-chapter mode), with glossary enforcement
7. Update NOVEL_STATE.json

### `/asset-map` — Image Filtering & Placement

Processes raw images (e.g., extracted from a PDF outline) into an export-ready set with chapter placements. A four-phase pipeline:

```
/asset-map "all"         # Full pipeline: scan → filter → place
/asset-map "scan-only"   # Phase 1-2: analyze and classify images
/asset-map "filter-only" # Phase 3: apply selection rules
/asset-map "place"       # Phase 4: insert into chapter markdown
/asset-map "report"      # Show current status
```

**Filtering logic**: size-based pre-filter → visual classification (character / map / scene / diagram / icon) → duplicate detection → relevance cross-reference with chapter text → density cap (configurable per chapter). Requires user confirmation before inserting image references into chapter files.

### `/novel-export` — Export to Final Formats

Assembles all chapter files into a single document per language and converts to the requested output format(s) via pandoc.

```
/novel-export "epub docx"     # Default
/novel-export "all"           # txt + docx + pdf + epub
/novel-export "ch01-ch05 epub" # Partial export (preview)
```

**Auto-detects image assets**: if `novel/assets/` exists with images, the export skill automatically checks `IMAGE_MAP.json` and triggers `/asset-map` if images haven't been processed yet.

**Format support**:

| Format | Images   | Tool             |
| ------ | -------- | ---------------- |
| EPUB   | embedded | pandoc           |
| DOCX   | embedded | pandoc           |
| PDF    | embedded | pandoc + xelatex |
| TXT    | stripped | pandoc / sed     |

## Installation

1. Install [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
2. Clone this repo into your project directory
3. The `.claude/commands/` folder contains all skill definitions — Claude Code auto-discovers them as slash commands

```bash
git clone https://github.com/YOUR_USERNAME/claude-novel-pipeline.git my-novel-project
cd my-novel-project
claude  # start Claude Code
```

1. (Optional) Install [pandoc](https://pandoc.org/installing.html) for export — the pipeline will prompt you if it's missing

### Dependencies

| Tool        | Required for           | Install                                                              |
| ----------- | ---------------------- | -------------------------------------------------------------------- |
| Claude Code | Everything             | [docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code) |
| pandoc      | Export (EPUB/DOCX/PDF) | `winget install JohnMacFarlane.Pandoc` / `brew install pandoc`       |
| xelatex     | PDF export only        | TeX Live / MiKTeX                                                    |
| pymupdf     | PDF image extraction   | `pip install pymupdf`                                                |

## Quick Start

```
> /novel-writing-pipeline "my_story_outline.md -bilingual en->zh -genre: romance -chapter_words: 5000"
```

Or run each phase independently:

```
> /character-design "novel/OUTLINE.md"
> /language-setting "en -> zh, literary, per-chapter"
> /novel-style "romance, first-person, 5000 words"
> /novel-write "all"
> /novel-export "epub docx"
```

## Customization

### Custom Style Guide

Create a markdown file describing your desired prose style, then reference it:

```
/novel-writing-pipeline "outline.md -custom_style_file: my_style.md"
```

The style file is loaded before every chapter and overrides the default genre style. It can describe sentence rhythm, dialogue conventions, emotional expression rules, narrative voice, forbidden vocabulary — anything you want the agent to internalize.

See `examples/custom_style_guide_western_fantasy.md` for a complete example (medieval epic fantasy with cinematic narration, dialogue-driven pacing, and strict immersion rules).

### Translation Glossary

For bilingual projects, `novel/settings/TRANSLATION_GLOSSARY.md` is automatically maintained. Confirmed translations are enforced across all chapters (the "Banana Rule" — a proper noun always maps to exactly one translation). New terms discovered during writing are appended to a pending section for your review.

## Known Limitations & Honest Assessment

### Single-Agent Bottleneck

The entire pipeline runs inside one Claude Code agent. This works, but has real consequences:

- **Bilingual contamination**: When the same agent writes Japanese prose and then immediately translates to Chinese, linguistic interference can creep in — Japanese sentence structures leaking into the Chinese translation, or Chinese vocabulary choices being subconsciously biased by the Japanese source still in context. This is especially noticeable in long sessions where both languages share the context window for hours.
- **Context window pressure**: By chapter 10+, the character bible, outline, style guide, glossary, and previous chapter all compete for context space. The agent manages this by loading only what's needed, but information can still be lost.
- **Style drift**: Over many chapters, the prose style may gradually shift as earlier chapters fall out of the context window.

### Multi-Agent Architecture (Not Implemented, But Possible)

A more robust architecture would split the work across specialized agents:

| Agent          | Role                                               | Why it helps                                                  |
| -------------- | -------------------------------------------------- | ------------------------------------------------------------- |
| **Writer**     | Draft chapters in the primary language only        | Stays immersed in one language's prose rhythm                 |
| **Translator** | Translate finished chapters to the target language | No source-language contamination in context                   |
| **Reviewer**   | Read the full draft and flag inconsistencies       | Fresh eyes, full-novel perspective (e.g., use Codex for this) |
| **Editor**     | Apply style corrections across all chapters        | Consistent voice without context-window decay                 |

This is straightforward to implement — you can ask your own Claude Code agent to set up sub-agents, or use the [Claude Agent SDK](https://docs.anthropic.com/en/docs/claude-code/sdk) to orchestrate them programmatically. I chose not to build this because it would multiply token costs significantly, and for the kind of project I was working on (personal creative writing, not publication-grade output), the single-agent approach was the right tradeoff.

### AI-Generated Fiction in General

Let's be honest: AI-generated novels, with current technology, are not ready for professional publication. What this pipeline produces is best described as a **high-quality first draft** — structurally coherent, character-consistent, and stylistically controlled, but still recognizably machine-written to a careful reader. The prose tends toward a certain uniformity of rhythm, the emotional beats can feel mechanically placed, and genuinely surprising creative choices are rare.

This tool is best suited for:

- **Personal enjoyment** — turning your worldbuilding notes into a readable novel for yourself and friends
- **Rapid prototyping** — generating a full draft to see if a story concept works before investing human writing time
- **Fan fiction / derivative works** — like our Battle Spirits lore novelization, where the source material provides rich structure
- **Learning** — studying how an AI interprets your outline can teach you about story structure

If you want to push toward publication quality, the multi-agent review architecture described above would help — but ultimately, a human editor remains essential.

## Project Structure

```
claude-novel-pipeline/
  .claude/
    commands/
      novel-writing-pipeline.md   # Orchestrator
      character-design.md         # Character bible generator
      language-setting.md         # Language configuration
      novel-style.md              # Style configuration
      novel-write.md              # Chapter drafting engine
      asset-map.md                # Image filtering & placement
      novel-export.md             # Export to EPUB/DOCX/PDF/TXT
  examples/
    STYLE_SETTING.json                        # Example style config
    LANGUAGE_SETTING.json                     # Example bilingual config
    CHAPTER_TEMPLATE.md                       # Example chapter template (fantasy-scifi)
    custom_style_guide_western_fantasy.md     # Example custom style guide
  README.md
  LICENSE
```

## License

MIT
