---
description: 在任务生成后，对 spec.md、plan.md 和 tasks.md 进行非破坏性的跨工件一致性和质量分析。
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

## 用户输入

```text
$ARGUMENTS
```

您**必须**在继续之前考虑用户输入（如果不为空）。

## 预检查

**检查扩展钩子（分析前）**：
- 检查项目根目录中是否存在 `.specify/extensions.yml`。
- 如果存在，请读取该文件并查找 `hooks.before_analyze` 键下的条目
- 如果无法解析 YAML 或该文件无效，静默跳过钩子检查并继续正常执行
- 筛选掉 `enabled` 明确设置为 `false` 的钩子。将没有 `enabled` 字段的钩子视为默认启用
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或该字段为 null/空，则将该钩子视为可执行
  - 如果钩子定义了非空的 `condition`，请跳过该钩子，将条件评估留给 HookExecutor 实现
- 对于每个可执行的钩子，根据其 `optional` 标志输出以下内容：
  - **可选钩子**（`optional: true`）：

```
## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **Mandatory hook** (`optional: false`)：

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Goal.
```

- 如果没有注册任何钩子或 `.specify/extensions.yml` 不存在，静默跳过

## 目标

在实施前识别三个核心工件（`spec.md`、`plan.md`、`tasks.md`）中的不一致性、重复、歧义和规范不足的项目。此命令必须在 `/speckit.tasks` 成功生成完整的 `tasks.md` 后才能运行。

## 操作约束

**严格只读**：不要修改任何文件。输出结构化分析报告。提供可选的修复计划（用户必须明确批准后才会手动执行任何后续编辑命令）。

**宪法权威**：项目宪法（`/memory/constitution.md`）在此分析范围内**不可协商**。宪法冲突自动标记为严重问题，需要调整规范、计划或任务——而不是稀释、重新解释或静默忽略原则。如果原则本身需要变更，必须在 `/speckit.analyze` 之外的单独、明确的宪法更新中进行。

## 执行步骤

### 1. 初始化分析上下文

从仓库根目录运行一次 `{SCRIPT}` 并解析 JSON 以获取 FEATURE_DIR 和 AVAILABLE_DOCS。推导绝对路径：

- SPEC = FEATURE_DIR/spec.md
- PLAN = FEATURE_DIR/plan.md
- TASKS = FEATURE_DIR/tasks.md

如果任何必需文件缺失则以错误消息中止（指示用户运行缺失的前置命令）。
对于参数中的单引号（如 "I'm Groot"），请使用转义语法：例如 'I'\''m Groot'（或尽可能使用双引号："I'm Groot"）。

### 2. 加载工件（渐进式披露）

仅从每个工件加载最少的必要上下文：

**从 spec.md：**

- 概述/背景
- 功能需求
- 成功标准（可衡量的结果——例如性能、安全性、可用性、用户成功、业务影响）
- 用户故事
- 边缘情况（如果存在）

**从 plan.md：**

- 架构/技术栈选择
- 数据模型引用
- 阶段
- 技术约束

**从 tasks.md：**

- 任务 ID
- 描述
- 阶段分组
- 并行标记 [P]
- 引用的文件路径

**从宪法：**

- 加载 `/memory/constitution.md` 进行原则验证

### 3. 构建语义模型

创建内部表示（不要在输出中包含原始工件）：

- **需求清单**：为每个功能需求（FR-###）和成功标准（SC-###）记录一个稳定键。当存在时，使用显式的 FR-/SC- 标识符作为主键，还可选择派生一个祈使短语slug以提高可读性（例如，"User can upload file" → `user-can-upload-file`）。只包含需要可构建工作的成功标准项（例如，负载测试基础设施、安全审计工具），排除发布后的业务指标和KPI（例如，"将支持工单减少50%"）。
- **用户故事/动作清单**：具有验收标准的离散用户操作
- **任务覆盖映射**：将每个任务映射到一个或多个需求或故事（通过关键字/显式引用模式如ID或关键短语推断）
- **宪法规则集**：提取原则名称和 MUST/SHOULD 规范声明

### 4. 检测遍历（高效令牌分析）

聚焦于高信号发现。限制总计50个发现；将剩余部分汇总在溢出摘要中。

#### A. 重复检测

- 识别近似重复的需求
- 标记需要整合的低质量措辞

#### B. 歧义检测

