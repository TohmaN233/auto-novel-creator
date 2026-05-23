# Claude Code 小说写作管线

**[English README](README.md)**

基于 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的结构化长篇小说写作系统。从大纲到成书，包含章节起草、校对、插图映射和 EPUB 导出。

核心架构：**编辑（主代理）+ 写手（子代理）**。写手在独立上下文中创作，编辑调用 `/proof-reader` 技能做机械性质量检查后自己动手修正。两个 agent 的上下文完全隔离，消除审查盲区。已在 25 万字以上的长篇小说中实战验证。

---

## 设计思路

### 为什么要分两个 agent？

AI 写长文最大的问题不是"写不好"，而是**自己看不出自己写得不好**。让同一个上下文既写又审，它会对自己刚写的内容产生确认偏误——校对时会跳过问题，因为它"记得"自己的意图。

所以我们把创作和审查拆到两个独立上下文里：
- **写手**（子代理）在隔离的上下文里创作，不受编辑累积的偏见影响
- **编辑**（主代理）拿到写手的成稿后，调用 `/proof-reader` 技能做机械检查（计数、格式、术语匹配），然后自己读报告、自己判断、自己动手改。不把修订打回给写手——重新 spawn 一次子代理的上下文加载代价太大，编辑拿着校对报告直接 Edit 更快更准

### Load-Instructions，不是 Content-Injection

编辑只给写手**文件路径**，写手自己用 Read 工具去读原始文件。这样做的好处：
1. 写手获得第一手材料，不是编辑消化过的二手总结
2. 编辑的上下文保持干净，留给后面的校对和修订
3. 唯一跨越边界的编辑内容是 SPECIAL_NOTES（≤300 字的编辑备注），用来传递"上一章校对发现破折号超标，注意控制"这类方向性指示

### 写手人格

长篇小说需要几十万字读起来像同一个人写的。人格系统（`persona.yaml`）定义了写手的美学偏好、遣词习惯、回避列表。每章写完后写手把收获追加到 `memo.md`——作者跨章节甚至跨书籍的成长记忆。长期写作下来，人格会积累出真正的 craft insights。
也就是说——在写作时同时培养属于自己的作家AI。

### 状态全在磁盘，随时中断恢复

写到第 17 章 token 用完了？关掉终端，下次打开直接 `/novel-write "continue"`，从第 18 章继续。所有进度（NOVEL_STATE.json）、时间线（TIMELINE.md）、世界状态（CONTINUITY_MAP.md）都是编辑每章更新的磁盘文件。

---

## 架构

```
  编辑（主代理）                          写手（子代理）
  ┌──────────────────────┐               ┌───────────────────────┐
  │ 1. 读取 NOVEL_STATE  │               │ 自行读取所有项目文件    │
  │ 2. 扫描大纲本章段落   │  模板 + 路径   │（人格、风格、角色圣经、 │
  │ 3. 填充模板参数       │──────────────>│ 术语表、时间线、大纲、  │
  │   （路径 + 编辑备注）  │               │ 上一章结尾）            │
  │                      │               │                        │
  │                      │  文件路径       │ 写章节到磁盘            │
  │ 4. 读取草稿文件   <──│<──────────────│ 写笔记到磁盘            │
  │                      │               │ 更新 memo.md           │
  │ 5. /proof-reader     │               └───────────────────────┘
  │   （6项检查）         │
  │ 6. 编辑自己修正问题   │
  │ 7. 更新状态文件       │
  └──────────────────────┘

  边界规则：
  • 编辑不替写手预读内容文件
  • 写手不更新 NOVEL_STATE / TIMELINE / CONTINUITY_MAP
  • SPECIAL_NOTES（≤300字）是唯一跨越编辑→写手边界的编辑内容
```

---

## 快速开始-使用自定义系统提示词


```bash
git clone https://github.com/user/claude-novel-pipeline.git my-novel
cd my-novel
claude --system-prompt-file ".claude/system-prompt.md"
```
我们知道CC的系统提示词专门为写代码设计，因此不适合写小说——本步骤使用了我们自定义的编辑人格系统提示词，因此没有原本系统的限制。
可直接终端启动start-novel.ps1。

