---
name: character-design
description: "Design and document novel characters from a story outline. Creates a CHARACTER_BIBLE.md with appearance, personality, speech patterns, backstory, and relationships. Automatically adds new supporting/major characters as they emerge during writing. Use when user says \"设计角色\", \"character design\", \"人物设定\", \"写人物\", or wants to build a character roster from an outline."
argument-hint: [outline-file-or-description]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob
---

# Character Design: Build the Character Bible

Design characters for: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

This skill reads a story outline and produces a structured `CHARACTER_BIBLE.md` — the single source of truth for every named character in the novel. All later writing phases read this file to ensure consistency.

The character bible is a living document: new characters are added as they appear during writing, and existing profiles are deepened as more story details emerge.

## Inputs

1. **`{project}/OUTLINE.md`** — story outline (preferred source)
2. **`$ARGUMENTS`** — outline text pasted inline, or path to a different outline file (`.md`, `.txt`, `.pdf`)
3. **`{project}/characters/CHARACTER_BIBLE.md`** — existing bible (if updating, not starting fresh)

If no outline is found, ask: "Please provide a story outline (file path or paste the text)."

### PDF Input Handling

If the input is a `.pdf` file, use the Read tool with multimodal mode (the tool renders both text and embedded images):

```
Read(file_path="XXX.pdf")                        # reads up to 2000 lines / ~20 pages
Read(file_path="XXX.pdf", pages="1-10")          # for large PDFs, read in page-range batches
```

**Text content**: extracted directly — treat as standard outline text.

**Embedded images** (character illustrations, maps, world-setting diagrams): the Read tool will describe what it sees. Use these descriptions to enrich character profiles:
- Character illustration → fill in the Appearance section (height/build, hair, eye color, distinctive features, typical clothing)
- Map / world diagram → add a "World Notes" entry in the CHARACTER_BIBLE for location-based context
- Table / chart / timeline → extract any named entities and incorporate into relevant profiles

**Large PDFs**: if the PDF exceeds 20 pages, read in batches:
```
Read(file_path="XXX.pdf", pages="1-10")
Read(file_path="XXX.pdf", pages="11-20")
...
```
Merge all extracted content before proceeding to Step 1.

**Image-heavy pages with little text**: if a page is mostly a character illustration, explicitly note the visual details extracted:
> "Page 7: character illustration — tall male figure, silver hair tied back, mechanical left arm, dark military coat with gold trim → added to 艾登's Appearance section."

## State Check

Before starting, check if `{project}/characters/CHARACTER_BIBLE.md` already exists:

- If absent → **fresh design** (Step 1 → Step 5)
- If present and `$ARGUMENTS` mentions a character name or "add" → **add/update mode** (jump to Step 4)
- If present and `$ARGUMENTS` mentions "refresh" or "rebuild" → **full rebuild** (archive old bible first, then Step 1)

Archive command:
```bash
cp {project}/characters/CHARACTER_BIBLE.md \
   "{project}/characters/CHARACTER_BIBLE_$(date +%Y%m%d_%H%M%S).md"
```

## Workflow

### Step 1: Parse the Outline

Read the outline carefully and extract:

1. **All named characters** — any name that appears more than once, or once with a role description
2. **Character roles** — protagonist, antagonist, supporting, minor
3. **Relationships** — who knows whom, alliances, conflicts, romance
4. **Story beats** — what each character does at major turning points
5. **Setting clues** — time period, cultural context (affects speech register and clothing)

Create an extraction table:

```
Character | First mention | Role | Known traits | Relationships
---------|---------------|------|--------------|---------------
[Name]   | Ch N / Scene  | ...  | ...          | ...
```

Classify each character:
- **Major**: protagonist, deuteragonist, primary antagonist — write full profiles
- **Supporting**: recurring characters with dialogue and plot influence — write standard profiles
- **Minor**: one-scene characters, no dialogue — list name + one-line description only, do NOT write full profiles

### Step 2: Research Setting for Authenticity

For each major/supporting character, consider the story's time period and cultural setting:

- **Contemporary**: modern speech, contemporary fashion references
- **Historical**: period-appropriate vocabulary register (avoid anachronisms)
- **Fantasy/Sci-Fi**: invent consistent linguistic register that fits the world-building

Note any cultural specifics (e.g., Chinese honorifics, regional dialects, formal/informal registers) that should be reflected in speech patterns.

### Step 3: Build Full Profiles

For each **Major** character, write a complete profile using this schema:

