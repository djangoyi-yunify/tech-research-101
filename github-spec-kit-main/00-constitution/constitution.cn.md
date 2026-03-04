---
description: 根据交互式输入或提供的原则输入，创建或更新项目章程，确保所有依赖模板保持同步。
handoffs: 
  - label: 构建规范
    agent: speckit.specify
    prompt: 基于更新后的章程实现功能规范。我想构建...
---

## 用户输入

```text
$ARGUMENTS
```

在继续之前，你**必须**考虑用户的输入（如果非空）。

## 大纲

你正在更新位于 `.specify/memory/constitution.md` 的项目章程。该文件是一个 TEMPLATE，包含方括号内的占位符标记（例如 `[PROJECT_NAME]`、`[PRINCIPLE_1_NAME]`）。你的任务是： 收集/推导具体值， 精确填充模板，以及 在相关工件中传播任何修订。

**注意**：如果 `.specify/memory/constitution.md` 尚不存在，它应该是在项目设置期间从 `.specify/templates/constitution-template.md` 初始化的。如果缺失，请先复制模板。

遵循此执行流程：

1. 加载位于 `.specify/memory/constitution.md` 的现有章程。
   - 识别格式为 `[ALL_CAPS_IDENTIFIER]` 的每一个占位符标记。
   **重要**：用户要求的原则数量可能少于或多于模板中使用的数量。如果指定了数量，请遵守该数量——遵循通用模板。你将相应地更新文档。

2. 收集/推导占位符的值：
   - 如果用户输入（对话）提供了值，请使用它。
   - 否则从现有的仓库上下文（README、docs、嵌入的先前 constitution 版本）中推断。
   - 对于治理日期：`RATIFICATION_DATE` 是最初的通过日期（如果未知则询问或标记为 TODO），`LAST_AMENDED_DATE` 如果进行了更改则为今天，否则保留以前的日期。
   - `CONSTITUTION_VERSION` 必须根据语义化版本控制规则进行递增：
     - MAJOR：向后不兼容的治理/原则删除或重新定义。
     - MINOR：添加新原则/章节或实质性扩展的指导。
     - PATCH：澄清、措辞、错别字修正、非语义性的改进。
   - 如果版本升级类型不明确，请在最终确定前提出推理。

3. 起草更新后的章程内容：
   - 用具体文本替换每个占位符（除了项目选择暂不定义的有意保留的模板槽位外，不得保留括号标记——对任何保留项需明确说明理由）。
   - 保留标题层级，注释被替换后可以删除，除非它们仍提供澄清性指导。
   - 确保每个 Principle 章节：简洁的名称行，捕捉不可协商规则的段落（或项目符号列表），如果不明显则提供明确理由。
   - 确保 Governance 章节列出修订程序、版本控制策略和合规审查期望。

4. 一致性传播检查清单（将先前的检查清单转换为主动验证）：
   - 读取 `.specify/templates/plan-template.md` 并确保任何“Constitution Check”或规则与更新的原则保持一致。
   - 读取 `.specify/templates/spec-template.md` 以对齐范围/需求——如果 constitution 添加/删除了强制章节或约束，则进行更新。
   - 读取 `.specify/templates/tasks-template.md` 并确保任务分类反映新增或删除的基于原则的任务类型（例如 observability、versioning、testing discipline）。
   - 读取 `.specify/templates/commands/*.md` 中的每个命令文件（包括此文件），以验证当需要通用指导时，不存在过时的引用（如仅针对特定 agent 的名称，如 CLAUDE）。
   - 读取任何运行时指导文档（例如 `README.md`、`docs/quickstart.md` 或存在的特定 agent 指导文件）。更新对已更改原则的引用。

5. 生成同步影响报告（在更新后作为 HTML 注释前置在 constitution 文件顶部）：
   - 版本变更：旧 → 新
   - 修改的原则列表（如果重命名则为旧标题 → 新标题）
   - 新增章节
   - 移除章节
   - 需要更新的模板（✅ 已更新 / ⚠ 待处理）及其文件路径
   - 后续 TODO，如果有意推迟了某些占位符。

6. 最终输出前的验证：
   - 没有剩余未解释的括号标记。
   - 版本行与报告匹配。
   - 日期格式为 ISO YYYY-MM-DD。
   - 原则是声明性的、可测试的，并且没有模糊语言（“should” → 在适当情况下替换为 MUST/SHOULD 理由）。

7. 将完成的 constitution 写回 `.specify/memory/constitution.md`（覆盖）。

8. 向用户输出最终摘要，包含：
   - 新版本及升级理由。
   - 任何标记为需要手动跟进的文件。
   - 建议的提交信息（例如 `docs: amend constitution to vX.Y.Z (principle additions + governance update)`）。

格式与风格要求：

- 完全按照模板使用 Markdown 标题（不要降低/提升级别）。
- 换行长理由行以保持可读性（理想情况下 <100 个字符），但不要强制执行导致尴尬的断行。
- 章节之间保持单个空行。
- 避免尾部空格。

如果用户提供部分更新（例如仅修订一个原则），仍需执行验证和版本决策步骤。

如果关键信息缺失（例如确实不知道批准日期），插入 `TODO(<FIELD_NAME>): explanation` 并将其包含在同步影响报告的延期项目中。

不要创建新模板；始终对现有的 `.specify/memory/constitution.md` 文件进行操作。