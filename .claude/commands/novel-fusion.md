---
name: novel-fusion
description: "Fuse new content into an existing novel. Reads a source novel + a fusion outline that specifies new chapters (before/after existing ones) and optional modifications. Generates a structured OUTLINE.md with three chapter types: passthrough (use source as-is), modify (rewrite with instructions), and new (original content). Use when user says \"融合小说\", \"扩写小说\", \"novel fusion\", \"expand novel\", \"insert chapters\", \"补写\", or wants to weave new content into an existing work."
argument-hint: [source-novel-path fusion-outline-path — or inline instructions]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent
---

# Novel Fusion: Expand & Weave Into an Existing Novel

Fusion target: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

This skill takes an **existing novel** and a **fusion outline** describing where to insert new content, then produces a structured OUTLINE.md that the regular `/novel-write` pipeline can consume. Each chapter in the final outline is typed:

| Type | Meaning | What `/novel-write` does |
|------|---------|--------------------------|
| **passthrough** | Use source novel chapter as-is | Copy source text to draft — no writing |
| **modify** | Rewrite source chapter with specific changes | Subagent receives source text + modification instructions |
| **new** | Original content not in source | Subagent writes from beats, using adjacent source chapters as context |

**Use cases:**
- Expand a novel with new scenes between existing chapters
- Fill in events only briefly mentioned ("what happened between Ch.3 and Ch.4")
- Add a subplot that weaves through the existing structure
- Rewrite specific chapters to incorporate new plotlines
- Write prequels/sequels that connect to the existing text

## Input

### 1. Source Novel

The existing novel — accepted formats:

| Format | Handling |
|--------|----------|
| Single file (`.md`, `.txt`) | Auto-split into chapters by heading detection (`# Chapter`, `# 第N章`, `---` scene breaks, etc.) |
| Single file (`.pdf`, `.docx`) | Convert to text first, then split |
| Pre-split chapters (`{project}/source/ch01.md`, etc.) | Read directly |
| Directory of files | Each file = one chapter, sorted by filename |

After reading, save chapter-split source to `{project}/source/`:
```
{project}/source/
  ch01_source.md
  ch02_source.md
  ...
  chNN_source.md
  SOURCE_INDEX.md    ← chapter titles, word counts, brief summaries
```

### 2. Fusion Outline

The user's plan for what to add or change. Can be:
- A separate file (`.md`, `.txt`)
- Inline text in the arguments
- A conversation with the user (the skill asks questions)

**Fusion outline format** — flexible, but must specify insertion points:

```markdown
# Fusion Outline

## Before Ch.1
[Description of new content to insert before the novel begins]

## Ch.1 — keep
(No changes)

## Ch.2 — keep  

## After Ch.2
[Description of new scene/chapter to insert after Ch.2]

## Ch.3 — modify
Changes: [Specific modification instructions — what to add, remove, or rewrite and why]

## Between Ch.5 and Ch.6
[Description of new content — can span multiple new chapters]

## Ch.8 — keep

## After Ch.12 (end)
[Epilogue or sequel content]
```

**Supported directives:**

| Directive | Meaning |
|-----------|---------|
| `Ch.N — keep` / `第N章 保留` | Pass through source chapter unchanged |
| `Ch.N — modify` / `第N章 修改` | Rewrite source chapter with listed changes |
| `Before Ch.N` / `第N章之前` | Insert new content before this chapter |
| `After Ch.N` / `第N章之后` | Insert new content after this chapter |
| `Between Ch.N and Ch.M` / `第N章和第M章之间` | Insert new content in this gap |
| `Replace Ch.N` / `替换第N章` | Discard source chapter, write from scratch |

If no directive is given for a source chapter, it defaults to **keep**.

## Workflow

### Step 0: Parse Arguments

Extract:
1. **Source novel path** — file or directory
2. **Fusion outline** — file, inline, or "ask me"
3. **Target chapter count** — detected from outline or calculated (source chapters + new chapters)

If source novel is missing: ask immediately.
If fusion outline is missing: ask — or offer to enter interactive mode where the user describes insertions one by one.

### Step 1: Read & Split Source Novel

1. Read the source novel
2. Detect chapter boundaries:
   - Heading patterns: `# Chapter N`, `# 第N章`, `## [title]`
   - Separator patterns: `---` or `***` between large text blocks
   - If no clear structure: ask the user how chapters are separated
3. Split into individual chapter files → `{project}/source/chNN_source.md`
4. Generate `{project}/source/SOURCE_INDEX.md`:

```markdown
# Source Novel Index

**Title**: [detected]
**Total chapters**: [N]
**Total words**: ~[N]

| Ch | Title | Words | Summary |
|----|-------|-------|---------|
| 01 | [title] | [N] | [1-line summary] |
| 02 | [title] | [N] | [1-line summary] |
| ... | ... | ... | ... |
```