- 标记缺乏可衡量标准的模糊形容词（快速、可扩展、安全、直观、健壮）
- 标记未解析的占位符（TODO、TKTK、???、`<placeholder>` 等）

#### C. 规范不足

- 有动词但缺少对象或可衡量结果的需求
- 缺少验收标准对齐的用户故事
- 引用 spec/plan 中未定义的文件或组件的任务

#### D. 宪法一致性

- 与 MUST 原则冲突的任何需求或计划元素
- 缺少宪法要求的章节或质量门

#### E. 覆盖缺口

- 零关联任务的需求
- 未映射需求/故事的任务
- 需要可构建工作的成功标准（性能、安全、可用性）未在任务中反映

#### F. 不一致性

- 术语漂移（同一概念在不同文件中命名不同）
- 计划中引用但规范中缺失的数据实体（或反之）
- 任务排序矛盾（例如，集成任务在基础设置任务之前且无依赖说明）
- 冲突的需求（例如，一个要求 Next.js 而另一个指定 Vue）

### 5. 严重性分配

使用此启发式方法对发现进行优先级排序：

- **严重**：违反宪法 MUST、缺少核心规范工件，或需求零覆盖阻止基线功能
- **高**：重复或冲突的需求、模糊的安全/性能属性、不可测试的验收标准
- **中**：术语漂移、缺少非功能任务覆盖、规范不足的边缘情况
- **低**：样式/措辞改进、不影响执行顺序的轻微冗余

### 6. 生成紧凑分析报告

输出 Markdown 报告（不写入文件），结构如下：

## 规范分析报告

| ID | 类别 | 严重性 | 位置 | 摘要 | 建议 |
|----|------|--------|------|------|------|
| A1 | 重复 | 高 | spec.md:L120-134 | 两个相似的需求... | 合并措辞；保留更清晰的版本 |

（每个发现添加一行；生成以类别首字母开头的稳定ID。）

**覆盖摘要表：**

| 需求键 | 有任务？ | 任务ID | 备注 |
|--------|----------|--------|------|

**宪法一致性问题：**（如有）

**未映射任务：**（如有）

**指标：**

- 需求总数
- 任务总数
- 覆盖率%（有>=1个任务的需求）
- 歧义计数
- 重复计数
- 严重问题计数

### 7. 提供下一步操作

在报告末尾，输出简洁的下一步操作块：

- 如果存在严重问题：建议在 `/speckit.implement` 之前解决
- 如果只有低/中问题：用户可以继续，但提供改进建议
- 提供明确的命令建议：例如"运行 /speckit.specify 进行细化"、"运行 /speckit.plan 调整架构"、"手动编辑 tasks.md 添加 'performance-metrics' 的覆盖"

### 8. 提供修复方案

询问用户："您是否希望我为前 N 个问题建议具体的修复编辑？"（不要自动应用它们。）

### 9. 检查扩展钩子

报告后，检查项目根目录是否存在 `.specify/extensions.yml`。
- 如果存在，读取它并查找 `hooks.after_analyze` 键下的条目
- 如果 YAML 无法解析或无效，静默跳过钩子检查并正常继续
- 过滤掉 `enabled` 明确设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为启用。
- 对于每个剩余的钩子，不要尝试解释或评估钩子 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或为 null/空，则将钩子视为可执行
  - 如果钩子定义了非空 `condition`，跳过该钩子，将条件评估留给 HookExecutor 实现
- 对于每个可执行钩子，根据其 `optional` 标志输出以下内容：
  - **可选钩子**（`optional: true`）：

```
## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **强制钩子**（`optional: false`）：

```
## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
```

- 如果没有注册钩子或 `.specify/extensions.yml` 不存在，静默跳过

## 操作原则

### 上下文效率

- **最小化高信号令牌**：聚焦于可操作的发现，而非详尽的文档
- **渐进式披露**：增量加载工件；不要将所有内容倾倒到分析中
- **令牌高效输出**：将发现表限制在 50 行；总结溢出内容
- **确定性结果**：无变更重新运行应产生一致的 ID 和计数

### 分析指南

- **永不修改文件**（这是只读分析）
- **永不虚构缺失部分**（如果不存在，准确报告它们）
- **优先处理违规行为**（这些始终是严重问题）
- **使用示例而非详尽规则**（引用具体实例，而非通用模式）
- **优雅地报告零问题**（输出带有覆盖率统计的成功报告）

## 上下文

{ARGS}