### 启动 Pipeline（交互式引导）

Pipeline 不是一行命令出书——它是分阶段的交互流程，每个阶段会展示配置结果并等你确认。

```
/novel-writing-pipeline "大纲.pdf -genre: 奇幻 -chapter_words: 5000"
```

启动后 pipeline 会引导你走过 5 个阶段：

```
阶段1  大纲结构化 + 角色圣经    → 展示章节拆分和角色列表，等你确认/调整
阶段2  语言模式配置             → 单语/双语/台词双语？等你确认
阶段3  风格配置                 → 类型、视角、字数、文风，等你确认
阶段4  章节写作                 → 确认后自动推进，每章写手起草 → 编辑校对修正
阶段5  导出                     → 选择格式，生成最终文件
```

> 💡 阶段 1–3 是配置阶段，各需几分钟。通过阶段 3 的确认后，阶段 4 自动执行——推荐在晚上启动写作、第二天早上收稿。

### 分步执行（手动控制每一步）

如果你想跳过 pipeline 的引导流程，直接逐步配置：

```
/novel-outline "大纲.pdf 20章 每章5000字"               # 阶段1：结构化大纲 + 角色圣经
/language-setting "台词双语 中文写作 日语台词"             # 阶段2：语言模式
/novel-style "literary, third-person limited, 5000字"    # 阶段3：风格配置
/novel-write "all"                                       # 阶段4：起草全部章节
/novel-export "epub"                                     # 阶段5：导出
```

每个技能独立可用，不必按顺序调用。已有大纲和角色圣经？直接 `/novel-write "all"`。

### 融合扩写

```
/novel-fusion "原作.docx 扩写大纲.md"                     # 设置融合结构
/novel-write "all"                                        # 自动路由：直通/改写/新章
/novel-export "epub"
```

---

## 可配参数

通过 pipeline 启动参数或分步技能调用时指定。所有参数都有默认值，可以不传。

### 基础配置

| 参数 | 默认值 | 说明 | 示例 |
|------|--------|------|------|
| `genre` | 自动检测 | 类型/风格 | `genre: 奇幻` `genre: literary` |
| `chapter_words` | 8000 | 每章目标字数 | `每章5000字` `chapter_words: 12000` |
| `language_mode` | `monolingual` | 语言模式 | `台词双语` `bilingual: zh/en` |
| `primary_language` | `zh` | 写作语言 | `primary: ja` |
| `secondary_language` | `en` | 第二语言（双语模式） | `secondary: zh` |
| `output_format` | `epub docx` | 导出格式 | `output: epub pdf` `output: all` |
| `writer_persona` | 目录内首个 | 写手人格名称 | `persona: Elie` |

### 流程控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `auto_proceed` | `false` | 设为 `true` 跳过阶段 1–3 的确认门，全自动推进 |
| `auto_export` | `false` | 设为 `true` 写完自动导出 |
| `chapter_range` | `all` | 写哪些章 — `ch05` / `ch03-ch08` / `all` |

### 写手日志模式（journal_mode）

控制写手在每章的 `chNN_notes.md` 里写什么。这个参数决定了你能不能"看到"写手的创作过程。

| 模式 | 写什么 | 适合场景 |
|------|--------|---------|
| **`full`** | 写作手记（~300-500字）+ 写后感（~200-300字）+ Handoff Notes | **推荐。** 能感受到写手的人格——TA 的犹豫、兴奋、对角色的看法 |
| **`brief`** | Handoff Notes + 可选的一段简短感想 | 只要关键交接信息 |
| **`off`** | 仅 Handoff Notes | 纯工作模式 |

> 💡 **写手日志是这个系统最有趣的产物之一。** 子代理的 `<thinking>` 默认是看不到的，但日志模式让写手把思考过程、创作感受写成可读的文本保存到磁盘。你可以看到写手对某个场景的纠结、对角色的理解、对自己作品的诚实评价。
>
> 默认的 Elie 性格偏沉稳内敛，日志风格也比较中规中矩。但如果你创建一位更有个性的写手——比如一位傲娇的天才作家——TA 的日志会完全不同：对自己的作品又骄傲又不满意，对编辑的修改意见有情绪反应，写后感里夹杂着不服气的自我辩护……这都是人格系统带来的可玩性。

