---
name: novel-write
description: "Write one or more chapters of a novel. Main agent orchestrates: loads context, delegates writing to a subagent, then reviews with proof-reader checks. Supports monolingual, bilingual, and dialogue-bilingual modes. Automatically checks character consistency, adds new characters to the bible, and handles translation. Use when user says \"写小说\", \"写第N章\", \"write chapter\", \"继续写\", \"continue writing\", or wants to generate novel content."
argument-hint: [chapter-number(s) or range — e.g., "ch01" / "ch03-ch05" / "all"]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, Skill
---

# Novel Write: Chapter-by-Chapter Drafting

Write novel chapters: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Architecture: Orchestrator + Subagent

This skill uses a two-agent pattern (the proof-reader is a skill the orchestrator invokes, not a separate agent):

```
Main Agent (you) ─── orchestrator
  │
  ├─ Step 0: Prerequisites — initialize TIMELINE.md, GLOSSARY if missing
  │
  ├─ Step 1-2: Load context, prepare writing brief
  │
  ├─ Step 3: Agent() → Spawn writing subagent ──→ [Subagent writes chapter, saves draft]
  │
  ├─ Step 4: Skill("proof-reader") → MANDATORY review ──→ [Structured report with verdict]
  │     │
  │     ├─ PASS → proceed to save
  │     └─ NEEDS_REVISION → fix → Skill("proof-reader") again (max 2 cycles)
  │
  └─ Step 5-7: Finalize, update state, report
```

## Prerequisites Check

Before writing, verify required files exist — **and create missing ones immediately, not later**:

```
Required:
- {project}/OUTLINE.md             → if missing, ask user to provide outline
- {project}/characters/CHARACTER_BIBLE.md → if missing, run /character-design first

Required — create NOW if missing (do NOT defer to "after first chapter"):
- {project}/settings/STYLE_SETTING.json   → if missing, run /novel-style with defaults (zh, 5000字, general)
- {project}/settings/LANGUAGE_SETTING.json → if missing, run /language-setting with default (monolingual zh)
- {project}/settings/TIMELINE.md          → if missing, create a blank template NOW (see Step 0a)
- {project}/settings/CONTINUITY_MAP.md    → if missing, create a blank template NOW (see Step 0c)
- {project}/settings/TRANSLATION_GLOSSARY.md → if missing AND mode is bilingual/dialogue-bilingual, create NOW (see Step 0b)

Fusion mode additional:
- {project}/NOVEL_STATE.json with "mode": "fusion" → triggers fusion workflow
- {project}/source/                                → source novel chapter files
- {project}/source/SOURCE_INDEX.md                 → source chapter index
```

### Step 0a: Initialize TIMELINE.md (if absent)

Create `{project}/settings/TIMELINE.md` before writing Chapter 1:

```markdown
# Story Timeline

> Auto-initialized by /novel-write. Updated after each chapter.

| Chapter | Story Day | Events | Characters Present | Location |
|---------|-----------|--------|--------------------|----------|
```

### Step 0b: Initialize TRANSLATION_GLOSSARY.md (if absent, bilingual/dialogue-bilingual only)

If `LANGUAGE_SETTING.json` mode is `bilingual` or `dialogue-bilingual` and no glossary exists:

1. Read `{project}/characters/CHARACTER_BIBLE.md`
2. Extract all named characters
3. Create `{project}/settings/TRANSLATION_GLOSSARY.md` with character name entries pre-filled (translations as `[待填写]`)
4. Warn user (in user language):
   ```
   ⚠️ 译名表已自动创建，但所有译名列为空（[待填写]）。
   建议在写作前填入关键角色的译名。未填入的将在写作时自动推断并标记为"待确认"。
   ```

### Step 0c: Initialize CONTINUITY_MAP.md (if absent)

Create `{project}/settings/CONTINUITY_MAP.md` before writing Chapter 1:

```markdown
# Continuity Map

> World-state snapshots at each chapter boundary.
> Updated by the orchestrator after each chapter.
>
> CONTINUITY_MAP tracks STATE (who/what/where is X now).
> TIMELINE tracks EVENTS (what happened when).
> These two files do not overlap.

---

## Before Ch.1

- **Character states**: [initial states from OUTLINE / CHARACTER_BIBLE]
- **Relationships**: [initial relationship dynamics]
- **Items/artifacts**: [any starting items]
- **Unresolved threads**: [opening questions / hooks]
- **Transition notes**: [what Ch.1 needs to establish]
```

## State Persistence

Writing state is persisted to `{project}/NOVEL_STATE.json` after each chapter:

```json
{
  "title": "[Novel title or 'Untitled']",
  "total_chapters": 20,
  "chapters_written": 3,
  "last_chapter": "ch03",
  "current_arc": "Part 1",
  "language_mode": "dialogue-bilingual",
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
  "proof_reader_results": {
    "ch01": "PASS",
    "ch02": "PASS_WITH_NOTES",
    "ch03": "PASS"
  },
  "timestamp": "[ISO timestamp]"
}
```

On invocation:
- If absent → fresh start from Chapter 1 (or the chapter specified in `$ARGUMENTS`)
- If present → resume from `last_chapter + 1`, or jump to the chapter specified
- If `"mode": "fusion"` → enable fusion workflow (see Fusion Mode below)

## Argument Parsing

| Argument form | Meaning |
|---------------|---------|
| `ch01` / `第一章` / `1` | Write Chapter 1 only |
| `ch03-ch05` / `3-5` | Write Chapters 3 through 5 |
| `all` / `全部` / `继续` / `continue` | Write all remaining unwritten chapters |
| `ch07 --restyle` | Rewrite Chapter 7 with current style settings |
| _(nothing)_ | Write the next unwritten chapter |

## Per-Chapter Workflow

### Division of Labor

```
┌─────────────────────────────────────────────────────────────┐
│  EDITOR (you, the orchestrator)                             │
│                                                             │
│  Step 0:  Initialize missing files (TIMELINE, GLOSSARY…)    │
│  Step 1:  Load minimal context (NOVEL_STATE, outline scan)  │
│  Step 1f: Fusion routing (passthrough / new / modify)       │
│  Step 2:  Pre-processing — new character stubs,             │
│           resolve writer persona                            │
│  Step 3a–b: Fill template slots, compose SPECIAL_NOTES      │
│  Step 3c: Spawn subagent ──────────────────────────┐        │
│  Step 3d: Receive & verify (files on disk)    ◄────┘        │
│  Step 4:  Proof-reader (mechanical) + persona fidelity      │
│  Step 4c: Fix issues yourself (Edit tool)                   │
│  Step 6:  Update NOVEL_STATE, TIMELINE, CONTINUITY_MAP      │
│  Step 7:  Progress report                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  WRITER (subagent, executes SUBAGENT_TEMPLATE Steps 1–9)    │
│                                                             │
│  Step 1:  Load identity       (Read persona.yaml)           │
│  Step 2:  Load settings       (Read STYLE + LANGUAGE)       │
│  Step 3:  Load memory         (Read memo.md + checklist.md) │
│  Step 4:  Load chapter context (Read outline, bible,        │
│           glossary, timeline, continuity map, prev chapter)  │
│  Step 5:  Pre-writing thinking (structured <thinking>)      │
│  Step 6:  Write chapter → disk (Write draft/chNN.md)        │
│  Step 7:  Write notes → disk   (Write draft/chNN_notes.md)  │
│  Step 8:  Update memo.md       (Edit writer_persona/.../    │
│                                 memo.md)                    │
│  Step 9:  Return lean message  (status + handoff only)      │
└─────────────────────────────────────────────────────────────┘

Boundary rule:
 - The editor NEVER reads content files on behalf of the writer.
 - The writer NEVER updates NOVEL_STATE, TIMELINE, or CONTINUITY_MAP.
 - SPECIAL_NOTES is the ONLY editorial content that crosses the boundary.
```

### Step 1: Load Minimal Context (Editor Only)

> **You do not need to read all context files.** The subagent reads them itself.
> You only need enough context to: (a) determine chapter type, (b) check for new characters, (c) compose SPECIAL_NOTES.

Read these files:

1. `{project}/NOVEL_STATE.json` — current progress, chapter number, mode
2. `{project}/OUTLINE.md` — **only this chapter's section** to determine:
   - Chapter type (passthrough / modify / new) in fusion mode
   - Characters present (for new character check in Step 2b)
   - Basic scene context (for SPECIAL_NOTES)
3. `{project}/settings/active_writer.json` — resolve writer persona path

**You do NOT need to read** CHARACTER_BIBLE, GLOSSARY, TIMELINE, CONTINUITY_MAP, STYLE_SETTING, LANGUAGE_SETTING, or the previous chapter. The subagent handles all of these.

**Fusion mode only** (for passthrough routing):
- Read `{project}/source/chNN_source.md` header to confirm passthrough chapters exist

### Step 1f: Fusion Chapter Type Routing

**Skip this step if `NOVEL_STATE.json` does not have `"mode": "fusion"`.**

Read the chapter's `TYPE:` annotation from `OUTLINE.md` and branch:

| Chapter type | Route to |
|-------------|----------|
| **passthrough** | → Step 1f-P (copy source, skip writing) |
| **modify** | → Step 2 with modification brief (source text + change instructions) |
| **new** | → Step 2 with continuity anchors (adjacent source chapter excerpts) |

#### Step 1f-P: Passthrough — Copy Source Chapter

1. Read `{project}/source/chNN_source.md` (the source chapter file listed in the outline)
2. Copy to `{project}/draft/chNN.md`
3. **Skip Steps 2–3 entirely** — no subagent needed
4. Run a lightweight proof-reader pass (Step 4) with **glossary alignment only**:
   - Check proper nouns against `TRANSLATION_GLOSSARY.md`
   - If mismatches found, flag them but do NOT edit the source text — instead note:
     ```
     ⚠️ Passthrough Ch.[N]: glossary mismatch found — [details].
     Source text is preserved as-is. Consider upgrading to "modify" if alignment is required.
     ```
5. → Jump to Step 6 (Update State), recording `"type": "passthrough"` in state

### Step 2: Editor Pre-Processing (Before Delegation)

> **This step is the editor's sole responsibility.** You handle administrative
> prep so the subagent has clean data to read. You do NOT extract beats, style
> settings, glossary, or chapter content — the writer reads those herself.

#### 2a: New Character Gate

Skim the chapter's section in `{project}/OUTLINE.md` to identify:
- **Characters present** in this chapter
- **New characters** not yet in CHARACTER_BIBLE

If all characters have existing entries → skip to 2c.
If any character is missing → go to 2b before proceeding.

#### 2b: New Character Stub (if needed)

If a new **supporting or major** character appears for the first time:

1. Run `/character-design` to create a stub in CHARACTER_BIBLE.md **BEFORE** delegating writing
2. See `/character-design` for stub format and minimum required fields

#### 2c: Resolve Writer Persona

1. Read `{project}/settings/active_writer.json` → get `writer_dir`
2. Resolve absolute path to the writer directory (e.g., `writer_persona/Elie/`)
3. Confirm the directory contains: `persona.yaml`, `checklist.md`, `memo.md`

### Step 3: Delegate Writing to Subagent — Load Instructions Pattern

> **This is the handoff boundary.** After Step 3c, control passes to the
> writer (subagent), who executes SUBAGENT_TEMPLATE Steps 1–9 independently.
> You resume at Step 3d when the subagent returns.
>
> **Core principle: you don't pre-read, don't summarize, don't inject file contents.**
> The subagent inherits all tools (Read, Write, Edit, Glob, Grep, Bash) and reads files itself.

#### 3a: Fill Template Slots (metadata only)

Read `.claude/SUBAGENT_TEMPLATE.md` and fill **parameter slots** — these are
pure metadata, not file contents:

| Slot | Source | Example |
|------|--------|---------|
| `{{CHAPTER_NUMBER}}` | Current chapter | `19` |
| `{{CHAPTER_TITLE}}` | From OUTLINE.md | `家庭課題` |
| `{{PREV_CHAPTER}}` | Previous chapter number | `18` |
| `{{WORD_COUNT_TARGET}}` | STYLE_SETTING or NOVEL_STATE | `8000–12000` |
| `{{PROJECT_DIR}}` | Active project path | `novel-pkm` |
| `{{WRITER_DIR}}` | From active_writer.json | `writer_persona/Elie` |
| `{{OUTPUT_MODE}}` | From NOVEL_STATE.json `journal_mode` | See template appendix: "full" / "brief" / "off" |
| `{{SPECIAL_NOTES}}` | Your editorial notes | See 3b |

#### 3b: Compose SPECIAL_NOTES (the only editorial judgment you inject)

This is the **sole place** where your editorial voice enters the subagent's
prompt. Everything else is paths and numbers.

Sources:
- **Prior proof-reader warnings**: "ch18 had 90 dashes — be vigilant"
- **Fusion-mode context**: "TYPE: new, between passthrough ch02 and ch03"
- **Scene-specific direction**: "亚欧 appears for the first time since ch11 : re-read his CHARACTER_BIBLE entry"
- **Editor notes log**: patterns from `{project}/draft/_editor_notes.md` (if exists)

**Keep under ~300 words.** You're giving editorial direction, not data.

#### 3c: Spawn Subagent

```
Agent({
  description: "艾莉 writes ch[N]",
  model: "opus",
  prompt: [filled template string]
})
```

**What happens inside the subagent** (SUBAGENT_TEMPLATE Steps 1–9):

| Template Step | What the writer does | Reads/Writes |
|---------------|---------------------|--------------|
| 1. Load Identity | Read persona.yaml | `{WRITER_DIR}/persona.yaml` |
| 2. Load Settings | Read style + language config | `{project}/settings/STYLE_SETTING.json`, `LANGUAGE_SETTING.json` |
| 3. Load Memory | Read craft memory + quality gates | `{WRITER_DIR}/memo.md`, `checklist.md` |
| 4. Load Context | Read outline, bible, glossary, timeline, continuity map, prev chapter ending | Multiple files in `{project}/` |
| 5. Think | Structured `<thinking>` (7-part, ~800–1500 words, in persona voice) | — |
| 6. Write Chapter | Save chapter prose to disk | → `{project}/draft/chNN.md` |
| 7. Write Notes | Save journal + handoff notes to disk | → `{project}/draft/chNN_notes.md` |
| 8. Update Memo | Append craft insights (if any) to memo | → `{WRITER_DIR}/memo.md` |
| 9. Return Message | Lean status (paths, word count, handoff, self-check) | — |

You do NOT supervise these steps. The writer operates independently.

#### 3d: Receive and Verify

The subagent's return message contains **only metadata** — no chapter text.

1. Verify files exist on disk:
   - `{project}/draft/chNN.md` (chapter text)
   - `{project}/draft/chNN_notes.md` (journal + handoff)
2. Skim the handoff notes for state-update information
3. Skim any new memo.md entries to understand what the writer learned
4. Proceed to Step 4 (proof-reader) — **you resume control here**

### Step 4: Proof-Reader Review — MANDATORY

> **⚠️ THIS STEP IS NOT OPTIONAL.** After every subagent draft, you MUST invoke the `/proof-reader` skill before proceeding. Do not skip this step. Do not "mentally review" the draft yourself — use the skill.

#### 4a: Invoke /proof-reader

**Immediately after receiving and processing the subagent output (Step 3c), invoke the proof-reader:**

```
Skill({ skill: "proof-reader", args: "ch[NN]" })
```

The proof-reader checks six categories:
1. **Character consistency** — speech fingerprints, identity anchors, knowledge state, relationships
2. **Timeline consistency** — event order, location continuity, story-day tracking
3. **Language quality** — repetitive phrases, awkward phrasing, prose rhythm, style compliance
4. **Language contamination** — grammar pattern leakage, script mixing, translationese
5. **Dialogue format compliance** — bilingual pairs complete, correct brackets, format consistency
6. **Glossary alignment** — proper noun consistency, missing terms flagged

It returns a structured report with a verdict. See `/proof-reader` for the full check protocol.

**Fusion mode: additional checks by chapter type:**

