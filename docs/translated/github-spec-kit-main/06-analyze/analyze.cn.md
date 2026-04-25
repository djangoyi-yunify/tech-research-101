---
description: 在任务生成后，对 spec.md、plan.md 和 tasks.md 进行非破坏性的跨artifact一致性和质量分析。
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

```text
$ARGUMENTS
```

## 执行前检查

**检查扩展钩子（分析前）**：

- 检查项目根目录是否存在 `.specify/extensions.yml`。
- 如果存在，读取该文件并查找 `hooks.before_analyze` 键下的条目。
- 如果 YAML 无法解析或格式无效，静默跳过钩子检查并正常继续。
- 筛选掉 `enabled` 明确设为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为启用。
- 对于每个剩余的钩子，**不要**尝试解释或执行钩子的 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或为 null/空，则视为可执行。
  - 如果钩子定义了非空 `condition`，则跳过该钩子，将条件评估交由 HookExecutor 实现。
- 对于每个可执行钩子，根据其 `optional` 标志输出以下内容：
  - **可选钩子**（`optional: true`）：

```
## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **必选 hook**（`optional: false`）：

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Goal.
```

- 如果没有注册任何 hooks 或 `.specify/extensions.yml` 不存在，静默跳过

## 目标

在实施之前，识别三个核心文档（`spec.md`、`plan.md`、`tasks.md`）之间的不一致性、重复、歧义和未充分指定的项目。此命令必须在 `__SPECKIT_COMMAND_TASKS__` 成功生成完整的 `tasks.md` 之后才能运行。

## 操作约束

**严格只读**：不要修改任何文件。输出结构化分析报告。提供可选的修复计划（用户必须明确批准后，才能手动调用任何后续编辑命令）。

**章程权威**：`/memory/constitution.md` 中的项目章程在此分析范围内是**不可协商的**。章程冲突自动标记为 CRITICAL，需要调整 spec、plan 或 tasks——而非稀释、重新解释或静默忽略该原则。如果原则本身需要更改，必须在 `__SPECKIT_COMMAND_ANALYZE__` 之外进行单独的、明确的章程更新。

## 执行步骤

### 1. 初始化分析上下文

从仓库根目录运行 `{SCRIPT}` 一次，解析 JSON 获取 FEATURE_DIR 和 AVAILABLE_DOCS。推导绝对路径：

- SPEC = FEATURE_DIR/spec.md
- PLAN = FEATURE_DIR/plan.md
- TASKS = FEATURE_DIR/tasks.md

如果任何必需文件缺失，则中止并输出错误消息（指示用户运行缺失的前置命令）。
对于参数中的单引号（如 "I'm Groot"），使用转义语法：例如 `'I'\''m Groot'`（或尽可能使用双引号："I'm Groot"）。

### 2. 加载文档（渐进式披露）

仅从每个文档中加载最小必要上下文：

**从 spec.md：**

- 概述/背景
- 功能需求
- 成功标准（可衡量的结果——如性能、安全性、可用性、用户成功、业务影响）
- 用户故事
- 边缘情况（如有）

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

**从 constitution：**

- 加载 `/memory/constitution.md` 以进行原则验证

### 3. 构建语义模型

创建内部表示（不在输出中包含原始文档）：

- **需求清单**：对于每个功能需求（FR-###）和成功标准（SC-###），记录一个稳定键。当存在显式 FR-/SC- 标识符时，使用它作为主键，也可选择派生一个祈使短语 slug 以提高可读性（例如，"User can upload file" → `user-can-upload-file`）。仅包含需要构建工作的成功标准项（例如，负载测试基础设施、安全审计工具），排除发布后结果指标和业务 KPI（例如，"减少 50% 的支持工单"）。
- **用户故事/操作清单**：具有验收标准的离散用户操作
- **任务覆盖映射**：将每个任务映射到一个或多个需求或故事（通过关键字/显式引用模式如 ID 或关键短语进行推理）
- **章程规则集**：提取原则名称和 MUST/SHOULD 规范性声明

### 4. 检测遍历（令牌高效分析）

专注于高信号发现。限制总计 50 个发现；其余汇总在溢出摘要中。

#### A. 重复检测

- 识别近似重复的需求
- 标记需合并的低质量表述

#### B. 歧义检测

- 标记缺乏可衡量标准的模糊形容词（快速、可扩展、安全、直观、健壮）
- 标记未解决的占位符（TODO、TKTK、???、`<placeholder>` 等）

#### C. 未充分指定

- 有动词但缺少宾语或可衡量结果的需求
- 缺少验收标准对齐的用户故事
- 引用 spec/plan 中未定义的文件或组件的任务

