---
name: language-setting
description: "Configure language settings for novel writing. Supports monolingual, bilingual, and dialogue-bilingual modes. Dialogue-bilingual writes prose in one language with character dialogue in another language plus inline translation — saves tokens while preserving authentic dialogue voice. Use when user says \"语言设置\", \"language setting\", \"双语写作\", \"bilingual\", \"台词双语\", \"dialogue-bilingual\", \"中英双语\", or wants to configure the writing language before starting a novel."
argument-hint: [primary-language — secondary-language (optional)]
allowed-tools: Read, Write, Edit, Bash(*)
---

# Language Setting: Configure Novel Language Mode

Configure language for: **$ARGUMENTS**

## Project Resolution

Read `.claude/active-project.json` → `project_dir`. All paths below use `{project}` as shorthand (default: `novel`). Override via argument if needed.

## Overview

This skill sets and persists the language configuration for the novel-writing pipeline. The configuration is saved to `{project}/settings/LANGUAGE_SETTING.json` and read by `/novel-write` and `/novel-export`.

Three modes are available:

| Mode | Description |
|------|-------------|
| **Monolingual** | Write entirely in one language. One output file per format. |
| **Bilingual** | Write in primary language, then translate each chapter to secondary language. Produces separate output files for each language. |
| **Dialogue-bilingual** | Prose in writing language; character dialogue in dialogue language with inline translation. Single output file. Saves ~40-50% tokens vs full bilingual while keeping authentic dialogue voice and avoiding translationese in narration. |

## Supported Languages

| Code | Language | Notes |
|------|----------|-------|
| `zh` | 简体中文 (Mandarin, Simplified) | Default Chinese mode |
| `zh-tw` | 繁體中文 (Traditional Chinese) | Use for Taiwan/HK setting |
| `en` | English | |
| `ja` | 日本語 | Japanese |
| `ko` | 한국어 | Korean |
| `fr` | Français | French |
| `de` | Deutsch | German |
| `es` | Español | Spanish |

Any language code supported by pandoc's `--metadata lang` flag is valid.

## Parsing Arguments

Extract from `$ARGUMENTS`:

1. **Primary language** — the language the novel is written in first
   - Detect from keywords: "中文" / "Chinese" / "zh" → `zh`; "英文" / "English" / "en" → `en`; etc.
   - Default if unspecified: `zh` (Chinese)

2. **Bilingual mode** — activated by any of:
   - "双语" / "bilingual" / "中英" / "英中"
   - Two language codes separated by "/" or "+"
   - "同时翻译" / "translate to" / "with translation"

2b. **Dialogue-bilingual mode** — activated by any of:
   - "台词双语" / "对白双语" / "dialogue-bilingual" / "dialogue bilingual"
   - "台词[语言]" / "dialogue in [language]" (e.g., "台词日语", "dialogue in Japanese")
   - "写作语言X台词语言Y" / "prose X dialogue Y"
   When detected, set `mode: "dialogue-bilingual"`. The first language is the **writing language** (narration/prose), the second is the **dialogue language** (character speech). If only dialogue language is specified, infer writing language from primary language.

3. **Secondary language** — for bilingual mode; **dialogue language** — for dialogue-bilingual mode
   - Parsed from keywords: if primary is `zh`, default secondary is `en`; if primary is `en`, default secondary is `zh`

4. **Translation style** — optional quality preference:
   - "直译" / "literal" → `literal`
   - "意译" / "literary" / "流畅" → `literary` (default)
   - "忠实原文" → `faithful`

5. **Simultaneous vs. post-hoc translation** (bilingual mode only; dialogue-bilingual always translates inline):
   - "同步翻译" / "chapter by chapter" → translate immediately after each chapter is written
   - "最后翻译" / "translate at end" → translate all chapters after writing is complete (default)

## Workflow

### Step 1: Parse and Confirm Settings

Parse `$ARGUMENTS` and confirm the detected settings with the user:

```
Language settings detected:

Mode: [Monolingual / Bilingual / Dialogue-bilingual]
Primary language: [language name] ([code])
[Secondary language: [language name] ([code])]                     (bilingual only)
[Writing language: [name] ([code])]                                (dialogue-bilingual only)
[Dialogue language: [name] ([code])]                               (dialogue-bilingual only)
[Translation style: literary / literal / faithful]                 (bilingual / dialogue-bilingual)
[Translation timing: per-chapter / post-writing]                   (bilingual only)
[Dialogue format: [original_wrapper] → [translation_wrapper]]      (dialogue-bilingual only)

Confirm? (or adjust any setting)
```

Wait for user confirmation or correction before writing the settings file.

### Step 2: Translation Guidelines (Bilingual Only)

For bilingual mode, establish translation guidelines based on the selected style:

**Literary (意译, default)**:
- Prioritize natural flow in the target language
- Adapt idioms and culture-specific references to target-language equivalents
- Preserve emotional tone, not literal wording
- Adjust sentence structure to fit the target language's natural rhythm
- Preserve character speech fingerprints — each character should sound distinct in both languages

**Literal (直译)**:
- Stay close to source wording
- Preserve sentence structure where possible
- Keep culture-specific terms (add translator note in parentheses if needed)
- Only restructure when the source is grammatically impossible to preserve

**Faithful (忠实原文)**:
- Hybrid: preserve meaning and cultural context precisely, but allow natural phrasing
- Do not domesticate culture-specific elements — keep them with brief contextual notes

### Step 3: Write LANGUAGE_SETTING.json

```json
{
  "mode": "bilingual",
  "primary_language": "zh",
  "primary_language_name": "简体中文",
  "secondary_language": "en",
  "secondary_language_name": "English",
  "translation_style": "literary",
  "translation_timing": "post-writing",
  "output_filenames": {
    "primary": "novel_zh",
    "secondary": "novel_en"
  },
  "configured_at": "[ISO timestamp]"
}
```

For monolingual:
```json
{
  "mode": "monolingual",
  "primary_language": "zh",
  "primary_language_name": "简体中文",
  "output_filenames": {
    "primary": "novel"
  },
  "configured_at": "[ISO timestamp]"
}
```

For dialogue-bilingual:
```json
{
  "mode": "dialogue-bilingual",
  "writing_language": "zh",
  "writing_language_name": "简体中文",
  "dialogue_language": "ja",
  "dialogue_language_name": "日本語",
  "dialogue_format": {
    "original_wrapper": "「{text}」",
    "translation_wrapper": "（{text}）",
    "placement": "next-line"
  },
  "translation_style": "literary",
  "output_filenames": {
    "primary": "novel_zh"
  },
  "configured_at": "[ISO timestamp]"
}
```

**Dialogue format explanation** (dialogue-bilingual only):
- `original_wrapper`: brackets for the dialogue-language line. Default `「」` for Japanese, `""` for English.
- `translation_wrapper`: brackets for the writing-language translation. Default `（）`.
- `placement`: always `"next-line"` — translation immediately follows the original on the next line.

**Example output** (writing: Chinese, dialogue: Japanese):
```markdown
诺德凝视着反转大陆。手中的P度量仪不断闪烁，数值远超预期。

「これは単なる偶然じゃない。P度量の測定値がすべてを物語っている」
（这不是单纯的巧合。P度量的测定值已经说明了一切）

他放下仪器，目光扫过其他五名调查员。

「全員、覚悟はいいか。これから先は、既知のデータが通用しない領域だ」
（各位，准备好了吗。从这里开始，是已知数据无法适用的领域）
```

Note: narration is exclusively in the writing language. All character dialogue appears first in the dialogue language, then in the writing language on the next line. Internal monologue in dialogue markers follows the same rule.

Save to `{project}/settings/LANGUAGE_SETTING.json`.

### Step 3.5: Create Translation Glossary (Bilingual Mode Only)

Skip this step for monolingual mode.

Create `{project}/settings/TRANSLATION_GLOSSARY.md` — the single source of truth for all cross-language name and term mappings. Every translation in `/novel-write` must consult this file before rendering any proper noun.

**Pre-populate from CHARACTER_BIBLE (if it already exists):**

Read `{project}/characters/CHARACTER_BIBLE.md` and extract every named character. For each, create a glossary row with:
- Primary-language name (as written in the bible)
- Secondary-language translation (to be filled by user or inferred from context)
- Romanization if applicable
- Notes (role, pronunciation hints)

**Initial template:**

```markdown
# 译名对照表 / Translation Glossary

> **规则：所有翻译必须优先查阅此表。表中有的词条直接使用，不得自行创造新译名。**
> **Rule: Always consult this table before translating any proper noun. Never invent a new translation for a registered term.**
>
> 更新方式：每章翻译后，将新出现的未登录词追加到「待确认」节。
> How to update: after each chapter, append newly encountered unregistered terms to the "Pending" section.

---

## 人名 / Character Names

| 原文（Primary） | 译名（Secondary） | 罗马字 / Romanization | 备注 |
|----------------|------------------|----------------------|------|
| [从CHARACTER_BIBLE自动填入] | [待填写] | [待填写] | [角色身份] |

## 地名・势力名 / Place & Faction Names

| 原文 | 译名 | 备注 |
|------|------|------|
|      |      |      |

## 专有术语 / Specialized Terms
(技能、道具、法术体系、科技概念等 / Skills, items, magic systems, sci-fi concepts, etc.)

| 原文 | 译名 | 说明 |
|------|------|------|
|      |      |      |

## 称谓・语气词 / Honorifics & Register Markers
(敬语、方言词、角色口头禅的对应译法 / Formal registers, dialects, character-specific phrases)

| 原文 | 译法 | 适用角色 | 说明 |
|------|------|----------|------|
|      |      |          |      |

---

## 待确认词条 / Pending — New Terms (Not Yet Confirmed)

> /novel-write 翻译时发现的新词，自动追加至此，下次写作前请人工确认并移入上方对应表格。
> Terms auto-appended by /novel-write during translation. Confirm and move to the tables above before the next writing session.

| 出现章节 | 原文 | 自动译名（待审） | 类型 | 状态 |
|----------|------|-----------------|------|------|
|          |      |                 |      | 待确认 |
```