| Chapter type | Extra checks |
|-------------|-------------|
| **new** | 7. **Continuity seams** — does the opening connect to the preceding source chapter? Does the closing lead into the following source chapter? Any contradiction with source novel events? |
| **modify** | 7. **Modification completeness** — were ALL listed modifications applied? 8. **Preservation check** — is preserved content intact and unaltered? 9. **Style match** — does the rewritten prose match the source author's voice, not the writer persona? |
| **passthrough** | Glossary alignment only (handled in Step 1f-P) |

#### 4b: Read the Verdict

Read the proof-reader's report and act on the verdict:

- **PASS** (0 HIGH, ≤2 MEDIUM) → proceed to Step 5
- **PASS_WITH_NOTES** (0 HIGH, >2 MEDIUM) → optionally fix MEDIUM issues, then proceed
- **NEEDS_REVISION** (≥1 HIGH) → fix and re-review (see Step 4c)

#### 4c: Editor Fixes (if NEEDS_REVISION)

**You (the editor) fix all issues directly.** Do not re-spawn a subagent for revision — it costs a full context re-transmission and is slow. You have the draft open; use Edit tool to fix it yourself.

Typical fixes:
- Wrong character name / glossary mismatch → find-and-replace
- Missing translation line → insert the pair
- Speech fingerprint violation → rewrite the dialogue line(s)
- Timeline contradiction → adjust the reference
- Flat prose / AI boilerplate → rewrite the offending passage
- Missing plot beat → insert a bridging paragraph or scene

**After fixes, re-invoke `/proof-reader`:**
```
Skill({ skill: "proof-reader", args: "ch[NN]" })
```

**Maximum 2 review cycles.** If still NEEDS_REVISION after 2 rounds, save the best version and flag:
```
⚠️ Chapter [N] saved with [N] unresolved HIGH issues after 2 revision cycles.
Manual review recommended: [list issues]
```

#### 4d: Writer Persona Fidelity Check

After proof-reader verdict is PASS or PASS_WITH_NOTES (or after final revision cycle):

1. Check if `writer_persona/{persona_name}/checklist.md` exists for the active persona
2. If yes → read the checklist, evaluate the draft against each item
3. Append a `## Persona Fidelity` section to the chapter's review notes
4. If ≥3 items fail → treat as HIGH, apply fixes (targeted edits to prose style, not full rewrite)
5. If 1–2 items fail → note as MEDIUM, optionally fix

See `system-prompt.md` → "Writer Persona Fidelity Check" for full protocol.

### Step 5: Save Final Draft

After PASS or PASS_WITH_NOTES:

**Monolingual / Dialogue-bilingual:**
- Save to `{project}/draft/chNN.md` (already saved by subagent; verify final version is in place after any edits)

**Bilingual (per-chapter translation):**
- Primary already saved by subagent
- Run translation step (Step 5a)

#### Step 5a: Translation (Bilingual Mode Only)

For full bilingual mode with `translation_timing: per-chapter`:

1. Load glossary and translation style from LANGUAGE_SETTING.json
2. Pre-scan for proper nouns — check each against glossary
3. Translate following the configured style (literary / literal / faithful)
4. Glossary terms are non-negotiable (The Banana Rule)
5. Save translation to `{project}/draft/[secondary-code]/chNN_[code].md`
6. Append new unregistered terms to glossary's 待确认 section

### Step 6: Update State

Update `{project}/NOVEL_STATE.json`:
- Increment `chapters_written`
- Update `last_chapter`
- Record word count
- Record proof-reader verdict
- Log any new characters added
- Log any character knowledge state updates
- **Fusion mode**: record `"chapter_type": "passthrough" / "new" / "modify"` per chapter

Update `{project}/settings/TIMELINE.md` and `{project}/settings/CONTINUITY_MAP.md`:

**TIMELINE = The Clock** — tracks WHEN and WHAT HAPPENED (events, not states):
- Append this chapter's events with story-day/age references
- Ensure time continuity with previous chapter
- Update Pokémon evolution progress table if any evolutions occurred
- **Do NOT track**: character emotional/relationship states, item possession, writing transition notes
- **Fusion passthrough**: import timeline events from the source chapter as-is
- **Fusion modify**: update timeline entries to reflect modifications

