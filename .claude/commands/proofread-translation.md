---
name: proofread-translation
description: "Translation proofreading skill. Reads source text and translation in parallel, flags issues by severity (HIGH/MEDIUM/LOW) into a review file for human approval before any edits are made. Adapts checking style based on translation type (game/novel/technical/subtitle/academic/sg-research). Use when user says '校对翻译', '翻译审查', 'proofread', 'check translation', or provides a translation file to review."
argument-hint: [translation-type; source file; translation file; optional glossary — e.g. "rpg game; source.txt; translated.txt; glossary.md"]
<!-- Types: game / novel / technical / subtitle / academic -->
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob
---

# Proofread Translation

Arguments: **$ARGUMENTS**

## Overview

This skill performs a structured bilingual proofreading pass over a translation. It reads source and target texts in parallel, applies type-specific checks, and outputs a **review file** (`[basename]_review.md`) listing all flagged issues with severity ratings.

**No edits are made to the translation.** All findings go to the review file for human judgment.

---

## Argument Parsing

Parse `$ARGUMENTS` for:

| Field | Example values | Default |
|-------|---------------|---------|
| Translation type | `rpg game`, `novel`, `technical`, `subtitle`, `academic`, `sg-research` | asked if missing |
| Source file | `original_en.txt`, `source.md` | asked if missing |
| Translation file | `translated_zh.txt`, `ch01_zh.md` | asked if missing |
| Glossary file | `terms.md`, `TRANSLATION_GLOSSARY.md` | optional |

**Flexible input formats** (all equivalent):
```
"rpg game; source.txt; translation.txt; glossary.md"
"rpg game" --source=source.txt --translation=translation.txt
"novel"     → then ask for source and translation paths
```

If translation type is not provided or unclear, ask:
```
翻译类型是什么？
  1. rpg / 游戏       （RPG/视觉小说/剧情游戏）
  2. novel / 小说     （文学/轻小说/同人文）
  3. technical / 技术  （技术文档/说明书）
  4. subtitle / 字幕  （影视字幕/配音脚本）
  5. academic / 学术  （论文/研究报告）
```

---

## Severity Levels

### HIGH — 可能造成误解或严重失准
必须审查，通常需要修改。

| 代码 | 类型 | 描述 |
|------|------|------|
| `H1` | 错译 | 语义理解错误，译文与原文意思相悖或严重偏离 |
| `H2` | 张冠李戴 | 说话人/动作主体搞错（尤其对话、代词指代） |
| `H3` | 术语不符 | 与译名表/术语表中已确定的译法不一致 |
| `H4` | 未译句 | 整句或整段原文未翻译，直接保留源语言 |
| `H5` | 漏译 | 原文某段内容在译文中完全消失（非有意删节） |
| `H6` | 增译 | 译文出现原文中没有的信息（非有意意译补充） |

### MEDIUM — 质量问题，影响流畅性或准确性
建议审查，视上下文决定是否修改。

| 代码 | 类型 | 描述 |
|------|------|------|
| `M1` | 语境断裂 | 与前后段落/句子的逻辑或信息不连贯 |
| `M2` | 语气失配 | 角色/叙述者的语气/口吻与上下文不符（如正式→随意） |
| `M3` | 文化误处理 | 文化特定表达处理不当（直译造成歧义，或过度本土化） |
| `M4` | 模糊译法 | 译文可以被理解为多种意思，但原文只有一种意思 |
| `M5` | 术语摇摆 | 同一概念/名称在译文中译法不统一（但不在术语表中） |

### LOW — 目标语言表达问题，不影响理解
可选修改，视风格规范决定。

| 代码 | 类型 | 描述 |
|------|------|------|
| `L1` | 语法错误 | 目标语言语法不正确 |
| `L2` | 标点问题 | 标点符号使用不规范（如用英文标点代替中文标点） |
| `L3` | 表达不自然 | 语义正确但读来生硬，有"翻译腔" |
| `L4` | 格式问题 | 排版、换行、空格等格式与原文约定不符 |

---

## Translation Type Profiles

不同类型的翻译对应不同的检查重点和风格预期。

### `game` — RPG/视觉小说/剧情游戏
```
重点检查：H2（对话说话人），H3（道具/技能/地名术语），M2（角色口吻一致性）
风格预期：对话生动自然，角色有各自口癖；UI文字简洁；技能/道具名需与官方一致或内部统一
特别关注：
  - 选项文本长度（过长可能超出UI框）
  - 角色语气标签（例：温柔、傲娇、中二）
  - 人称代词一致性（"我/本座/咱"等不混用）
```

