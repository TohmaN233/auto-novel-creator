# Claude Code 小说生成工作流

一套 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的 slash command 技能，能将故事大纲自动转化为完整的双语插图小说，导出为 EPUB 和 DOCX。

## 前提条件

**你需要一份详细的、逐章逐节的故事大纲。** AI Agent 在没有精心编写的大纲的情况下，无法产出连贯一致的故事。大纲越详细，成品质量越高。

## 实际案例

用一份184页的 Battle Spirits 卡牌游戏世界观 PDF 作为大纲，生成了一部 **12章双语小说**（日语主体 + 中文文学意译），约12万字中文，内嵌98张卡牌插画。从 PDF 导入到最终 EPUB 导出，全程在单个 Claude Code 会话内完成。

## 灵感与致谢

本项目受 [wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) 以及 SillyTavern 社群关于将 Claude Code 从软件工程扩展到创意工作流的讨论启发。

## 工作流程

七个 slash command 串联成端到端的小说生成流水线：

```
  PDF / 文本大纲
        |
        v
  /novel-writing-pipeline  (总调度)
        |
        +---> /character-design    阶段1：生成角色圣经
        +---> /language-setting    阶段2：配置单语/双语模式
        +---> /novel-style         阶段3：类型、视角、章节字数、自定义文风
        +---> /novel-write         阶段4：逐章写作（+ 翻译）
        +---> /asset-map           阶段4.5：图片筛选、分类、嵌入章节
        +---> /novel-export        阶段5：组装导出 EPUB / DOCX / PDF / TXT
```

### 状态持久化

所有中间状态保存在磁盘上，流水线可以随时中断、随时恢复：

```
novel/
  OUTLINE.md                    # 故事大纲（输入，不会被修改）
  NOVEL_STATE.json              # 流水线进度追踪
  characters/
    CHARACTER_BIBLE.md          # 角色圣经（持续更新）
  settings/
    LANGUAGE_SETTING.json       # 单语/双语配置
    STYLE_SETTING.json          # 类型、视角、字数目标
    CHAPTER_TEMPLATE.md         # 章节结构模板
    TIMELINE.md                 # 故事时间线（故事内日期与事件）
    TRANSLATION_GLOSSARY.md     # 专有名词翻译对照表（双语用）
  draft/
    ja/ch01_ja.md ... ch12_ja.md    # 主语言章节
    zh/ch01_zh.md ... ch12_zh.md    # 副语言章节
    FULL_NOVEL_ja.md                # 组装后的全文
    FULL_NOVEL_zh.md
  assets/
    IMAGE_MAP.json              # 图片清单与状态追踪
    *.jpeg / *.png              # 原始图片（来自 PDF 或用户提供）
    export/                     # 筛选后的导出用图片
  output/
    novel_ja.epub / .docx       # 最终导出文件
    novel_zh.epub / .docx
```

## 技能说明

### `/novel-writing-pipeline` — 总调度

顶层入口。解析大纲来源和配置参数，按顺序驱动所有子技能。

```
/novel-writing-pipeline "my_outline.pdf -bilingual ja->zh -genre: fantasy -chapter_words: 8000"
```

**可选参数**（均有默认值）：

| 参数 | 示例 | 默认值 |
|------|------|--------|
| 大纲来源 | `outline.md`、`story.pdf`、直接粘贴文本 | *（必填）* |
| 语言模式 | `-bilingual ja->zh`、`中英双语` | 单语中文 |
| 类型 | `-genre: romance`、`玄幻` | 自动检测 |
| 章节字数 | `-chapter_words: 5000`、`每章3000字` | 3000 |
| 视角 | `-pov: first_person`、`第一人称` | 第三人称限制视角 |
| 输出格式 | `-output: epub pdf` | epub + docx |
| 自动推进 | `-auto-proceed: true` | false（每阶段暂停确认） |
| 自定义文风 | `-custom_style_file: "文风指导.md"` | 无 |

**确认门控**：默认在阶段1（角色审阅）、阶段2（语言确认）、阶段3（文风确认）后暂停，等用户确认后才开始自动写作。

### `/character-design` — 角色圣经生成

读取大纲，生成结构化的 `CHARACTER_BIBLE.md`：外貌锚点、性格内核、语癖指纹、人物关系图、知识边界。

```
/character-design "novel/OUTLINE.md"
/character-design "add: 第八章出现的一个对手剑士"
```