**CONTINUITY_MAP = The Snapshot** — tracks WORLD STATE at chapter boundaries (states, not events):
- Update the chapter-boundary snapshot: character states (physical, emotional, knowledge), relationship dynamics, item/artifact tracking, unresolved plot threads
- Include writing transition notes for the next chapter's subagent
- **Do NOT track**: specific events (those go in TIMELINE), time positioning, Pokémon evolution history
- **Fusion passthrough**: import state snapshot from source chapter
- **Fusion new/modify**: update with new facts introduced

**Rule: TIMELINE never tracks state. CONTINUITY_MAP never tracks events or time. If uncertain where something goes — events are "X happened", states are "X is now Y".**

### Step 7: Progress Report

After each chapter:

```
Chapter [N] complete.

Title: [chapter title]
[Fusion type: passthrough / new / modify]           (fusion mode only)
Words: [actual count] / [target]  ([+/- %])
[Words: [count] (source, unchanged)]                 (passthrough only)
POV: [character]
New characters added: [names or "none"]
Proof-reader: [PASS / PASS_WITH_NOTES / NEEDS_REVISION→fixed]
  [If PASS_WITH_NOTES: brief list of noted issues]
[Language mode: dialogue-bilingual (ja→zh inline)]
[Glossary: [N] new terms appended to 待確認]
[Modifications applied: [list]]                      (modify only)
[Continuity seams: OK / ⚠️ [issue]]                  (new/modify only)

Files:
- {project}/draft/chNN.md
[- {project}/draft/en/chNN_en.md]  (bilingual only)
[- {project}/source/chNN_source.md → copied]         (passthrough only)

Progress: [N] / [total] chapters ([N] passthrough, [N] new, [N] modify)
Estimated total words: [sum]

Next: Chapter [N+1] — [brief beat from outline]
Run /novel-write to continue, or /novel-export when ready.
```

## Batch Writing Mode

When writing multiple chapters (`all` or a range):

- Write chapters sequentially (each depends on the previous)
- Full workflow (Steps 1–7) for each chapter — do not skip proof-reader
- After every chapter, save state before starting the next
- Present brief progress after each chapter, full summary at end

**Fusion mode batch optimization:**
- Passthrough chapters are fast (file copy + glossary check) — process them without pause
- Group consecutive passthrough chapters for rapid processing, then pause to report before the next new/modify chapter

**State tracking (all modes):**
- Update TIMELINE.md and CONTINUITY_MAP.md after EVERY chapter (Step 6) — not just fusion mode
- CONTINUITY_MAP is the subagent's primary reference for "what does the world look like entering this chapter"

**Context management for long novels** (chapter 10+):
- Orchestrator reads minimal context (NOVEL_STATE, outline section, editor notes)
- Subagent reads full files itself — its context is independent of yours
- This keeps your context clean for proof-reading and multi-chapter orchestration

## Restyle Mode (`--restyle`)

When called with `--restyle`:
1. Read the existing chapter
2. Apply current STYLE_SETTING.json
3. Preserve all plot events and character information
4. Delegate rewrite to subagent with explicit "preserve content, restyle prose" instructions
5. Run proof-reader on the restyled version
6. Save timestamped backup before overwriting: `{project}/draft/chNN_backup_YYYYMMDD_HHmmss.md`

## Dialogue-Bilingual: Detailed Format Reference

When `LANGUAGE_SETTING.json` has `mode: "dialogue-bilingual"`:

### What counts as "dialogue"
- Direct speech marked with dialogue brackets (「」or configured `original_wrapper`)
- Quoted internal monologue that uses dialogue markers
- NOT: narration, description, unquoted thoughts, author commentary

### Format for each dialogue occurrence

```markdown
[writing-language narration]

「dialogue-language line」
（writing-language translation）

[more writing-language narration]
```

### Multi-turn dialogue

```markdown
「Speaker A's line in dialogue-language」
（Speaker A's translation）

诺德摇了摇头。

「Speaker B's response in dialogue-language」
（Speaker B's translation）
```

