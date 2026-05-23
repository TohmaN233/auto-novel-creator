---
name: proof-reader
description: "Review novel chapter drafts for quality and consistency. Checks character bible conflicts, timeline errors, repetitive/awkward language (车轱辘话), language contamination, bilingual dialogue compliance, and glossary alignment. Use when user says \"审稿\", \"校对章节\", \"proof-read\", \"review chapter\", \"检查章节\", or automatically invoked by the writing workflow after a subagent draft."
argument-hint: [chapter-number(s) or range — e.g., "ch01" / "ch03-ch05" / "all"]
allowed-tools: Read, Grep, Glob
---

# Proof-Reader: Chapter Quality Review

Review: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

Review novel chapter drafts against the project's reference files and produce a structured quality report. Designed for two invocation paths:

1. **Automated** — called by the writing workflow (main agent) after a writing subagent produces a draft
2. **Manual** — called directly by the user to review existing chapters

The proof-reader is **read-only**: it identifies issues and returns a structured report. The caller decides how to fix.

## Load Reference Files

Before reviewing any chapter, load:

| File | Required | Purpose |
|------|----------|---------|
| `{project}/characters/CHARACTER_BIBLE.md` | Yes | Character identity, speech fingerprints, relationships, knowledge state |
| `{project}/settings/TIMELINE.md` | If exists | Event ordering, story-day tracking |
| `{project}/settings/LANGUAGE_SETTING.json` | Yes | Language mode — determines which checks apply |
| `{project}/settings/TRANSLATION_GLOSSARY.md` | If exists | Proper noun mappings (required for bilingual/dialogue-bilingual) |
| `{project}/settings/STYLE_SETTING.json` | Yes | Style targets, custom style file path |
| `{project}/OUTLINE.md` | Yes | Plot beats for the chapter under review |
| Previous chapter (last 300 words) | If exists | Continuity check |

Also load the custom style file if `STYLE_SETTING.json` specifies one — the proof-reader must know the style rules to avoid false positives.

## Argument Parsing

| Argument | Meaning |
|----------|---------|
| `ch01` / `第一章` / `1` | Review Chapter 1 |
| `ch03-ch05` / `3-5` | Review Chapters 3–5 |
| `all` / `全部` | Review all existing draft chapters |
| _(raw text)_ | If called by the writing workflow, the argument may be the draft text itself + chapter metadata — accept and review inline |

Locate draft files based on `LANGUAGE_SETTING.json`:
- Monolingual: `{project}/draft/chNN.md`
- Bilingual: `{project}/draft/[primary-code]/chNN_[code].md`
- Dialogue-bilingual: `{project}/draft/chNN.md`

## Review Categories

Run all six categories for every chapter. Skip category-specific checks that don't apply to the current language mode.

---

### 1. Character Consistency — 角色一致性

For every named character appearing in the chapter:

**Identity anchors (must never drift):**
- Physical description matches CHARACTER_BIBLE — flag if a character's eye color, height, or distinctive feature contradicts their profile
- Name and nickname usage is consistent with the registered form
- Age-appropriate behavior and capability

**Speech fingerprint:**
- Vocabulary level matches (formal / colloquial / dialect / archaic)
- Sentence style matches (terse / verbose / fragmented)
- Registered pet phrases (口头禅) appear where natural — flag if a character's signature phrase disappears for multiple chapters
- "Avoidance words" are not used — if the bible says a character never says X, flag any occurrence
- Emotional register fits: how they express anger, fear, affection should match their profile

**Knowledge state:**
- A character must not reference information they haven't learned yet in the story
- Cross-check with CHARACTER_BIBLE's "Knowledge State" section and TIMELINE.md
- Flag: "Character X mentions [fact] but only learns it in Chapter [N+2]"

**Relationships:**
- How Character A addresses/refers to Character B matches the registered relationship
- If a relationship changes in this chapter, it must be plot-justified
- Formal/informal address switches without cause → flag

---

### 2. Timeline Consistency — 时间线一致性

- Events follow the order laid out in OUTLINE.md for this chapter
- Time gaps between scenes are plausible (no "walked across the continent in an afternoon")
- Character locations are consistent — a character who was in City A at the end of Scene 1 shouldn't appear in City B at the start of Scene 2 without travel
- In-world dates and day numbers match TIMELINE.md
- Continuity with previous chapter's ending: location, time of day, character emotional states, any unresolved mid-action cliffhangers
- Plot beats from the outline are all addressed — flag any missing beat

---

### 3. Language Quality — 语言质量