角色分为主要/次要/龙套三档。角色圣经是活文档——`/novel-write` 会在新角色出场时自动创建简略档案。

### `/language-setting` — 语言配置

配置单语或双语模式。双语模式下，每章先写主语言，再翻译为副语言，并强制执行术语表。

```
/language-setting "ja -> zh, 文学翻译, 逐章翻译"
/language-setting "zh"  # 单语中文
```

**翻译特性**：
- 逐章翻译 或 写完后统一翻译
- 文学意译 / 直译 / 忠实原文 三种风格
- `TRANSLATION_GLOSSARY.md` 专有名词翻译一致性管控
- 跨语言保持角色语癖特征

### `/novel-style` — 文风配置

设定类型、视角、时态、章节字数目标、文字密度，可选加载自定义文风指导文件。

```
/novel-style "奇幻科幻混合, 第三人称限制视角, 每章8000字"
/novel-style "文风指导.md"  # 加载自定义文风文件
```

生成 `STYLE_SETTING.json` 和 `CHAPTER_TEMPLATE.md`（根据类型定制的章节结构模板）。

### `/novel-write` — 逐章写作引擎

核心写作技能。读取大纲、角色圣经和文风设定，逐章写作并进行自洽性检查。

```
/novel-write "ch01"           # 写一章
/novel-write "ch03-ch05"      # 写一个范围
/novel-write "all"            # 写所有剩余章节
/novel-write "ch07 --restyle" # 用当前文风设定重写
```

**每章流程**：
1. 加载大纲节拍、角色档案、上一章末尾场景、故事时间线
2. 写作前分析（场景目标、情感弧线、新角色）
3. 按所有文风约束起草正文
4. 对照角色圣经做自洽性检查
5. 保存主语言草稿
6. 翻译（双语 + 逐章模式下），强制执行术语表
7. 更新 NOVEL_STATE.json 和 TIMELINE.md

### `/asset-map` — 图片筛选与定位

将原始图片（如从 PDF 大纲中提取的）处理为可导出的图片集，并分配到章节中。四阶段流水线：

```
/asset-map "all"         # 全流程：扫描 → 筛选 → 定位
/asset-map "scan-only"   # 阶段1-2：分析和分类图片
/asset-map "filter-only" # 阶段3：应用筛选规则
/asset-map "place"       # 阶段4：插入章节 markdown
/asset-map "report"      # 查看当前状态
```

**筛选逻辑**：体积预过滤 → 视觉分类（角色/地图/场景/图表/图标）→ 去重 → 与章节文本交叉比对相关性 → 每章密度上限。插入章节前需要用户确认。

### `/novel-export` — 导出最终格式

将所有章节文件组装为每种语言一个文档，通过 pandoc 转换为目标格式。

```
/novel-export "epub docx"      # 默认
/novel-export "all"            # txt + docx + pdf + epub
/novel-export "ch01-ch05 epub" # 部分导出（预览）
```

**自动检测图片资源**：如果 `novel/assets/` 目录存在且包含图片，导出技能会自动检查 `IMAGE_MAP.json`，在图片未处理时触发 `/asset-map`。

**格式支持**：

| 格式 | 图片 | 工具 |
|------|------|------|
| EPUB | 嵌入 | pandoc |
| DOCX | 嵌入 | pandoc |
| PDF | 嵌入 | pandoc + xelatex |
| TXT | 剥离 | pandoc / sed |

## 安装

1. 安装 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
2. 克隆本仓库到你的项目目录
3. `.claude/commands/` 文件夹包含所有技能定义，Claude Code 会自动识别为 slash command

```bash
git clone https://github.com/TohmaN233/auto-novel-creator.git my-novel-project
cd my-novel-project
claude  # 启动 Claude Code
```