Present to user for confirmation:
```
Source novel loaded: [title]
Chapters detected: [N]
Total words: ~[N]

[Chapter list with titles]

Correct? (or adjust chapter splits)
```

### Step 2: Parse Fusion Outline

Read the fusion outline and build a **fusion map** — an ordered list of what the final novel looks like:

```
fusion_map = [
  { final_ch: 1,  type: "new",         beats: "...",  context_after: "source ch01" },
  { final_ch: 2,  type: "passthrough",  source_ch: 1 },
  { final_ch: 3,  type: "passthrough",  source_ch: 2 },
  { final_ch: 4,  type: "new",         beats: "...",  context_before: "source ch02", context_after: "source ch03" },
  { final_ch: 5,  type: "modify",      source_ch: 3,  modifications: "..." },
  { final_ch: 6,  type: "passthrough",  source_ch: 4 },
  ...
]
```

**For new chapters:** automatically identify adjacent source chapters that serve as continuity anchors:
- `context_before`: the source chapter immediately before the insertion point (last ~500 chars)
- `context_after`: the source chapter immediately after (first ~500 chars)

**For modify chapters:** read the full source chapter — the subagent needs the complete text to know what to preserve.

Present the fusion map to the user:

```
Fusion map generated:

Final Ch | Type        | Source     | Notes
---------|-------------|-----------|-------
01       | NEW         | —         | [brief description]
02       | passthrough | Source 01 | [source title]
03       | passthrough | Source 02 | [source title]
04       | NEW         | —         | [brief description]  
05       | modify      | Source 03 | Changes: [summary]
06       | passthrough | Source 04 | [source title]
...

Total: [N] chapters ([N] passthrough, [N] new, [N] modify)

Confirm? (or adjust)
```

### Step 3: Generate Structured OUTLINE.md

Produce the outline in the same format as `/novel-outline`, but with chapter type annotations:

```markdown
# [Novel Title] — Fusion Edition

> Source: [source novel] | Fusion outline: [outline file]
> Chapters: [N] total ([N] passthrough, [N] new, [N] modify)
> Generated: [ISO timestamp]

## Story Overview

**Original premise**: [from source novel]
**Fusion additions**: [summary of what the fusion adds]

---

## Chapter 1: [Title] — TYPE: new

**Insertion point**: Before source Chapter 1
**POV**: [character]
**Characters**: [list]
**Word target**: ~[N]

### Plot Beats
1. [Beat 1]
2. [Beat 2]
...

### Continuity Anchors
- **Leads into**: Source Ch.1 — [brief description of what Ch.1 opens with, so the new content connects]

---

## Chapter 2: [Title] — TYPE: passthrough (Source Ch.1)

**Source file**: `{project}/source/ch01_source.md`
**Words**: [N] (source length)

> This chapter uses the source novel text directly. No writing needed.
> `/novel-write` will copy the source file to the draft.

---

## Chapter 4: [Title] — TYPE: new

**Insertion point**: After source Chapter 2, before source Chapter 3
**POV**: [character]
**Characters**: [list]
**Word target**: ~[N]

### Plot Beats
1. [Beat 1]
2. [Beat 2]
...

### Continuity Anchors
- **Follows from**: Source Ch.2 — [how Ch.2 ends, what state characters are in]
- **Leads into**: Source Ch.3 — [what Ch.3 opens with, what the new chapter must set up]

---

## Chapter 5: [Title] — TYPE: modify (Source Ch.3)

**Source file**: `{project}/source/ch03_source.md`
**Words**: [N] (source) → ~[N] (target after modifications)

### Modification Instructions
- [Specific change 1: what to add/remove/rewrite and why]
- [Specific change 2]
- [What to preserve: key scenes, dialogue, plot points that must stay]

### Continuity Notes
- Must remain consistent with new Chapter 4 (which precedes this)
- [Any new information introduced in new chapters that this chapter must acknowledge]

---

## Chapter 6: [Title] — TYPE: passthrough (Source Ch.4)
...
```

### Step 4: Extract Characters

Scan both the source novel and the fusion outline for characters:

1. Characters from source novel → extract profiles (same as `/novel-outline`)
2. New characters introduced in fusion outline → create stubs
3. Merge into `{project}/characters/CHARACTER_BIBLE.md`

For source novel characters, pay special attention to:
- Speech patterns (extracted from actual dialogue in the source)
- Relationship states at each chapter boundary (important for new chapters that splice in)

### Step 5: Generate Continuity Map

Create `{project}/settings/CONTINUITY_MAP.md` — a reference for `/novel-write` that tracks the state of the world at each chapter boundary:

```markdown
# Continuity Map

## After Source Ch.1 / Before Final Ch.3
- Location: [where characters are]
- Character states: [emotional, physical, knowledge]
- Unresolved threads: [what's hanging]
- Time: [story time]

## After Final Ch.4 (NEW) / Before Source Ch.3 (Final Ch.5 MODIFY)
- Location: [where the new chapter leaves characters]
- New information introduced: [what the new chapter established]
- What Source Ch.3 must now account for: [adjustments needed]
...
```

This is critical for modify chapters — the subagent needs to know what new content has been inserted around the source chapter it's rewriting.

### Step 6: Update State

Write `{project}/NOVEL_STATE.json`:

```json
{
  "title": "[title]",
  "mode": "fusion",
  "source_novel": "[source path]",
  "fusion_outline": "[outline path]",
  "total_chapters": 15,
  "passthrough_chapters": 8,
  "new_chapters": 5,
  "modify_chapters": 2,
  "phase": 0,
  "status": "outline_complete",
  "timestamp": "[ISO timestamp]"
}
```

### Step 7: Report

```
Novel fusion outline complete.

Source: [novel title] ([N] chapters, ~[N] words)
Fusion result: [N] total chapters
  - [N] passthrough (source as-is)
  - [N] new (original content)
  - [N] modify (source with changes)

Files created:
- {project}/source/          ← [N] source chapter files
- {project}/source/SOURCE_INDEX.md
- {project}/OUTLINE.md       ← fusion outline with chapter types
- {project}/characters/CHARACTER_BIBLE.md
- {project}/settings/CONTINUITY_MAP.md

Next steps:
- Review the outline — especially continuity anchors for new chapters
- Review modification instructions for modify chapters
- /language-setting → configure language
- /novel-style → configure style
- /novel-write all → writes new chapters, copies passthrough, rewrites modify
```

## How /novel-write Handles Fusion Chapters

When `/novel-write` encounters a fusion outline, it reads the `TYPE:` annotation on each chapter:

### passthrough
1. Read `{project}/source/chNN_source.md`
2. Copy to `{project}/draft/chNN.md`
3. No subagent needed — just file copy
4. Still run proof-reader for glossary alignment (proper nouns may need to match the project's glossary)

### new
1. Assemble brief as normal, but add:
   - **Continuity anchors** from the outline (adjacent source chapter excerpts)
   - **Continuity map** entry for this insertion point
2. Subagent writes the chapter with explicit instruction: "This chapter must seamlessly connect with the source text before and after it"
3. Proof-reader checks continuity with adjacent source chapters in addition to standard checks

### modify
1. Assemble brief with:
   - Full source chapter text
   - Modification instructions from outline
   - Continuity map (what new chapters before/after this one have established)
   - Explicit "preserve" list (scenes/dialogue that must stay)
2. Subagent rewrites the chapter, applying modifications while preserving everything not listed for change
3. Proof-reader checks:
   - All modifications were applied
   - Preserved content is intact
   - Continuity with new/modified adjacent chapters
   - No contradictions introduced

## Edge Cases

### Source novel has no clear chapter structure
Ask the user to specify break points, or offer to split by:
- Every N words
- Scene breaks (`---`)
- Manual markers the user adds

### Fusion outline refers to chapters that don't exist
If the user says "After Ch.15" but the source only has 12 chapters:
- Warn and ask for clarification
- If they mean "after the end", treat as an epilogue/sequel section

### Multiple new chapters at one insertion point
"Between Ch.3 and Ch.4, I want 3 new chapters" — supported. The fusion map creates three sequential `new` entries at that point, and the outline gives each its own beats.

### Modify chapter becomes too different from source
If modification instructions essentially rewrite >80% of a chapter, suggest reclassifying as `replace` (TYPE: new, with source as reference material rather than base text). This is cleaner for the subagent.

### Source novel is in a different language than the writing language
Handle like `/novel-outline`: the structured outline and continuity anchors are in the writing language. Passthrough chapters remain in their original language. If the user wants everything in one language, passthrough chapters become modify chapters with "translate" as the modification instruction.

## Key Rules

- **Never modify passthrough chapters.** If the outline says "keep", it's keep. The source text is sacred unless explicitly marked for modification.
- **Continuity is the core challenge.** Every new chapter must read as if it was always part of the novel. The continuity anchors and continuity map are the primary tools for this.
- **Preserve the source author's voice in modify chapters.** Modifications should feel like edits by the same author, not a different writer grafting onto the text. The subagent should match the source's prose style for modify chapters (even overriding the writer persona if needed).
- **Flag continuity risks.** If a new chapter introduces information that contradicts a downstream passthrough chapter, flag it — the user may need to upgrade that passthrough to a modify.
- **Source files are read-only.** Never edit files in `{project}/source/`. All changes go to `{project}/draft/`.