**Repetitive language (车轱辘话):**
- Same adjective or adverb used >3× within 500 characters → flag with all occurrences
- Identical sentence structure pattern repeated in consecutive paragraphs (e.g., three "他看着……他想着……他感到……" in a row)
- Circular descriptions: two sentences that say the same thing with different words, adding no new information
- Filler phrases that pad word count: "不由得", "忍不住", "情不自禁" stacking; "的确如此" / "毫无疑问" as empty affirmations

**Awkward phrasing:**
- Unnatural word order that sounds like machine translation
- Overly long sentences (>80 characters in Chinese / >40 words in English) that lose syntactic coherence
- Mixed metaphors within the same paragraph
- Excessive hedging ("似乎可能大概也许")

**Prose rhythm:**
- >5 consecutive sentences of similar length → flag monotony
- Action scenes with long contemplative sentences → pacing mismatch
- Quiet scenes with staccato fragments → pacing mismatch (unless the style guide specifically calls for it)

**Style compliance:**
- Compare against STYLE_SETTING.json and the custom style file
- If the style says "rich prose" don't flag dense description; if it says "terse" don't flag short sentences
- Dialogue density: if STYLE_SETTING says `dialogue_density: high`, flag chapters with <30% dialogue

---

### 4. Language Contamination — 语言污染

Applies to ALL language modes. This is often the subtlest issue and the hardest to catch.

**Monolingual mode:**
- No foreign-language words leaking into prose (exception: terms registered in glossary as intentionally foreign)
- No grammar patterns from another language:
  - Chinese prose with Japanese-style 连体修饰 (long pre-nominal modifier chains)
  - Chinese prose with English-style passive voice overuse
  - Japanese prose with Chinese-style 四字成语 that don't exist in Japanese

**Dialogue-bilingual mode (critical — this is where contamination is most likely):**

