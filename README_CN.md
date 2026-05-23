# Claude Code 小说写作管线

**[English README](README.md)**

基于 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的结构化长篇小说写作系统。从大纲到成书，包含章节起草、校对、插图映射和 EPUB 导出。

核心架构：**编辑（主代理）+ 写手（子代理）+ 校对员（技能调用）**。编辑统筹全局，写手在独立上下文中创作，校对员做机械性质量检查。三个角色互不共享上下文，消除审查盲区。

已在 33 章、25 万字以上的长篇小说中实战验证——台词双语（中文叙事 / 日语台词）、融合扩写模式、38 张嵌入插图。

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

## 快速开始

```bash
git clone https://github.com/user/claude-novel-pipeline.git my-novel
cd my-novel
claude --system-prompt-file ".claude/system-prompt.md"
```

### 一键启动

```
/novel-writing-pipeline "大纲.pdf -genre: 奇幻 -chapter_words: 5000"
```

### 分步执行

```
/novel-outline "大纲.pdf 20章 每章5000字"               # 阶段0：结构化大纲 + 角色圣经
/language-setting "台词双语 中文写作 日语台词"             # 阶段1：语言模式
/novel-style "literary, third-person limited, 5000字"    # 阶段2：风格配置
/novel-write "all"                                       # 阶段3：起草全部章节
/novel-export "epub"                                     # 阶段4：导出
```

### 融合扩写

```
/novel-fusion "原作.docx 扩写大纲.md"                     # 设置融合结构
/novel-write "all"                                        # 自动路由：直通/改写/新章
/novel-export "epub"
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
| `/proofread-translation` | 翻译质量审查 | `/proofread-translation "ch01-ch05"` |

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

---

## 语言模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **单语** | 全部用一种语言 | 常规小说 |
| **双语** | 逐章完整翻译 | 出版级双语版 |
| **台词双语** | 叙事用语言A，台词用语言B + 行内翻译 | 需要台词真实感的作品 |

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

### 硬约束（逐章强制）

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

## 实战经验

1. **写手的自检永远不可信。** 编辑必须独立 grep 验证。
2. **比喻密度是最难控制的约束。** 每段只留一个——删掉它句子是否塌了？没塌就删。
3. **modify 章需要不同的肌肉。** 最好的改写是透明的。
4. **不要为修订重新 spawn 子代理。** 编辑直接 Edit 修正。
5. **memo 是人格的成长档案。** 这是有价值的知识资产，不是废纸。

---

## License

MIT