```markdown
## [Character Name]

**Role**: [Protagonist / Antagonist / Supporting / Minor]
**First appears**: Chapter [N]
**Status**: [Active / Deceased / Absent / Unknown]

### Identity
- Full name: [family name + given name, with romanization if applicable]
- Age at story start: [N years old]
- Pronouns: [he/him / she/her / they/them]

### Appearance
- Height / build: [e.g., 175 cm, lean athletic build]
- Distinctive features: [scars, eye color, hair, anything plot-relevant]
- Typical clothing: [era-appropriate, reflects personality and status]

### Personality Core
Three traits that never change (anchor points for consistency checks):
1. [Trait — e.g., "Stubborn loyalty: will not abandon allies even at personal cost"]
2. [Trait — e.g., "Dry humor: deflects emotion with understated sarcasm"]
3. [Trait — e.g., "Perfectionism: secretly redoes others' work if it falls short of her standard"]

### Speech Fingerprint
- Vocabulary: [formal / colloquial / mixed / regional dialect / invented register]
- Sentence style: [terse / verbose / clipped / flowing / fragmented under stress]
- Pet phrases: [e.g., "不是我说" / "Well, technically..." / specific filler words]
- Would never say: [words or phrases inconsistent with this character]
- Emotional register: [How they express anger / affection / fear / grief]
- Dialogue example (2-3 lines demonstrating the fingerprint):
  > "[Sample dialogue in this character's voice]"

### Backstory (story-relevant only)
[Only what the reader needs to understand the character's motivations. Avoid backstory not referenced by the plot.]

### Relationships
| Character | Relationship | Notes |
|-----------|--------------|-------|
| [Name] | [e.g., estranged brother] | [context] |

### Story Arc
- Goal: [what they want at story start]
- Internal conflict: [what they struggle with internally]
- Key turning points:
  - Chapter [N]: [event that changes them]
  - Chapter [N]: [decision point]

### Knowledge State
- What they know at story start: [key information they hold]
- Learns [X]: Chapter [N] (fill in as story develops)
```

For each **Supporting** character, write a shorter profile (Identity + Appearance + Personality Core + Speech Fingerprint + Relationships — skip detailed arc and knowledge state unless plot-critical).

### Step 4: Add/Update Characters

When called mid-writing to add a new character:

1. Read the context provided in `$ARGUMENTS` (chapter number, how they appear, what role they play)
2. Create a stub profile immediately with at minimum:
   - Name, role, first-appearance chapter
   - One physical anchor
   - One personality anchor
   - Speech fingerprint (even if brief)
3. Append the stub to `CHARACTER_BIBLE.md` under the appropriate role section
4. Present the stub to the user for feedback before writing the character's first real dialogue scene

For updates to existing characters (new information revealed by the plot):
- Append to the "Knowledge State" section
- Add new relationships as they form
- Update "Story Arc" with new turning points
- Never delete old entries — use ~~strikethrough~~ with a date note for retcons

### Step 5: Write the CHARACTER_BIBLE.md

Structure the file:

```markdown
# Character Bible

> Generated by /character-design from [OUTLINE source] on [DATE]
> Novel: [Title if known]
> Last updated: [DATE] (Chapter [N] reached)

---

## Major Characters

[Full profiles for protagonist, antagonist, and primary supporting cast]

---

## Supporting Characters

[Standard profiles]

---

## Minor Characters

| Name | First appears | Role | One-line description |
|------|---------------|------|---------------------|
| [Name] | Ch [N] | [role] | [description] |

---

## Relationship Map

[Text diagram or table showing key relationships between major characters]

Example:
主角A ←── 青梅竹马 ──→ 角色B
   ↓                        ↑
  仇人 ──────────────── 角色C

---

## World Notes (Character-Relevant)
[Cultural context, naming conventions, honorifics, dialect notes that affect all characters]
```

Save with versioning:
```bash
# Write timestamped version first
cp {project}/characters/CHARACTER_BIBLE.md \
   "{project}/characters/CHARACTER_BIBLE_$(date +%Y%m%d_%H%M%S).md" 2>/dev/null || true
# Then write the current version
```

### Step 6: Present Summary

After writing the bible, present:

```
Character Bible complete.

Characters designed:
- Major: [N] ([names])
- Supporting: [N] ([names])
- Minor: [N]

Key relationships:
- [Brief summary of the main relationship dynamics]

Potential consistency watch-points:
- [Any characters with similar names that could be confused]
- [Characters with complex role shifts across the story]

Output: {project}/characters/CHARACTER_BIBLE.md

Next steps:
- /novel-style   → configure writing style and chapter length
- /language-setting → configure language / bilingual mode
- /novel-write   → start writing Chapter 1
```

## Consistency Rules

Read `../shared-references/character-consistency.md` before writing any character profile. Apply the character schema exactly as specified there.

## Key Rules

- **Large file handling**: If the Write tool fails due to file size, immediately retry using Bash (`cat << 'EOF' > file`) in chunks. Do NOT ask the user for permission.
- **Never invent plot details** not present in the outline. If the outline is vague about a character, flag it: "⚠️ [Name]'s personality is not specified in the outline — I've inferred [X] from context. Confirm or adjust."
- **Speech fingerprint before dialogue**: never write a character's first dialogue without their speech fingerprint being registered in the bible.
- **Versioning**: always write a timestamped archive copy before overwriting the existing CHARACTER_BIBLE.md.
- **Minor character threshold**: if a character appears in only one scene with no dialogue and no effect on the plot, they are minor — one-line table entry only.