After creating the template, **scan CHARACTER_BIBLE.md and fill in the Character Names table** with all named characters found, leaving the secondary-language translation column as `[待填写]` for the user to complete before writing begins.

Present the filled-in glossary to the user:

```
译名表已创建：{project}/settings/TRANSLATION_GLOSSARY.md

已从 CHARACTER_BIBLE 预填 [N] 个人名条目。
请在开始写作前填入译名列（否则翻译阶段将使用自动推断译名并标记为"待确认"）。

{project}/settings/TRANSLATION_GLOSSARY.md
```

### Step 4: Create Draft Directory Structure

```bash
mkdir -p {project}/draft
mkdir -p {project}/settings
mkdir -p {project}/output
```

For bilingual mode, also create language-specific draft folders:
```bash
mkdir -p {project}/draft/zh
mkdir -p {project}/draft/en
```
(Replace `zh` and `en` with the actual language codes.)

For dialogue-bilingual mode, **no language subdirectories** — single output:
```bash
mkdir -p {project}/draft
# All chapters go to {project}/draft/chNN.md — both languages are inline
```

### Step 5: Confirm Output

```
Language setting saved.

Mode: [Monolingual / Bilingual]
Primary: [language name]
[Secondary: [language name]]
[Translation: [style], [timing]]

Draft directories:
- {project}/draft/          (primary language chapters)
[- {project}/draft/[code]/  (secondary language chapters)]   (bilingual only)

Output files will be named:
- {project}/output/novel.[ext]              (monolingual)
[- {project}/output/novel_[code].[ext]      (bilingual)]

Settings file: {project}/settings/LANGUAGE_SETTING.json

Next steps:
- /novel-style          → configure writing style and chapter length
- /character-design     → design characters from the outline
- /novel-write          → start writing
```

## Bilingual Writing Notes

When `/novel-write` operates in bilingual mode, it follows this per-chapter sequence:

1. Write the chapter in the primary language
2. Save to `{project}/draft/[primary-code]/chNN.md`
3. Translate to secondary language (immediately if `per-chapter`, or deferred if `post-writing`)
4. Save translation to `{project}/draft/[secondary-code]/chNN.md`
5. Both files get the same filename prefix (e.g., `ch01.md`) for easy pairing

**Character name consistency across languages**: when a character has both a Chinese name and an English name (e.g., 林夏 / Lin Xia), both forms are registered in the CHARACTER_BIBLE and used consistently in the respective language version.

**Chapter heading format** (must match in both languages for TOC generation):
- Primary (zh): `# 第N章：[章节名]`
- Secondary (en): `# Chapter N: [Title]`

## Dialogue-Bilingual Writing Notes

When `/novel-write` operates in dialogue-bilingual mode:

1. Write the chapter as a single file with inline bilingual dialogue
2. Save to `{project}/draft/chNN.md`
3. No separate translation step — dialogue translation happens inline during writing

**Core rules:**
- All narration, description, and internal exposition: **writing language only**
- All character dialogue (「」marked speech): **dialogue language first, then writing-language translation on next line**
- Internal monologue in dialogue markers: same bilingual treatment
- Character thoughts NOT in dialogue markers: writing language only

**Dialogue format:**
```
[writing-language narration]

「dialogue-language speech」
（writing-language translation）

[writing-language narration continues]
```

**Why this mode exists:**
- Saves ~40-50% tokens vs full bilingual (only dialogue is doubled, not narration)
- Dialogue in the source language preserves authentic character voice and register

**Translation glossary**: still required. All proper nouns must be consistent between dialogue-language lines and their writing-language translations. The glossary's character name table maps names across both languages.

**Chapter heading format** (dialogue-bilingual):
- `# 第N章　[章节名]` (writing-language heading only — single file, single TOC)

## Key Rules

- **Do not start writing before this file is set.** `/novel-write` checks for `LANGUAGE_SETTING.json` before proceeding.
- **Re-running this skill** overwrites the settings file. If chapters have already been written in the old language, warn: "⚠️ Language settings changed after [N] chapters were written. This won't retroactively change existing drafts."
- **Never guess the language** from the outline text — always ask explicitly in bilingual mode which language is primary.