_Writing-language narration:_
- Must be clean, natural writing-language prose
- No dialogue-language grammar patterns in narration (e.g., Japanese SOV order leaking into Chinese SVO narration)
- No dialogue-language script leaking into narration (e.g., katakana/hiragana in Chinese prose, unless it's a registered proper noun in the glossary)

_Dialogue-language lines:_
- Must be natural, idiomatic dialogue-language
- Grammar, particles, conjugation must be correct for the dialogue language
- Register must match the character's speech fingerprint IN THAT LANGUAGE

_Translation lines:_
- Must be natural writing-language
- Not a word-for-word literal translation
- Should convey the same meaning and emotional tone, not the same syntax
- Character voice should come through in both versions

**Full bilingual mode:**
- Each language version must read naturally as standalone prose
- Translation should not produce translationese in either direction

---

### 5. Dialogue Format Compliance — 台词格式

**Dialogue-bilingual mode (primary focus):**

Check the `dialogue_format` settings from LANGUAGE_SETTING.json.

- Every character dialogue line has BOTH the original and translation
- Original uses the configured `original_wrapper` brackets (default: 「」 for Japanese)
- Translation follows immediately on the next line using `translation_wrapper` (default: （） )
- No dialogue line is missing its translation pair — count originals and translations, they must match
- When a character speaks multiple consecutive lines (with narration tags between), each line follows the paired format
- Internal monologue marked with dialogue indicators follows the same bilingual rule
- Narration between dialogue lines is exclusively in the writing language
- Verify the translation is semantically accurate (not just present)

**Monolingual / full bilingual mode:**
- Dialogue formatting is consistent (same quotation mark style throughout)
- Proper quotation marks for the language (「」for Japanese, ""for Chinese, "" for English)
- No orphaned opening/closing marks

---

### 6. Glossary Alignment — 译名对齐

Applicable when TRANSLATION_GLOSSARY.md exists (bilingual or dialogue-bilingual modes).

- All proper nouns match their registered form in the glossary
  - Character names: exact match, no creative variations
  - Place names, faction names: exact match
  - Specialized terms (skills, items, magic systems): exact match
- The Banana Rule: a registered translation is non-negotiable — "ノルド" is always "诺德", never "诺尔德" or "北方人"
- Flag any proper noun that appears in the text but is NOT in the glossary → candidate for "待确认" section
- If a glossary term appears in the "待确认" section, flag as MEDIUM (should be confirmed before next chapter)
- Cross-language consistency: the same character's name in dialogue-language lines and translation lines must both match the glossary

---

## Output Format

Produce this structured report:

```markdown
# Proof-Read Report: Chapter [NN]

**Verdict**: [PASS / PASS_WITH_NOTES / NEEDS_REVISION]
**Issues**: [N] total — [N] HIGH / [N] MEDIUM / [N] LOW
**Word count**: [N] (target: [N] ± 10%)

---

## HIGH — Must Fix

> If empty: "No high-severity issues found."

### [H1] [Category]: [Brief title]
- **Location**: [paragraph number or quote of surrounding text]
- **Text**: "[exact problematic text]"
- **Issue**: [specific description]
- **Fix**: [concrete suggestion — not vague guidance]

### [H2] ...

---

## MEDIUM — Should Fix

### [M1] [Category]: [Brief title]
- **Location**: ...
- **Text**: "..."
- **Issue**: ...
- **Fix**: ...

---

## LOW — Consider

### [L1] [Category]: [Brief title]
- **Location**: ...
- **Note**: ...

---

## Glossary Updates

### Mismatches Found
| Location | Term Used | Glossary Entry | Action |
|----------|-----------|----------------|--------|

### New Terms to Register
| Term | Suggested Translation | Type | First Occurrence |
|------|----------------------|------|-----------------|

---

## Summary
[1-2 sentences: overall quality assessment and most important action item]
```

## Severity Definitions

| Severity | Criteria | Examples |
|----------|----------|---------|
| **HIGH** | Factual error, character contradiction, missing dialogue translation, plot-beat omission, knowledge-state violation, serious language contamination | Character uses wrong name for another character; dialogue line has no translation pair; character references future event |
| **MEDIUM** | Glossary mismatch, minor timeline ambiguity, noticeable repetition (>5×), awkward phrasing that breaks immersion, unconfirmed glossary terms | Proper noun doesn't match glossary; same adjective 6× in 500 chars; passive voice overuse in Chinese |
| **LOW** | Style preference, minor repetition (3-4×), optional prose improvement, rhythm suggestion | Slight monotony in sentence length; could vary paragraph structure |

## Verdict Rules

| Verdict | Condition |
|---------|-----------|
| **PASS** | 0 HIGH and ≤ 2 MEDIUM |
| **PASS_WITH_NOTES** | 0 HIGH and > 2 MEDIUM |
| **NEEDS_REVISION** | ≥ 1 HIGH issue |

## Batch Review Mode

When reviewing multiple chapters (`ch01-ch05` or `all`):

1. Review each chapter in order
2. Track cross-chapter patterns:
   - Character speech drift (does a character's voice change gradually?)
   - Recurring glossary violations
   - Systemic language contamination patterns
3. Produce per-chapter reports + a batch summary
4. Batch summary highlights systemic issues:
   ```
   Systemic issues across Ch03-Ch05:
   - Character X's speech becomes increasingly informal without plot justification
   - Term "Y" inconsistently translated (3 variations found)
   - Narration shows increasing Japanese SOV patterns in later chapters
   ```

## Integration Notes

### When called by /novel-write (automated)

The writing workflow passes the chapter draft (as saved file) and chapter number. The proof-reader:
1. Loads all reference files
2. Reviews the draft
3. Returns the structured report
4. The main agent reads the verdict:
   - **PASS** → save the draft as final
   - **PASS_WITH_NOTES** → save, optionally apply MEDIUM fixes
   - **NEEDS_REVISION** → main agent applies HIGH-severity fixes directly (or re-delegates to writing subagent with specific correction instructions), then re-runs proof-reader. Maximum 2 review cycles.

### When called manually by user

User invokes `/proof-reader ch05` or `/proof-reader all`. The proof-reader:
1. Loads reference files
2. Reviews the specified chapter(s)
3. Presents the report to the user
4. Does NOT auto-fix — the user decides what to act on

## Key Rules

- **Be specific.** "语言不够自然" is useless. "第3段: '他的眼神中透露出一种难以言喻的复杂情感' — 套话, replace with a concrete emotional indicator tied to this character's profile" is useful.
- **Quote the problem.** Every HIGH/MEDIUM issue must include the exact problematic text.
- **Respect the style guide.** If the custom style file encourages long contemplative sentences, don't flag them. Read the style settings BEFORE flagging style issues.
- **Don't over-flag.** If the character bible says a character uses formal speech, formal speech is correct, not stiff. If the style says "rich prose", dense description is correct, not purple.
- **Distinguish intentional from accidental.** A Japanese word in Chinese narration might be an intentional proper noun (check glossary first). A Chinese grammar pattern in Japanese dialogue is almost always accidental.
- **Count before flagging repetition.** "He said / she said" dialogue tags are expected to repeat. Flag repetition only when the same NON-FUNCTIONAL word appears in close proximity beyond normal prose rhythm.