### `novel` — 文学/轻小说/同人文
```
重点检查：H1（语义准确），M2（叙事腔调），M3（文化元素），L3（翻译腔）
风格预期：流畅自然的目标语表达；文学手法（比喻、排比）需对应处理而非直译
特别关注：
  - 内心独白语气（是否与叙事者声音一致）
  - 比喻/意象的跨文化适配
  - 章节/场景切换处的衔接感
```

### `technical` — 技术文档/说明书/API文档
```
重点检查：H1（精确性），H3（术语一致），H4（不遗漏），H5（完整性），L1（语法规范）
风格预期：准确、简洁、一致；术语需精确统一；无歧义
特别关注：
  - 数字、单位、格式是否保留
  - 代码/命令不应被翻译
  - 注意事项/警告标识的语气（必须/建议/可选）
```

### `subtitle` — 影视字幕/配音脚本
```
重点检查：H4（未译），M1（上下文连贯），M2（角色口吻），L3（口语化）
风格预期：口语化，符合目标语说话节奏；尽量保留情感强度
特别关注：
  - 字幕长度约束（每行建议≤20字）
  - 语气词、叹词的处理
  - 同一角色不同场合的语气变化是否合理
```

### `academic` — 学术论文/研究报告
```
重点检查：H3（术语一致），H5（完整性），H1（准确性），L1（正式语法）
风格预期：正式、客观、术语精确；引用格式保持原样
特别关注：
  - 专业术语的统一性（同一论文中不能有两种译法）
  - 脚注/引用不应消失
  - 被动句的处理（学术文体常用被动）
```

---

## Workflow

### Step 1: 加载资源

```python
import pathlib

# 读取源文本
source_text = pathlib.Path(SOURCE_FILE).read_text(encoding="utf-8")

# 读取译文
translation_text = pathlib.Path(TRANSLATION_FILE).read_text(encoding="utf-8")

# 读取术语表（若提供）
glossary = {}
if GLOSSARY_FILE and pathlib.Path(GLOSSARY_FILE).exists():
    glossary_text = pathlib.Path(GLOSSARY_FILE).read_text(encoding="utf-8")
    # 解析术语表中的确认对照（source_term → target_term）
    # 格式参考 TRANSLATION_GLOSSARY.md 的表格结构
```

### Step 2: 分段对齐

将源文本和译文按自然段落/句子对齐，形成平行段落对。

**对齐策略**（按优先级）：
1. **章节标题对齐**：`# Chapter N` ↔ `# 第N章` 等
2. **段落计数对齐**：按段落编号对应（假设译者未合并/拆分段落）
3. **场景分隔符对齐**：`---` 分隔的场景
4. **句子级对齐**（仅当段落结构差异明显时）

若段落数差异 > 10%：标注
```
⚠️ 段落数不匹配（源：[N] 段，译：[M] 段）
可能存在漏译或合并段落。将按顺序对齐，差异处手动核查。
```

### Step 3: 分段校对

**处理大文件**：按每批 20-30 段处理，保存进度，避免上下文溢出。

对每个对齐的段落对（源段, 译段），执行以下检查：

#### 检查 H4/H5（首先扫描，最快）
```python
# H4: 译文中是否有未翻译的源语言文字
# H5: 对应源段是否在译文中消失
for i, (src, tgt) in enumerate(aligned_pairs):
    if not tgt.strip():
        flag(H5, i, src, tgt, "对应源段在译文中缺失")
    if contains_source_language(tgt, source_lang):
        flag(H4, i, src, tgt, f"译文含未翻译的{source_lang}文本")
```

#### 检查 H3（术语表比对）
```python
for term_src, term_tgt in glossary.items():
    if term_src in src and term_tgt not in tgt:
        # 原文含该术语，但译文中没有对应的标准译名
        flag(H3, i, src, tgt, f'术语"{term_src}"应译为"{term_tgt}"，但译文中未找到')
```

#### 检查 H1/H2/H6/M1-M5/L1-L4（语义/语用层）

用以下框架逐段分析：

**分析提示框架**（内部使用）：
```
源文：[源文本]
译文：[译文]
上下文（前段）：[前一段译文摘要]
翻译类型：[type]
术语表：[相关术语对照]

检查项：
1. 语义是否准确？（H1）
2. 若含对话，说话人/动作主体是否正确？（H2）
3. 译文是否含原文没有的信息？（H6）
4. 与前段的衔接是否自然？（M1）
5. 语气/口吻是否符合[type]要求？（M2）
6. 是否有文化误处理？（M3）
7. 是否存在歧义？（M4）
8. 是否有语法/标点/自然度问题？（L1-L4）
```

对每个发现的问题，生成一条标记记录。

### Step 4: 生成审查文件

**输出路径**：`[translation_file的目录]/[basename]_review.md`