### 自定义文风（custom_style_file）

`/novel-style` 生成的 `STYLE_SETTING.json` 只包含基础参数（类型、视角、字数）。如果你想要更细粒度的文风控制——禁用词表、叙事节奏、世界观锚点、对话风格——可以写一个自定义文风文件，写手在创作时会像读 persona 一样读它。

在 `STYLE_SETTING.json` 里指向你的文风文件：

```json
{
  "custom_style_file": "custom_style_guide_western_fantasy.md"
}
```

仓库里附带了一个示例：`examples/custom_style_guide_western_fantasy.md`，包含：
- **禁用词表** — 打破中世纪沉浸感的现代词汇（"协议"、"数据"、"百分比"→替换建议）
- **叙事规则** — 对话驱动、电影式镜头运动、蒙太奇技法
- **氛围指导** — 凯尔特民间传说的神秘感、史诗的厚重感
- **硬禁止** — 元叙事评论、作者括注、第四面墙

你可以参照这个格式为自己的作品写一份。写手会在读取设置时一并加载这个文件，权重与 JSON 设置相同。

### 融合模式参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `fusion` / `融合` / `扩写` | 启用融合模式 | 自动检测，或手动指定 |
| `source:` | 源小说文件路径 | `source: 原作.docx` |

### 参数传递示例

```
# Pipeline 一次性传入多个参数
/novel-writing-pipeline "大纲.pdf genre: 奇幻 每章6000字 台词双语 中文 日语 persona: Elie auto_proceed: true"

# 分步执行时各技能独立接受参数
/novel-style "literary, third-person limited, 6000字"
/language-setting "台词双语 中文写作 日语台词"
/novel-write "ch01-ch10"
```

---

## 每章工作流

| 步骤 | 角色 | 操作 |
|------|------|------|
| 1 | 编辑 | 读取 `NOVEL_STATE.json` + 本章大纲段落 |
| 2 | 编辑 | 检查新角色 → 添加 stub 到 CHARACTER_BIBLE |
| 3 | 编辑 | 填充 `SUBAGENT_TEMPLATE.md`（章节号、标题、路径、SPECIAL_NOTES） |
| 4 | 写手 | *子代理启动* — 自行读取所有文件 → 思考 → 写章节+笔记到磁盘 |
| 5 | 编辑 | 读取草稿 → 调用 `/proof-reader`（6项检查） |
| 6 | 编辑 | 用 Edit 工具修正 HIGH/MEDIUM 问题（不重新 spawn） |
| 7 | 编辑 | Persona fidelity check（对照 `checklist.md`） |
| 8 | 编辑 | 更新 NOVEL_STATE、TIMELINE、CONTINUITY_MAP |

---

## 技能一览

| 技能 | 用途 | 示例 |
|------|------|------|
| `/novel-outline` | 原始素材 → 结构化大纲 + 角色圣经 | `/novel-outline "概念.pdf 20章"` |
| `/novel-writing-pipeline` | 端到端编排器 | `/novel-writing-pipeline "大纲.pdf"` |
| `/novel-write` | 章节起草（编辑+写手+校对） | `/novel-write "ch01"` / `"all"` |
| `/proof-reader` | 6项质量审查 | `/proof-reader "ch01-ch05"` |
| `/character-design` | 创建/更新角色档案 | `/character-design "添加ch08对手"` |
| `/language-setting` | 配置语言模式 | `/language-setting "台词双语 中文 日语"` |
| `/novel-style` | 类型、视角、字数、文风 | `/novel-style "文学, 8000字"` |
| `/novel-fusion` | 融合扩写（在原作基础上扩展） | `/novel-fusion "原作.docx 大纲.md"` |
| `/asset-map` | 图片→章节映射 | `/asset-map "all"` |
| `/novel-export` | 导出 EPUB / DOCX / PDF | `/novel-export "epub docx"` |

---

## 写手人格系统

每个写手人格是一个目录，包含三个文件：