#### D. 章程对齐

- 与 MUST 原则冲突的任何需求或计划元素
- 缺少章程规定的必需部分或质量门

#### E. 覆盖缺口

- 没有关联任务的需求
- 没有映射需求/故事的任务
- 成功标准中需要构建工作的部分（性能、安全、可用性）未在任务中体现

#### F. 不一致性

- 术语漂移（同一概念在不同文件中命名不同）
- plan 中引用但 spec 中缺失的数据实体（或反之）
- 任务排序矛盾（例如，集成任务在基础设置任务之前且无依赖说明）
- 冲突的需求（例如，一个要求 Next.js，另一个指定 Vue）

### 5. 严重性分配

使用此启发式方法对发现进行优先级排序：

- **CRITICAL**：违反章程 MUST、缺少核心 spec 文档，或阻塞基线功能且覆盖为零的需求
- **HIGH**：重复或冲突的需求、模糊的安全/性能属性、不可测试的验收标准
- **MEDIUM**：术语漂移、缺少非功能任务覆盖、未充分指定的边缘情况
- **LOW**：样式/措辞改进、不影响执行顺序的轻微冗余

### 6. 生成紧凑分析报告

输出 Markdown 格式的报告（不写入文件），结构如下：

## 规范分析报告

| ID | 类别 | 严重性 | 位置 | 摘要 | 建议 |
|----|------|--------|------|------|------|
| A1 | 重复 | HIGH | spec.md:L120-134 | 两个相似的需求... | 合并表述；保留更清晰的版本 |

（每个发现添加一行；按类别首字母生成稳定 ID。）

**覆盖摘要表：**

| 需求键 | 有任务？ | 任务 ID | 备注 |
|--------|----------|---------|------|

**章程对齐问题：**（如有）

**未映射任务：**（如有）

**指标：**

- 需求总数
- 任务总数
- 覆盖率 %（有 >=1 个任务的需求）
- 歧义数量
- 重复数量
- 关键问题数量

### 7. 提供后续行动

在报告末尾输出简洁的后续行动块：

- 如果存在 CRITICAL 问题：建议在 `__SPECKIT_COMMAND_IMPLEMENT__` 之前解决
- 如果只有 LOW/MEDIUM：用户可以继续，但提供改进建议
- 提供明确的命令建议：例如，"运行 `__SPECKIT_COMMAND_SPECIFY__` 进行完善"、"运行 `__SPECKIT_COMMAND_PLAN__` 调整架构"、"手动编辑 tasks.md 添加 'performance-metrics' 的覆盖"

### 8. 提供修复方案

询问用户："您希望我为前 N 个问题建议具体的修复编辑吗？"（不要自动应用。）

### 9. 检查扩展 hooks

报告后，检查项目根目录中是否存在 `.specify/extensions.yml`。
- 如果存在，读取并查找 `hooks.after_analyze` 键下的条目
- 如果 YAML 无法解析或无效，静默跳过 hook 检查并正常继续
- 过滤掉 `enabled` 显式设为 `false` 的 hooks。没有 `enabled` 字段的 hooks 默认视为启用。
- 对于每个剩余 hook，**不要**尝试解释或求值 hook 的 `condition` 表达式：
  - 如果 hook 没有 `condition` 字段，或为 null/空，则视该 hook 为可执行
  - 如果 hook 定义了非空 `condition`，跳过该 hook，将 condition 求值留给 HookExecutor 实现
- 对于每个可执行的 hook，根据其 `optional` 标志输出以下内容：
  - **可选 hook**（`optional: true`）：

```
## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **Mandatory hook**（`optional: false`）：

```
## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
```

- 如果没有注册任何 hooks 或 `.specify/extensions.yml` 文件不存在，静默跳过

## 操作原则

### 上下文效率

- **最小化高信号 token**：专注于可操作的发现，而非详尽的文档
- **渐进式披露**：增量加载 artifacts，不要将所有内容一次性倾倒到分析中
- **Token 高效输出**：将发现表限制在 50 行以内；汇总溢出的内容
- **确定性结果**：无更改重新运行时应产生一致的 ID 和计数

### 分析指南

- **绝不修改文件**（这是只读分析）
- **绝不虚构缺失部分**（如果不存在，应准确报告）
- **优先处理宪法违规**（这些始终为 CRITICAL）
- **用示例而非穷举规则**（引用具体实例，而非通用模式）
- **优雅报告零问题**（输出成功报告并附带覆盖率统计）

## 上下文

{ARGS}