4. （可选）安装 [pandoc](https://pandoc.org/installing.html) 用于导出——如果没装，流水线会提示你

### 依赖

| 工具 | 用途 | 安装方式 |
|------|------|----------|
| Claude Code | 一切 | [docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code) |
| pandoc | 导出（EPUB/DOCX/PDF） | `winget install JohnMacFarlane.Pandoc` / `brew install pandoc` |
| xelatex | 仅 PDF 导出 | TeX Live / MiKTeX |
| pymupdf | PDF 图片提取 | `pip install pymupdf` |

## 快速开始

```
> /novel-writing-pipeline "my_story_outline.md -bilingual en->zh -genre: romance -chapter_words: 5000"
```

或者逐步独立运行：

```
> /character-design "novel/OUTLINE.md"
> /language-setting "en -> zh, 文学翻译, 逐章翻译"
> /novel-style "言情, 第一人称, 每章5000字"
> /novel-write "all"
> /novel-export "epub docx"
```

## 自定义

### 自定义文风指导

创建一个 markdown 文件描述你想要的文风，然后引用它：

```
/novel-writing-pipeline "outline.md -custom_style_file: 我的文风.md"
```

这个文件会在每章写作前加载，覆盖默认的类型文风。你可以在里面描述句式节奏、对话规范、情感表达规则、叙事声音、禁用词汇——任何你希望 agent 内化的东西。

参见 `examples/custom_style_guide_western_fantasy.md`，这是一份完整的西幻史诗风格模板（中世纪史诗奇幻，电影感叙事，对话驱动，严格沉浸感规则）。

### 翻译术语表

双语项目中，`novel/settings/TRANSLATION_GLOSSARY.md` 会自动维护。已确认的翻译在所有章节中强制执行（"香蕉规则"——一个专有名词永远只对应一个译名）。写作过程中发现的新术语会追加到待确认区，等你审阅。

## 已知不足

### 单 Agent 瓶颈

整条流水线跑在同一个 Claude Code agent 里。能用，但有实际代价：

- **双语污染**：同一个 agent 先写日语再翻中文，语言干扰会渗透——日语句式泄漏到中文翻译里，或者中文用词被上下文里的日语原文潜移默化地影响。在两种语言共享 context window 数小时的长会话中尤为明显。
- **Context window 压力**：写到第10章以后，角色圣经、大纲、文风指导、术语表、上一章都在争抢上下文空间。agent 会按需加载来管理，但信息仍然可能丢失。
- **风格漂移**：跨越多章后，随着早期章节滑出 context window，文风可能逐渐偏移。

### 多 Agent 架构（未实现，但可行）

更稳健的架构是把工作拆分给专用 agent：

| Agent | 职责 | 好处 |
|-------|------|------|
| **写手** | 只用主语言写草稿 | 始终沉浸在一种语言的文感中 |
| **译者** | 翻译成品章节 | 上下文中没有源语言污染 |
| **审校** | 通读全稿，标记不一致 | 全新视角，全书视野（比如用 Codex） |
| **编辑** | 跨章节统一文风修正 | 不受 context window 衰减影响 |

实现起来不难——你可以直接让你的 Claude Code agent 来拆分子 agent，或者用 [Claude Agent SDK](https://docs.anthropic.com/en/docs/claude-code/sdk) 编排。我没有这样做，是因为 token 消耗会翻好几倍，而对于个人创作（不是出版级别）的项目来说，单 agent 方案是更合理的取舍。

### AI 生成小说的现实

实话实说：以当前技术水平，AI 生成的小说还不能直接发表。这条流水线产出的最好描述是**高质量初稿**——结构连贯、角色一致、文风可控，但仔细阅读仍能辨认出机器写作的痕迹。文风容易趋于节奏均匀化，情感节拍可能显得机械，真正出人意料的创意选择很少见。

这个工具最适合：
- **自娱自乐**——把你的世界观笔记变成一本可读的小说，给自己和朋友看
- **快速验证**——生成完整草稿来检验一个故事概念是否成立，再决定是否投入人力写作
- **同人/衍生创作**——比如我们的战斗之魂世界观小说化，源材料提供了丰富的结构
- **学习**——观察 AI 如何解读你的大纲，本身就是学习故事结构的过程

如果想推向发表级质量，上面描述的多 agent 审校架构会有帮助——但归根结底，人类编辑仍然不可或缺。

## 项目结构

```
auto-novel-creator/
  .claude/
    commands/
      novel-writing-pipeline.md   # 总调度
      character-design.md         # 角色圣经生成
      language-setting.md         # 语言配置
      novel-style.md              # 文风配置
      novel-write.md              # 逐章写作引擎
      asset-map.md                # 图片筛选与定位
      novel-export.md             # 导出 EPUB/DOCX/PDF/TXT
  examples/
    STYLE_SETTING.json                        # 文风配置示例
    LANGUAGE_SETTING.json                     # 双语配置示例
    CHAPTER_TEMPLATE.md                       # 章节模板示例（奇幻科幻）
    custom_style_guide_western_fantasy.md     # 自定义文风指导示例
  README.md                                   # English
  README_CN.md                                # 中文
  LICENSE
```

## 许可证

MIT
