# Claude Code Novel-Writing Pipeline

**[中文版 README](README_CN.md)**

A structured novel-writing system for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that turns a story outline into a fully drafted, proofread, illustrated novel — with EPUB export.

It uses an **orchestrator + writer subagent + proof-reader** pattern: you act as editor, a writer persona drafts each chapter in independent context, and a mechanical proof-reader catches consistency errors. The three roles never share context, which eliminates blind spots.

Battle-tested on a 33-chapter, 250,000+ character novel with bilingual dialogue (Chinese narration / Japanese dialogue), fusion expansion of an existing work, and 38 embedded illustrations.

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

### Why This Pattern?

- **Independent context** — the writer starts fresh, avoiding the editor's accumulated bias
- **Load-instructions, not content-injection** — the subagent reads files itself via tool calls, so the editor's context stays clean
- **Mechanical + aesthetic review** — proof-reader runs pattern checks; editor adds persona fidelity judgment
- **Resumable** — all state on disk; interrupt and resume at any chapter

---

## Quick Start

```bash
git clone https://github.com/user/claude-novel-pipeline.git my-novel
cd my-novel
claude --system-prompt-file ".claude/system-prompt.md"
```

### Full Pipeline (one command)

```
/novel-writing-pipeline "my_outline.pdf -genre: fantasy -chapter_words: 5000"
```

### Step by Step

```
/novel-outline "raw_outline.pdf 20章 每章5000字"        # Phase 0: Outline + Character Bible
/language-setting "台词双语 中文写作 日语台词"             # Phase 1: Language mode
/novel-style "literary, third-person limited, 5000字"    # Phase 2: Style config
/novel-write "all"                                       # Phase 3: Draft all chapters
/novel-export "epub"                                     # Phase 4: Export
```

### Novel Fusion (expand an existing novel)

```
/novel-fusion "existing_novel.docx fusion_outline.md"    # Set up fusion structure
/novel-write "all"                                       # Write new + modify chapters
/novel-export "epub"
```

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
| `/novel-write` | Write chapters (orchestrator + subagent + proof-reader) | `/novel-write "ch01"` / `"all"` / `"ch03-ch05"` |
| `/proof-reader` | 6-category quality review | `/proof-reader "ch01"` / `"all"` |
| `/character-design` | Create/update character profiles | `/character-design "add: rival in ch08"` |
| `/language-setting` | Configure language mode | `/language-setting "台词双语 中文 日语"` |
| `/novel-style` | Genre, POV, word count, prose style | `/novel-style "literary, 8000字"` |
| `/novel-fusion` | Set up fusion mode (expand existing novel) | `/novel-fusion "source.docx outline.md"` |
| `/asset-map` | Map images from source to chapters | `/asset-map "all"` |
| `/novel-export` | Export to EPUB / DOCX / PDF / TXT | `/novel-export "epub docx"` |
| `/proofread-translation` | Review translation quality | `/proofread-translation "ch01-ch05"` |

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
      proofread-translation.md      ← Translation review
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
git clone https://github.com/user/claude-novel-pipeline.git my-novel
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

## Lessons from Production

These observations come from writing a 33-chapter novel (250K+ characters) through the full pipeline:

1. **The writer's self-checks are unreliable.** Subagents consistently undercount dashes and overcount compliance. The editor must run independent grep verification every chapter. This is by design — the proof-reader exists for this reason.

2. **Metaphor density is the hardest constraint to hold.** Writers (human and AI) naturally reach for comparison. The fix: cut the weakest metaphor in each paragraph, keeping only the one that would leave a hole if removed.

3. **Modify chapters require a different skill.** In fusion mode, "modify" chapters must match the source author's voice, not the persona's. The best modification is invisible — readers shouldn't notice someone touched it.

4. **Don't re-spawn subagents for revision.** It wastes a full context load. The editor fixes proof-reader issues directly via Edit tool — faster and more precise.

5. **The memo is the persona's growth record.** Over 33 chapters, the writer persona develops genuine craft insights through practice. The memo should be treated as valuable institutional knowledge, not disposable notes.

---

## Inspiration

Inspired by [wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) and conversations in the SillyTavern community about creative applications of Claude Code.

## License

MIT
