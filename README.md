# Claude Code Novel-Writing Pipeline

**[中文版 README](README_CN.md)**

A structured novel-writing system for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that turns a story outline into a fully drafted, proofread, illustrated novel — with EPUB export.

It uses an **orchestrator + writer subagent** pattern: the main agent acts as editor, a writer persona drafts each chapter in independent context, and the editor invokes a `/proof-reader` skill for mechanical quality checks before fixing issues itself. The two agents never share context, which eliminates blind spots.


---

## Architecture

```
  Editor (Orchestrator)                    Writer (Subagent)
  ┌──────────────────────┐                ┌───────────────────────┐
  │ 1. Load NOVEL_STATE  │                │ Reads ALL files itself │
  │ 2. Scan outline      │   template +   │ (persona, style, bible,│
  │ 3. Fill template     │───paths only──>│  glossary, timeline,   │
  │    (paths + notes)   │                │  outline, prev chapter)│
  │                      │                │                        │
  │                      │   file paths   │ Writes chapter to disk │
  │ 4. Read draft file <─│<──  only  ─────│ Writes notes to disk   │
  │                      │                │ Updates memo.md        │
  │ 5. /proof-reader     │                └───────────────────────┘
  │    (6 categories)    │
  │ 6. Fix issues (Edit) │
  │ 7. Update state      │
  └──────────────────────┘

  Boundary rules:
  • Editor NEVER reads content files for the writer
  • Writer NEVER updates NOVEL_STATE / TIMELINE / CONTINUITY_MAP
  • SPECIAL_NOTES (≤300 words) is the ONLY editorial content injected
```

## Design Philosophy

### Why two agents instead of one writing start to finish?

The biggest problem with AI long-form writing isn't "bad prose" — it's **not being able to see its own bad prose.** When the same context writes and reviews, it develops confirmation bias — it skips problems during proofreading because it "remembers" its own intent.

So we split creation and review into two isolated contexts:
- The **writer** (subagent) creates in isolated context, free from the editor's accumulated bias
- The **editor** (main agent) receives the finished draft, invokes the `/proof-reader` skill for mechanical checks (counting, formatting, glossary matching), then reads the report and fixes issues directly via Edit tool — no re-spawning the writer, because a full context reload is expensive and the editor with a report in hand is faster and more precise

### Load-Instructions, not Content-Injection

The conventional approach: the editor pre-reads all files, summarizes them, and stuffs the summary into the writer's prompt. We don't do this.

The editor gives the writer **file paths only**. The writer reads original files itself via Read tool. Benefits:
1. The writer gets first-hand material, not the editor's digested summary
2. The editor's context stays clean for proofreading and revision
3. The only editorial content crossing the boundary is SPECIAL_NOTES (≤300 words) — directional guidance like "last chapter's proof-reader flagged 90 dashes, watch the count"

### Writer Persona: constraint, not decoration

A 30+ chapter novel needs to sound like the same author wrote every chapter. The persona system (`persona.yaml`) defines aesthetic preferences, vocabulary habits, and avoidance lists. After each chapter, the writer appends craft insights to `memo.md` — not a log, but cross-chapter growth memory. Over 33 chapters, the persona accumulates genuine craft knowledge.

### All state on disk — interrupt and resume anywhere

Token limit hit at chapter 17? Close the terminal, reopen, run `/novel-write "continue"` — picks up at chapter 18. All progress (NOVEL_STATE.json), timeline (TIMELINE.md), and world state (CONTINUITY_MAP.md) are disk files updated by the editor after each chapter.

---

## Quick Start

Install as a Claude Code skill:

```
npx skills add TohmaN233/auto-novel-creator
```

Or clone the repo:

```bash
git clone https://github.com/TohmaN233/auto-novel-creator.git my-novel
cd my-novel
claude --system-prompt-file ".claude/system-prompt.md"
```

### Launch Pipeline (interactive, guided)

The pipeline is not a fire-and-forget one-liner — it walks you through 5 phases, pausing at each gate for confirmation.

```
/novel-writing-pipeline "my_outline.pdf -genre: fantasy -chapter_words: 5000"
```

After launch, the pipeline guides you through:

```
Phase 1  Outline + Character Bible   → Shows chapter breakdown and cast, waits for review
Phase 2  Language Configuration      → Monolingual / bilingual / dialogue-bilingual? Confirm
Phase 3  Style Configuration         → Genre, POV, word count, prose density — confirm
Phase 4  Chapter Writing             → Runs autonomously after Phase 3 gate: subagent drafts, editor proof-reads & fixes
Phase 5  Export                      → Choose format, generate final files
```