```
writer_persona/
  Elie/                          ← 人格名称
    persona.yaml                 ← 身份：美学、哲学、说话方式
    checklist.md                 ← 机械质量门槛（每章写后自检）
    memo.md                      ← 成长手记（子代理每章追加）
```

- 子代理在**做任何事之前**先读 `persona.yaml`（模板 Step Zero）
- `memo.md` 工作区上限 200 行，满时合并旧条目
- 完结章时，写手在 `# Archive` 区写入简洁的完结小结

创建自定义人格：复制 `Elie/` 目录，修改 YAML。项目通过 `{project}/settings/active_writer.json` 指定使用哪个人格。
不同作品可以选择更适合的作家人格！
Elie的人格写作方法完全参考Sudachi https://github.com/LimeBlogs/Sudachi-Next

---

## 语言模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **单语** | 全部用一种语言 | 常规小说 |
| **双语** | 逐章完整翻译 | 双语版 | 学习使用/或想给其他语言的朋友阅读
| **台词双语** | 叙事用语言A，台词用语言B + 行内翻译 | 比如写同人，不希望角色变成中国网文说话风格，保留角色语癖 |

### 台词双语格式

```
她转过身，背对着窗户站了一会儿。

「……言わなきゃいけないことがある」
（……有些事必须说。）

她的声音很轻，像是怕把什么东西震碎。
```

校对员逐行计数验证：每条台词必须有原文+译文配对。

---

## 融合扩写模式

在原作基础上扩展——插入新章、改写旧章、保留原文。

| 类型 | 操作 | 写手任务 |
|------|------|---------|
| **passthrough** | 原样复制 | 无（仅术语校对） |
| **modify** | 织入指定修改 | 匹配原作者文风，不用人格文风 |
| **new** | 在原章之间插入新内容 | 完整创作，确保衔接 |

---

## 质量体系

### 校对员：6项检查

| # | 类别 | 检查内容 |
|---|------|---------|
| 1 | 角色一致性 | 口癖、外貌锚点、知识状态、关系 |
| 2 | 时间线 | 事件顺序、地点连续性、故事日追踪 |
| 3 | 语言质量 | 重复、拗口、节奏、风格合规 |
| 4 | 语言污染 | 语法串语、翻译腔 |
| 5 | 对话格式 | 双语配对完整、括号正确 |
| 6 | 术语对齐 | 专有名词一致（香蕉规则：注册译名不可变） |

### 硬约束

## 以下是预设，八股等问题用户根据自己喜好在CLAUDE.md里编辑。

| 约束 | 限额 | 目的 |
|------|------|------|
| "不是X是Y" 句式 | ≤ 2 | 防止公式化对比 |
| 破折号（——） | ≤ 30 | 防止滥用 |
| "有什么东西" | ≤ 1 | 模糊填充词 |
| 比喻密度 | ≤ 2/千字 | 防止堆砌 |
| AI 八股文 | 0 | 禁止"不由得感到一阵"等套话 |
| 精确感知数值 | 0 | 角色不是仪器 |
| 元标记泄露 | 0 | 正文不出现 chNN、Beat N |

### 判定规则

| 判定 | 条件 | 处理 |
|------|------|------|
| **PASS** | 0 HIGH, ≤2 MEDIUM | 保存，继续 |
| **PASS_WITH_NOTES** | 0 HIGH, >2 MEDIUM | 保存，可选修复 |
| **NEEDS_REVISION** | ≥1 HIGH | 编辑修正后重新校对（最多2轮） |

---

## 自定义与扩展

这个系统的所有配置文件都是用户可编辑的纯文本。写手读取的每一个文件——模板、人格、文风、角色圣经、术语表——你都可以直接打开修改。

如果你用过 SillyTavern 的世界书（World Info）或预设（Preset），可以这样理解：

