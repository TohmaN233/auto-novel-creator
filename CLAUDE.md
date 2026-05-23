# Novel Writing Pipeline — Project Instructions

## Overview

This is a structured novel-writing pipeline powered by Claude Code. It uses an **orchestrator + subagent + proof-reader** pattern to produce long-form fiction with consistent quality.

## Architecture

### Three Roles

1. **Editor (you, the orchestrator)** — loads context, delegates writing, reviews output, maintains state files
2. **Writer (subagent, spawned via Agent tool)** — reads all project files independently, writes chapter to disk, updates craft memory
3. **Proof-reader (skill invocation)** — mechanical quality review across 6 categories

### Core Principles

1. **Editor never pre-reads content for the writer.** Prompt contains paths only; subagent reads files itself.
2. **Subagent writes files to disk directly.** Return message contains only metadata.
3. **Subagent maintains its own craft memory.** After each chapter, it appends insights to `writer_persona/{name}/memo.md`.
4. **Editor handles state updates.** NOVEL_STATE, TIMELINE, CONTINUITY_MAP are editor responsibilities.

### Boundary Rules

- SPECIAL_NOTES (≤300 words) is the ONLY editorial content that crosses editor→writer boundary
- The writer NEVER updates NOVEL_STATE, TIMELINE, or CONTINUITY_MAP
- The editor NEVER reads CHARACTER_BIBLE, GLOSSARY, etc. on behalf of the writer

## Per-Chapter Workflow

1. Read `NOVEL_STATE.json` + this chapter's section of `OUTLINE.md`
2. Check for new characters → add stubs to CHARACTER_BIBLE if needed
3. Fill `SUBAGENT_TEMPLATE.md` parameter slots (chapter number, title, paths, SPECIAL_NOTES)
4. Spawn subagent with `Agent(model: "opus")` + filled template
5. Subagent reads all files → thinks → writes chapter → writes notes → updates memo → returns lean message
6. Editor reads `draft/chNN.md` → runs `/proof-reader` → persona fidelity check
7. Editor fixes issues via Edit tool (no re-spawn)
8. Update `NOVEL_STATE.json`, `TIMELINE.md`, `CONTINUITY_MAP.md`

## Quality Hard Constraints (per chapter)

| Constraint | Limit | Check |
|-----------|-------|-------|
| "不是X是Y" pattern | ≤ 2 | grep count |
| Em-dash (——) | ≤ 30 | grep count |
| "有什么东西" | ≤ 1 | exact search |
| Metaphor density | ≤ 2/thousand chars | grep + count |
| AI boilerplate | 0 | pattern blacklist |
| Sensor-tic (tgy) | 0 | no quantified perception |
| Ch-meta leakage | 0 | no `chNN`, `Beat N` in prose |

## Available Skills

| Skill | Purpose |
|-------|---------|
| `/novel-outline` | Source → structured OUTLINE.md + CHARACTER_BIBLE |
| `/language-setting` | Configure language mode |
| `/novel-style` | Configure genre, POV, word count, prose style |
| `/novel-write` | Write chapters (full orchestrator workflow) |
| `/novel-export` | Export to EPUB, DOCX, PDF, TXT |
| `/proof-reader` | Review chapters for quality (6 categories) |
| `/character-design` | Design or update character profiles |
| `/asset-map` | Map images from source to chapters |
| `/novel-writing-pipeline` | End-to-end pipeline orchestrator |
| `/novel-fusion` | Set up fusion mode (expand existing novel) |
| `/translate-text` | Batch translate with glossary |
| `/proofread-translation` | Review translation quality |

## File Structure

```
.claude/
  system-prompt.md              ← System prompt (novel editor identity)
  SUBAGENT_TEMPLATE.md          ← Writer subagent prompt template
  commands/                     ← Skill definitions (slash commands)

writer_persona/
  {name}/
    persona.yaml                ← Writer identity, aesthetics, philosophy
    checklist.md                ← Mechanical quality gates
    memo.md                     ← Growing craft memory (updated per chapter)

{project}/
  OUTLINE.md                   ← Structured plot outline
  NOVEL_STATE.json             ← Pipeline progress tracker
  characters/CHARACTER_BIBLE.md
  settings/
    LANGUAGE_SETTING.json
    STYLE_SETTING.json
    TRANSLATION_GLOSSARY.md
    TIMELINE.md
    CONTINUITY_MAP.md
    active_writer.json
  draft/chNN.md                ← Chapter drafts
  assets/IMAGE_MAP.json        ← Image-to-chapter assignments
  output/                      ← Exported files (EPUB, DOCX)
```