例：`ch01_zh_review.md`

**文件格式**：

```markdown
# 翻译校对报告

**原文**：[SOURCE_FILE]
**译文**：[TRANSLATION_FILE]
**术语表**：[GLOSSARY_FILE 或 "未提供"]
**翻译类型**：[type]
**校对时间**：[timestamp]
**校对引擎**：Claude

---

## 总览

| 严重度 | 数量 | 建议 |
|--------|------|------|
| 🔴 HIGH | [N] | 必须审查 |
| 🟡 MEDIUM | [N] | 建议审查 |
| 🟢 LOW | [N] | 酌情处理 |

**总计**：[N] 处标注

---

## 🔴 HIGH — 必须审查

### [H-001] H3 术语不符
**位置**：第3段 / 行 42
**原文**：`The divergence meter reads 0.571024`
**译文**：`偏差计数器显示0.571024`
**问题**：术语表中 "divergence meter" = "发散数计"；"reads" 在仪器语境应译为"读数为"
**术语表参考**：divergence meter → 发散数计（§2.1）
**建议方向**：`发散数计读数为 0.571024`
**处理**：[ ] 采纳建议 / [ ] 自行修改 / [ ] 忽略（附理由）

---

### [H-002] H2 张冠李戴
**位置**：第7段 / 行 89
**原文**：`"I understand," Kurisu said. Okabe nodded.`
**译文**：`"我明白了，"冈部说道。克里斯点头。`
**问题**：原文说话人为 Kurisu（克里斯），动作主体为 Okabe（冈部），译文颠倒
**处理**：[ ] 采纳建议 / [ ] 自行修改 / [ ] 忽略（附理由）

---

## 🟡 MEDIUM — 建议审查

### [M-001] M1 语境断裂
**位置**：第12段 / 行 134
**前段译文（摘要）**：...冈部决定发送D-Mail...
**当前译文**：`实验室里鸦雀无声。`
**问题**：前段以紧张的行动结尾，此句突然转为静止描写，过渡感较弱；原文此处有一个场景切换标记（`---`），但译文中未体现
**处理**：[ ] 添加场景分隔符 / [ ] 保持原样 / [ ] 其他

---

## 🟢 LOW — 酌情处理

### [L-001] L2 标点问题
**位置**：第2段 / 行 18
**译文**：`这是不可能的...真的吗?`
**问题**：`?` 应为中文问号 `？`；`...` 应为中文省略号 `……`
**处理**：[ ] 修改 / [ ] 保持（风格选择）

---

## 附：未核查段落

以下段落因源译文对齐困难，未完成自动校对，**建议人工复核**：
- 第 [N] 段（段落数不匹配区域）
- 第 [N] 段（源文含代码/表格，自动对齐可能有误）

---

## 术语一致性总览

（仅在提供术语表时显示）

| 术语（源语言） | 标准译名 | 本文实际用法 | 偏差 |
|--------------|---------|-----------|------|
| divergence meter | 发散数计 | 发散数计 (12次), 偏差计 (1次) | ⚠️ 1处不一致 → [H-001] |
| IBN 5100 | IBN 5100 | IBN 5100 (全部) | ✅ |
| ...
```

### Step 5: 摘要输出（命令行）

校对完成后，在会话中显示：

```
校对完成：[TRANSLATION_FILE]

🔴 HIGH  [N] 处：[H1×N, H2×N, H3×N, H4×N, H5×N, H6×N]
🟡 MEDIUM [N] 处：[M1×N, M2×N, ...]
🟢 LOW   [N] 处：[L1×N, L2×N, ...]

审查文件：[path]_review.md

⚠️ 未修改原译文。请审阅报告后自行决定修改方案。
```

---

## 大文件处理

若文件超过 ~3000 行或 ~30000 字，分批处理：

```
正在处理：段落 1–30 / 142 ...
正在处理：段落 31–60 / 142 ...
...
```

每批处理后追加写入审查文件（不覆盖），确保进度不丢失。

---

## Key Rules

- **绝不修改译文**：校对输出仅为标注建议，不对原文件进行任何编辑
- **给出位置**：每条标注必须包含行号或段落编号，方便人工定位
- **提供原文对照**：每条标注同时显示原文和译文，不要只描述问题
- **区分确信度**：对于有一定不确定性的标注，在"问题"字段加注"（待核查）"
- **术语表优先**：术语表中已确认的译名发现不符时，必须标为 H3，不降级
- **不强制建议**：对 MEDIUM/LOW 问题，给出方向即可，不必写出"正确答案"
- **尊重风格选择**：若译者明显有意采用某种风格（如古文体），不将其标为问题；若确实偏离上下文，标为 M2