### Long dialogue with narration tag mid-sentence

```markdown

Split into two pairs if the interruption is substantial:

```markdown
「First part of dialogue」
（第一部分翻译）

他停下来，整理了一下思绪。

「Second part of dialogue」
（第二部分翻译）
```

### Shouting / whispering / special delivery

Narration carries the delivery — the dialogue itself stays clean:

```markdown
诺德猛然提高声量。

「全員、退避しろ！」
（所有人，撤退！）
```

## Fusion Mode: Complete Workflow Reference

**Activated when** `NOVEL_STATE.json` has `"mode": "fusion"`. Generated by `/novel-fusion`.

### Quick Reference: Per-Chapter-Type Routing

```
For each chapter in OUTLINE.md:
  │
  ├─ TYPE: passthrough
  │    1. Read source file from {project}/source/chNN_source.md
  │    2. Copy to {project}/draft/chNN.md
  │    3. Glossary-only proof-reader check (flag but don't edit)
  │    4. Update state → next chapter
  │
  ├─ TYPE: new
  │    1. Load context (Step 1) + continuity anchors from adjacent source chapters
  │    2. Assemble brief with fusion additions (Step 2d-F: new)
  │    3. Spawn subagent (Step 3) — instruct: "seamless integration with source text"
  │    4. Proof-reader: standard 6 checks + continuity seams check
  │    5. Save, update state → next chapter
  │
  └─ TYPE: modify
       1. Load context (Step 1) + full source chapter text + modification instructions
       2. Assemble brief with fusion additions (Step 2d-F: modify)
       3. Spawn subagent (Step 3) — instruct: "match source author's voice, apply changes"
       4. Proof-reader: standard 6 checks + modification completeness + preservation check + style match
       5. Save, update state → next chapter
```

### Fusion-Specific Rules

- **Never edit passthrough chapters.** Source text is sacred. If glossary mismatches are found, flag them and suggest upgrading to "modify" — do not auto-fix.
- **Modify chapters must match the source voice.** The writer persona is overridden by the source novel's prose style. The subagent brief must include explicit instruction: "Match the source author's style, not your default persona."
- **Continuity is the core challenge.** Every new chapter's seams (opening and closing) are HIGH-severity proof-reader checks. A new chapter that doesn't connect to adjacent source text is a failure.
- **Flag downstream risks.** If a new or modified chapter introduces facts that contradict a later passthrough chapter, flag it immediately:
  ```
  ⚠️ Continuity risk: Ch.[N] (new) establishes [fact], but passthrough Ch.[M] assumes [contradicting fact].
  Consider upgrading Ch.[M] from passthrough to modify.
  ```
- **Source files are read-only.** All writes go to `{project}/draft/`. Never edit files in `{project}/source/`.

## Key Rules

- **Large file handling**: if Write fails, retry with Bash heredoc. Do not ask for permission.
- **Never invent plot events** not in the outline without noting: "Bridged outline gap at Chapter N with [scene description]."
- **Never skip the proof-reader.** After EVERY subagent draft, invoke `Skill({ skill: "proof-reader", args: "ch[NN]" })`. A chapter with a character contradiction is worse than a slightly shorter chapter. 
- **Word count is a target**: do not pad prose. End the chapter when the scene is complete.
- **Speech fingerprints are non-negotiable**: if a character's voice is wrong, fix the dialogue before saving.
- **Glossary terms are non-negotiable**: registered translations cannot be varied for style.
- **Bilingual translation happens after the primary draft is complete** — never translate a half-written chapter.
- **The subagent writes; you review.** Do not write the chapter yourself except for targeted edits during revision.
- **The subagent writes files directly.** Chapter text and notes are saved to disk by the subagent — you receive only a lean status message. If files are missing after subagent returns, something went wrong.
- **The subagent updates its own memo.** After writing, the subagent appends craft insights to `writer_persona/{name}/memo.md`. Skim the new entries during your review to understand what it learned.
- **Proof-reader maximum 2 cycles.** Don't loop endlessly: save the best version and flag remaining issues.