| SillyTavern 概念 | 本系统对应 | 文件位置 |
|------------------|-----------|---------|
| 预设 (Preset) | 编辑身份 + 写手人格 + 子代理模板 | `.claude/system-prompt.md` + `writer_persona/{name}/` + `.claude/SUBAGENT_TEMPLATE.md`（系统提示词和人格合在一起就是完整的"预设"——定义了写手是谁、怎么写、遵守什么规则） |
| 世界书 (World Info) | 角色圣经 + 术语表 + 时间线 + 连续性地图 + 自定义文风 | `{project}/characters/` + `{project}/settings/`（写手每章都会读，你往里写什么它就照着什么来） |
| 作者注 (Author's Note) | SPECIAL_NOTES | 编辑每章填入模板的 ≤300 字备注 |

### 你可以自由修改的内容

**子代理模板** (`.claude/SUBAGENT_TEMPLATE.md`)：写手读到的完整指令。你可以：
- 增删思考步骤（比如加一个"音乐性检查"步骤）
- 修改日志格式（比如要求写手用对话体写写后感）
- 增加额外约束（比如"每章必须有一个伏笔"）
- 改变读取文件的列表（比如加入一个自定义的世界观设定文件）

**角色圣经** (`CHARACTER_BIBLE.md`)：每个角色的言行约束。写手每章都会读。你可以随时加新字段——比如"这个角色绝不会做的事"、"角色的秘密（只有读者知道）"。

**术语表** (`TRANSLATION_GLOSSARY.md`)：双语模式下的专有名词对照。注册的译名不可变——写手和校对员都会严格遵守。

**校对规则**：硬约束表（上面的"不是X是Y"、破折号限额等）定义在 `CLAUDE.md` 和 `proof-reader` 技能里。你可以调整限额、增删条目、加入自己发现的 AI 八股文模式。

**自定义文风文件**：写任何你想要的规则——禁用词、句式偏好、叙事手法、氛围指导。放在项目目录里，在 `STYLE_SETTING.json` 的 `custom_style_file` 字段指向它。

> 💡 **核心思路：写手能读到的文件，你都能编辑。** 你往这些文件里写什么，写手就照着什么来。这不是一个封闭的黑盒——它是一套你可以持续调教的提示词架构。

---

## 状态文件

| 文件 | 追踪内容 | 更新者 |
|------|---------|--------|
| `NOVEL_STATE.json` | 管线进度、章节状态、校对结果 | 编辑 |
| `TIMELINE.md` | **事件** — 什么时候发生了什么 | 编辑 |
| `CONTINUITY_MAP.md` | **状态** — 每章结束时的世界状态快照 | 编辑 |

---

## 项目结构

```
claude-novel-pipeline/
  .claude/
    system-prompt.md                ← 小说编辑身份
    SUBAGENT_TEMPLATE.md            ← 写手子代理模板
    commands/                       ← 技能定义（斜杠命令）
  writer_persona/Elie/             ← 默认写手人格
  examples/                        ← 示例配置
  CLAUDE.md                        ← Claude Code 项目指令

{project}/                          ← 每个小说项目（gitignore）
  OUTLINE.md / NOVEL_STATE.json / characters/ / settings/ / draft/ / assets/ / output/
```

---

## 安装

| 依赖 | 用途 | 安装 |
|------|------|------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | 全部 | 见 Anthropic 文档 |
| Python 3 + ebooklib + Pillow | EPUB 导出（含插图） | `pip install ebooklib Pillow` |
| pandoc | DOCX/PDF 导出 | `winget install JohnMacFarlane.Pandoc` |

---

## 关于 AI 生成小说

说实话：以目前的技术水平，AI 生成的小说还不能直接拿去出版。这个管线产出的更准确地说是**高质量初稿**——结构连贯、角色一致、风格可控，但仔细读还是能感觉到机器的痕迹。节奏容易趋于均匀，情绪节拍有时过于机械，真正出人意料的创意选择很少。

这个工具更适合：

- **个人阅读** — 把你的世界观笔记变成一本可读的小说，给自己和朋友看
- **快速验证** — 生成完整草稿看一个故事概念是否成立，再决定要不要投入人力精写
- **同人 / 二次创作** — 原作提供了丰富的结构和设定，AI 在此基础上扩展
- **学习** — 观察 AI 如何解读你的大纲，反过来帮你理解叙事结构

如果想推向出版级质量，双 agent 的审查架构会有帮助——但最终，人类编辑仍然不可替代。

---

## License

MIT