> 💡 Phases 1–3 are configuration (a few minutes each). Once you confirm Phase 3, Phase 4 runs autonomously — start it at night, wake up to a finished draft.

### Step by Step (manual control)

Skip the pipeline orchestrator and call each skill directly:

```
/novel-outline "raw_outline.pdf 20章 每章5000字"        # Phase 1: Outline + Character Bible
/language-setting "台词双语 中文写作 日语台词"             # Phase 2: Language mode
/novel-style "literary, third-person limited, 5000字"    # Phase 3: Style config
/novel-write "all"                                       # Phase 4: Draft all chapters
/novel-export "epub"                                     # Phase 5: Export
```

Each skill is independently callable. Already have an outline and character bible? Jump straight to `/novel-write "all"`.

### Novel Fusion (expand an existing novel)

```
/novel-fusion "existing_novel.docx fusion_outline.md"    # Set up fusion structure
/novel-write "all"                                       # Write new + modify chapters
/novel-export "epub"
```

---

## Configuration Reference

All parameters have sensible defaults. Override via pipeline arguments or individual skill calls.

### Basic Settings

| Parameter | Default | Description | Example |
|-----------|---------|-------------|---------|
| `genre` | auto-detect | Genre/style | `genre: fantasy` `genre: literary` |
| `chapter_words` | 8000 | Target words per chapter | `每章5000字` `chapter_words: 12000` |
| `language_mode` | `monolingual` | Language mode | `台词双語` `bilingual: zh/en` |
| `primary_language` | `zh` | Writing language | `primary: ja` |
| `secondary_language` | `en` | Second language (bilingual mode) | `secondary: zh` |
| `output_format` | `epub docx` | Export formats | `output: epub pdf` `output: all` |
| `writer_persona` | first in directory | Writer persona name | `persona: Elie` |

### Flow Control

| Parameter | Default | Description |
|-----------|---------|-------------|
| `auto_proceed` | `false` | Set `true` to skip Phase 1–3 confirmation gates |
| `auto_export` | `false` | Set `true` to auto-export after writing completes |
| `chapter_range` | `all` | Which chapters to write — `ch05` / `ch03-ch08` / `all` |

### Writer Journal Mode (journal_mode)

Controls what the writer puts in each chapter's `chNN_notes.md`. This determines whether you can "see" the writer's creative process.

| Mode | Content | When to use |
|------|---------|-------------|
| **`full`** | Writing journal (~300-500 words) + afterword (~200-300 words) + Handoff Notes | **Recommended.** You get to experience the writer's personality — their hesitation, excitement, opinions about characters |
| **`brief`** | Handoff Notes + optional short reflection | Just the essential handoff info |
| **`off`** | Handoff Notes only | Pure work mode |

> 💡 **The writer journal is one of the most interesting outputs of this system.** A subagent's `<thinking>` is invisible by default, but journal mode lets the writer save their creative process and feelings as readable text to disk. You can see the writer wrestling with a scene, forming opinions about characters, giving honest self-assessments.
>
> The default Elie persona is calm and understated, so her journals are measured. But if you create a more distinctive writer — say, a tsundere genius novelist — their journals become something else entirely: proud yet dissatisfied, emotionally reactive to the editor's feedback, peppered with defiant self-justification in the afterword... This is the playability the persona system enables.

### Custom Style Guide (custom_style_file)

`/novel-style` generates a `STYLE_SETTING.json` with basic parameters (genre, POV, word count). For finer-grained control — forbidden vocabulary, narrative pacing, worldbuilding anchors, dialogue conventions — write a custom style guide that the writer loads alongside the JSON settings.

Point to it in `STYLE_SETTING.json`:

```json
{
  "custom_style_file": "custom_style_guide_western_fantasy.md"
}
```

See `examples/custom_style_guide_western_fantasy.md` for a complete example covering forbidden vocabulary, narrative rules, atmosphere direction, and hard prohibitions.

### Fusion Mode Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `fusion` / `融合` / `扩写` | Enable fusion mode | Auto-detected, or specify manually |
| `source:` | Source novel file path | `source: original.docx` |

---

## Per-Chapter Workflow

Each chapter follows this exact sequence:

| Step | Role | Action |
|------|------|--------|
| 1 | Editor | Read `NOVEL_STATE.json` + this chapter's outline section |
| 2 | Editor | Check for new characters → add stubs to CHARACTER_BIBLE |
| 3 | Editor | Fill `SUBAGENT_TEMPLATE.md` slots (chapter #, title, paths, SPECIAL_NOTES) |
| 4 | Writer | *Subagent spawned* — reads all project files, thinks, writes chapter + notes to disk |
| 5 | Editor | Read draft → invoke `/proof-reader` (6-category check) |
| 6 | Editor | Fix HIGH/MEDIUM issues via Edit tool (no re-spawn) |
| 7 | Editor | Persona fidelity check against `checklist.md` |
| 8 | Editor | Update NOVEL_STATE, TIMELINE, CONTINUITY_MAP |

The writer subagent executes **9 internal steps** (defined in `SUBAGENT_TEMPLATE.md`): load identity → load settings → load memory → load context → structured thinking → write chapter → write notes → update memo → return lean message.

---

## Skills Reference

| Skill | Purpose | Example |
|-------|---------|---------|
| `/novel-outline` | Raw source → structured OUTLINE.md + CHARACTER_BIBLE | `/novel-outline "concept.pdf 20章"` |
| `/novel-writing-pipeline` | End-to-end orchestrator (drives all other skills) | `/novel-writing-pipeline "outline.pdf -genre: fantasy"` |
| `/novel-write` | Write chapters (editor + writer subagent) | `/novel-write "ch01"` / `"all"` / `"ch03-ch05"` |
| `/proof-reader` | 6-category quality review | `/proof-reader "ch01"` / `"all"` |
| `/character-design` | Create/update character profiles | `/character-design "add: rival in ch08"` |
| `/language-setting` | Configure language mode | `/language-setting "台词双语 中文 日语"` |
| `/novel-style` | Genre, POV, word count, prose style | `/novel-style "literary, 8000字"` |
| `/novel-fusion` | Set up fusion mode (expand existing novel) | `/novel-fusion "source.docx outline.md"` |
| `/asset-map` | Map images from source to chapters | `/asset-map "all"` |
| `/novel-export` | Export to EPUB / DOCX / PDF / TXT | `/novel-export "epub docx"` |

---

## Writer Persona System

Each writer persona is a directory containing three files:

```
writer_persona/
  Elie/                          ← persona name
    persona.yaml                 ← identity: aesthetics, philosophy, speech patterns
    checklist.md                 ← mechanical quality gates (checked after each chapter)
    memo.md                      ← growing craft memory (subagent appends after each chapter)
```

### How It Works

1. The subagent reads `persona.yaml` **before anything else** (Step Zero in the template)
2. All output — prose, thinking, notes, messages — must sound like that persona
3. After writing, the subagent appends craft insights to `memo.md`
4. `memo.md` has a 200-line working cap; old entries get compressed into `# Archive`
5. On the final chapter, a completion summary is written to the Archive section

### Creating a Custom Persona

Copy the `Elie/` directory and modify the YAML. Key fields:

```yaml
name: "Your Writer Name"
aesthetics:
  prose_density: "rich" | "spare" | "balanced"
  favorite_techniques: [...]
  avoidance_list: [...]
personality:
  voice: "first-person description of how they think and speak"
  opinions: [...]
```

Point your project to it via `{project}/settings/active_writer.json`:

```json
{ "writer_dir": "writer_persona/YourWriter" }
```

---

## Language Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Monolingual** | Everything in one language | Standard novels |
| **Bilingual** | Full parallel translation per chapter | Published bilingual editions |
| **Dialogue-bilingual** | Prose in language A, dialogue in language B with inline translation | Authenticity — e.g., Japanese dialogue in Chinese narration |

### Dialogue-Bilingual Format

```
她转过身，背对着窗户站了一会儿。

「……言わなきゃいけないことがある」
（……有些事必须说。）

她的声音很轻，像是怕把什么东西震碎。
```

- Original dialogue uses configured brackets (default: `「」`)
- Translation follows immediately on next line (default: `（）`)
- Narration is exclusively in the writing language
- Every dialogue line must have both versions — the proof-reader counts and verifies

---

## Novel Fusion Mode

Expand an existing novel by inserting new chapters and modifying existing ones.

### Three Chapter Types

| Type | Action | Writer Task |
|------|--------|-------------|
| **passthrough** | Copy source chapter as-is | None (glossary check only) |
| **modify** | Rewrite with specific changes woven in | Match source author's voice, not persona's |
| **new** | Original content between source chapters | Full creative writing with continuity anchors |

### Setup

```
/novel-fusion "original_novel.docx fusion_outline.md"
```

This creates:
- `source/chNN_source.md` — extracted source chapters
- `source/SOURCE_INDEX.md` — chapter type mapping
- `NOVEL_STATE.json` with `"mode": "fusion"`
- Annotated `OUTLINE.md` with TYPE markers per chapter

Then `/novel-write "all"` handles routing automatically: passthrough chapters are copied, modify chapters get change instructions, new chapters get full creative briefs.

---

## Quality System

### Proof-Reader: 6 Categories

| # | Category | What It Checks |
|---|----------|----------------|
| 1 | **Character Consistency** | Speech fingerprints, identity anchors, knowledge state, relationships |
| 2 | **Timeline** | Event order, location continuity, story-day tracking |
| 3 | **Language Quality** | Repetition, awkward phrasing, prose rhythm, style compliance |
| 4 | **Language Contamination** | Grammar leakage between languages, translationese |
| 5 | **Dialogue Format** | Bilingual pairs complete, correct brackets, format consistency |
| 6 | **Glossary Alignment** | Proper noun consistency (the Banana Rule: registered = non-negotiable) |

### Hard Constraints (enforced per chapter)

| Constraint | Limit | Purpose |
|-----------|-------|---------|
| "不是X是Y" pattern | ≤ 2 | Prevents formulaic contrast structures |
| Em-dash (——) | ≤ 30 | Prevents dash overuse |
| "有什么东西" | ≤ 1 | Vague filler phrase |
| Metaphor density | ≤ 2 per 1000 chars | Prevents purple prose |
| AI boilerplate | 0 | Banned phrase patterns (e.g., "不由得感到一阵") |
| Quantified perception | 0 | No "temperature rose 0.3°" (characters aren't instruments) |
| Meta-leakage | 0 | No `chNN`, `Beat N` in prose text |

### Verdicts

| Verdict | Condition | Action |
|---------|-----------|--------|
| **PASS** | 0 HIGH, ≤2 MEDIUM | Save and proceed |
| **PASS_WITH_NOTES** | 0 HIGH, >2 MEDIUM | Save; optionally fix MEDIUMs |
| **NEEDS_REVISION** | ≥1 HIGH | Editor fixes via Edit tool, then re-reviews (max 2 cycles) |

### Persona Fidelity Check

After the mechanical proof-reader pass, the editor evaluates the draft against `checklist.md`:
- ≥3 items failed → HIGH severity (draft sounds generic)
- 1–2 items failed → MEDIUM (note and optionally fix)
- 0 items failed → persona confirmed

---

## Customization & Extension

Every configuration file in this system is user-editable plain text. Every file the writer reads — the template, persona, style guide, character bible, glossary — you can open and modify directly.

If you've used SillyTavern's World Info or Presets, here's how concepts map:

| SillyTavern Concept | This System | File Location |
|---------------------|-------------|---------------|
| Preset | Editor Identity + Writer Persona + Subagent Template | `.claude/system-prompt.md` + `writer_persona/{name}/` + `.claude/SUBAGENT_TEMPLATE.md` (together these form the complete "preset" — who the writer is, how they write, what rules they follow) |
| World Info | Character Bible + Glossary + Timeline + Continuity Map + Custom Style Guide | `{project}/characters/` + `{project}/settings/` (the writer reads these every chapter — whatever you put in, it follows) |
| Author's Note | SPECIAL_NOTES | Editor's ≤300-word per-chapter notes injected into the template |

### What you can freely modify

**Subagent Template** (`.claude/SUBAGENT_TEMPLATE.md`): The complete instruction set the writer receives. You can:
- Add or remove thinking steps (e.g., add a "musicality check" step)
- Change the journal format (e.g., require the writer to write afterwords as a dialogue)
- Add extra constraints (e.g., "every chapter must plant one foreshadowing seed")
- Modify the file-reading list (e.g., add a custom worldbuilding document)

**Character Bible** (`CHARACTER_BIBLE.md`): Per-character behavioral constraints. The writer reads this every chapter. Add any fields you want — "things this character would never do", "secrets only the reader knows".

**Glossary** (`TRANSLATION_GLOSSARY.md`): Proper noun mappings for bilingual modes. Registered translations are non-negotiable — both writer and proof-reader enforce them strictly.

**Proof-reader rules**: Hard constraints (the dash limits, pattern bans, etc.) are defined in `CLAUDE.md` and the `proof-reader` skill. Adjust limits, add/remove entries, add your own AI boilerplate patterns.

**Custom style guide**: Write any rules you want — forbidden words, sentence preferences, narrative techniques, atmosphere direction. Place it in your project directory and point `STYLE_SETTING.json`'s `custom_style_file` to it.

> 💡 **Core principle: if the writer can read it, you can edit it.** Whatever you put in these files, the writer follows. This is not a closed black box — it's a prompt architecture you can continuously tune.

---

## State Files

The pipeline tracks three complementary files:

| File | Tracks | Updated By |
|------|--------|------------|
| `NOVEL_STATE.json` | Pipeline progress, chapter status, proof-reader results | Editor |
| `TIMELINE.md` | **Events** — what happened when (chronological log) | Editor |
| `CONTINUITY_MAP.md` | **States** — who/what/where at each chapter boundary | Editor |

All three are updated after each chapter. The writer subagent reads them for context but never modifies them.

---

## Project Structure

```
claude-novel-pipeline/
  .claude/
    system-prompt.md                ← Novel editor identity
    SUBAGENT_TEMPLATE.md            ← Writer subagent prompt (load-instructions pattern)
    commands/
      novel-writing-pipeline.md     ← End-to-end orchestrator
      novel-outline.md              ← Outline structuring
      character-design.md           ← Character bible generator
      language-setting.md           ← Language mode (3 modes)
      novel-style.md                ← Style configuration
      novel-write.md                ← Chapter drafting engine
      novel-fusion.md               ← Novel expansion
      proof-reader.md               ← 6-category quality review
      asset-map.md                  ← Image mapping
      novel-export.md               ← Export (EPUB/DOCX/PDF/TXT)
  writer_persona/
    Elie/                           ← Default writer persona
      persona.yaml
      checklist.md
      memo.md
  examples/
    active-project.json
    STYLE_SETTING.json
    LANGUAGE_SETTING.json
    LANGUAGE_SETTING_dialogue_bilingual.json
    custom_style_guide_western_fantasy.md
  CLAUDE.md                         ← Claude Code project instructions
  start-novel.ps1                   ← Windows launcher
  README.md
  README_CN.md
  LICENSE

{project}/                          ← Created per novel (gitignored)
  OUTLINE.md
  NOVEL_STATE.json
  characters/CHARACTER_BIBLE.md
  settings/
    LANGUAGE_SETTING.json
    STYLE_SETTING.json
    TRANSLATION_GLOSSARY.md
    TIMELINE.md
    CONTINUITY_MAP.md
    active_writer.json
  source/                           ← Fusion mode only
  draft/chNN.md
  assets/IMAGE_MAP.json
  output/
```

---

## Installation

### Requirements

| Tool | Required For | Install |
|------|-------------|---------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | Everything | See Anthropic docs |
| Python 3 + ebooklib + Pillow | EPUB export with images | `pip install ebooklib Pillow` |
| pandoc | DOCX/PDF export | `winget install JohnMacFarlane.Pandoc` / `brew install pandoc` |
| xelatex | PDF export (CJK fonts) | TeX Live / MiKTeX |

### Setup

```bash
git clone https://github.com/TohmaN233/auto-novel-creator.git my-novel
cd my-novel

# Option A: Standard launch
claude

# Option B: With custom system prompt (recommended)
claude --system-prompt-file ".claude/system-prompt.md"

# Option C: Windows PowerShell
.\start-novel.ps1
```

Claude Code auto-discovers `.claude/commands/*.md` as slash commands. No further configuration needed.

### Multi-Project Support

Each novel lives in its own directory. Configure `.claude/active-project.json`:

```json
{
  "project_dir": "my-first-novel",
  "project_name": "The Title"
}
```

---

## AI-Generated Fiction in General

Let's be honest: AI-generated novels, with current technology, are not ready for professional publication. What this pipeline produces is best described as a **high-quality first draft** — structurally coherent, character-consistent, and stylistically controlled, but still recognizably machine-written to a careful reader. The prose tends toward a certain uniformity of rhythm, the emotional beats can feel mechanically placed, and genuinely surprising creative choices are rare.

This tool is best suited for:

- **Personal enjoyment** — turning your worldbuilding notes into a readable novel for yourself and friends
- **Rapid prototyping** — generating a full draft to see if a story concept works before investing human writing time
- **Fan fiction / derivative works** — where the source material provides rich structure to build on
- **Learning** — studying how an AI interprets your outline can teach you about story structure

If you want to push toward publication quality, the two-agent review architecture helps — but ultimately, a human editor remains essential.

---

## License

MIT
