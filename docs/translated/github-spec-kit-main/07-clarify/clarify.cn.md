---
description: 通过提出最多5个高度针对性的澄清问题，识别当前功能规范中未充分说明的区域，并将答案编码回规范中。
handoffs:
  - label: 构建技术计划
    agent: speckit.plan
    prompt: 为规范创建计划。我正在使用...
scripts:
   sh: scripts/bash/check-prerequisites.sh --json --paths-only
   ps: scripts/powershell/check-prerequisites.ps1 -Json -PathsOnly
---

## 用户输入

```text
$ARGUMENTS
```

在继续之前，你**必须**考虑用户输入（如果非空）。

## 执行前检查

**检查扩展钩子（澄清之前）**：

- 检查项目根目录中是否存在 `.specify/extensions.yml`。
- 如果存在，读取该文件并查找 `hooks.before_clarify` 键下的条目。
- 如果 YAML 无法解析或无效，静默跳过钩子检查并正常继续。
- 过滤掉 `enabled` 显式设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为启用。
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或为 null/空，则将该钩子视为可执行。
  - 如果钩子定义了非空 `condition`，则跳过该钩子并将条件评估留给 HookExecutor 实现。
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

- **必选钩子**（`optional: false`）：

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

- 如果没有注册任何 hooks 或 `.specify/extensions.yml` 不存在，静默跳过

## 概述

目标：检测并减少活动功能规范中的歧义或缺失的决策点，并将澄清内容直接记录在规范文件中。

注意：此澄清工作流程预期在调用 `__SPECKIT_COMMAND_PLAN__` 之前运行（并完成）。如果用户明确表示跳过澄清（例如探索性 spike），你可以继续，但必须警告下游返工风险会增加。

执行步骤：

1. 从仓库根目录运行一次 `{SCRIPT}`（组合的 `--json --paths-only` 模式 / `-Json -PathsOnly`）。解析最小的 JSON payload 字段：
   - `FEATURE_DIR`
   - `FEATURE_SPEC`
   - （可选捕获 `IMPL_PLAN`、`TASKS` 用于未来链式流程。）
   - 如果 JSON 解析失败，中止并指示用户重新运行 `__SPECKIT_COMMAND_SPECIFY__` 或验证功能分支环境。
   - 对于 args 中的单引号，如 "I'm Groot"，使用转义语法：例如 'I'\''m Groot'（或者如果可能使用双引号："I'm Groot"）。

2. 加载当前规范文件。使用此分类法执行结构化的歧义和覆盖扫描。对于每个类别，标记状态：Clear / Partial / Missing。生成用于优先级排序的内部覆盖图（除非不会提出问题，否则不输出原始映射）。

   功能范围与行为：
   - 核心用户目标与成功标准
   - 明确的超出范围声明
   - 用户角色/人物角色区分

   领域与数据模型：
   - 实体、属性、关系
   - 标识与唯一性规则
   - 生命周期/状态转换
   - 数据量/规模假设

   交互与 UX 流程：
   - 关键用户旅程/序列
   - 错误/空/加载状态
   - 无障碍或本地化说明

   非功能性质量属性：
   - Performance（延迟、吞吐量目标）
   - Scalability（水平/垂直扩展、限制）
   - Reliability & availability（正常运行时间、恢复预期）
   - Observability（日志、指标、追踪信号）
   - Security & privacy（AuthN/Z、数据保护、威胁假设）
   - 合规/监管约束（如有）

   集成与外部依赖：
   - 外部服务/API 及其失败模式
   - 数据导入/导出格式
   - 协议/版本假设

   边界情况与故障处理：
   - 负面场景
   - Rate limiting / throttling
   - 冲突解决（例如并发编辑）

   约束与权衡：
   - 技术约束（语言、存储、托管）
   - 明确的权衡或已拒绝的替代方案

   术语与一致性：
   - 规范词汇表术语
   - 避免的同义词/已弃用术语

   完成信号：
   - 验收标准可测试性
   - 可衡量的完成定义风格指标

   杂项/占位符：
   - TODO 标记/未解决的决策
   - 缺乏量化的模糊形容词（"robust"、"intuitive"）

   对于每个 Partial 或 Missing 状态的类别，添加候选问题机会，除非：
   - 澄清不会实质性改变实现或验证策略
   - 信息最好推迟到规划阶段（内部注明）

3. 内部生成优先级队列的候选澄清问题（最多 5 个）。不要一次性输出所有问题。应用以下约束：
   - 整个会话最多 5 个问题。
   - 每个问题必须可以用以下方式回答：
      - 简短的多项选择（2-5 个互斥选项），或
      - 一个单词/短语回答（明确约束："用 <=5 个词回答"）。
   - 仅包括答案对架构、数据建模、任务分解、测试设计、UX 行为、运维准备或合规验证有实质性影响的问题。
   - 确保类别覆盖平衡：优先尝试覆盖影响最大的未解决类别；避免在单个高影响领域（例如安全态势）未解决时询问两个低影响问题。
   - 排除已回答的问题、琐碎的风格偏好或计划级执行细节（除非阻塞正确性）。
   - 优先考虑减少下游返工风险或防止验收测试偏差的澄清。
   - 如果超过 5 个类别仍未解决，按（Impact * Uncertainty）启发式选择前 5 个。

4. 顺序提问循环（交互式）：
   - 每次只呈现一个问题。
   - 对于多项选择题：
      - **分析所有选项** 并根据以下因素确定**最合适的选项**：
         - 项目类型的最佳实践
         - 类似实现中的常见模式
         - 风险降低（Security、Performance、可维护性）
         - 与规范中可见的任何明确项目目标或约束的一致性
      - 在顶部突出显示你的**推荐选项**，并提供清晰的理由（1-2 句话解释为什么这是最佳选择）。
      - 格式：`**推荐：** 选项 [X] - <理由>`
      - 然后将所有选项渲染为 Markdown 表格：

      | 选项 | 描述 |
      |--------|-------------|
      | A | <选项 A 描述> |
      | B | <选项 B 描述> |
      | C | <选项 C 描述>（根据需要添加 D/E，最多 5 个） |
      | 简答 | 提供不同的简短回答（<=5 个词）（如果适合自由形式替代方案则包含） |

      - 表格后添加：`你可以回复选项字母（例如 "A"）、通过说 "yes" 或 "recommended" 接受推荐，或提供你自己的简短回答。`
   - 对于简答风格（没有有意义的离散选项）：
      - 根据最佳实践和上下文提供你的**建议答案**。
      - 格式：`**建议：** <你提出的答案> - <简短理由>`
      - 然后输出：`格式：简短回答（<=5 个词）。你可以通过说 "yes" 或 "suggested" 接受建议，或提供你自己的回答。`
   - 用户回答后：
      - 如果用户回复 "yes"、"recommended" 或 "suggested"，使用你之前声明的推荐/建议作为答案。
      - 否则，验证答案映射到某个选项或符合 <=5 个词的约束。
      - 如果模糊，要求快速消歧（计数仍属于同一问题；不推进）。
      - 一旦满意，将其记录到工作内存中（尚不写入磁盘），然后转到下一个排队的问题。
   - 在以下情况下停止提问：
      - 所有关键歧义提前解决（剩余排队项目变得不必要），或
      - 用户发出完成信号（"done"、"good"、"no more"），或
      - 你达到 5 个已问问题。
   - 绝不提前透露未来排队的问题。
   - 如果开始时没有有效问题，立即报告没有关键歧义。

5. 每个接受答案后的集成（增量更新方法）：
   - 维护规范的内存表示（会话开始时加载一次）以及原始文件内容。
   - 对于本会话中第一个集成的答案：
      - 确保存在 `## Clarifications` 部分（如果缺失，在规范模板中最高级别的上下文/概述部分之后创建它）。
      - 在其下，创建今天的 `### Session YYYY-MM-DD` 子标题（如果不存在）。
   - 在接受后立即追加一个bullet行：`- Q: <问题> → A: <最终答案>`。
   - 然后立即将澄清应用到最合适的部分：
      - 功能歧义 → 更新或添加 Functional Requirements 中的bullet。
      - 用户交互/角色区分 → 使用澄清后的角色、约束或场景更新 User Stories 或 Actors 子部分（如果存在）。
      - 数据结构/实体 → 更新 Data Model（添加字段、类型、关系）保留顺序；简洁地注明添加的约束。
      - 非功能约束 → 在 Success Criteria > Measurable Outcomes 中添加/修改可衡量标准（将模糊形容词转换为指标或明确目标）。
      - 边界情况/负面流程 → 在 Edge Cases / Error Handling 下添加新bullet（如果模板为此提供占位符，则创建该子部分）。
      - 术语冲突 → 规范整个规范中的术语；仅在必要时通过添加 `(formerly referred to as "X")` 保留原始术语。
   - 如果澄清使早期模糊陈述失效，替换该陈述而不是重复；不留下过时的矛盾文本。
   - 每次集成后保存规范文件以最小化上下文丢失风险（原子覆盖）。
   - 保持格式：不要重新排序无关部分；保持标题层次结构完整。
   - 保持每个插入的澄清最小化和可测试（避免叙述漂移）。

6. 验证（每次写入后以及最终检查时执行）：
   - 澄清会话每个接受的答案正好包含一个bullet（无重复）。
   - 总共提问（接受）的问题 ≤ 5。
   - 更新后的部分不包含新答案应该解决的遗留模糊占位符。
   - 没有矛盾性的早期陈述保留（扫描已移除的现在无效的替代选择）。
   - Markdown 结构有效；唯一允许的新标题：`## Clarifications`、`### Session YYYY-MM-DD`。
   - 术语一致性：所有更新的部分使用相同的规范术语。

7. 将更新后的规范写回 `FEATURE_SPEC`。

8. 报告完成（提问循环结束或提前终止后）：
   - 已提问和回答的问题数量。
   - 更新规范的路径。
   - 触及的部分（列出名称）。
   - 覆盖摘要表，列出每个分类类别及其状态：Resolved（原来是 Partial/Missing 且已解决）、Deferred（超出问题配额或更适合规划）、Clear（已经足够）、Outstanding（仍然是 Partial/Missing 但影响较小）。
   - 如果有任何 Outstanding 或 Deferred 保留，建议是继续执行 `__SPECKIT_COMMAND_PLAN__` 还是稍后在 plan 后再次运行 `__SPECKIT_COMMAND_CLARIFY__`。
   - 建议的下一个命令。

行为规则：

- 如果没有发现有意义的歧义（或所有潜在问题都是低影响的），回复："未检测到值得正式澄清的关键歧义。"并建议继续。
- 如果规范文件缺失，指示用户首先运行 `__SPECKIT_COMMAND_SPECIFY__`（不要在此创建新规范）。
- 绝不超过 5 个已问问题（单个问题的澄清重试不计入新问题）。
- 除非缺失会阻止功能清晰性，否则避免投机性的技术栈问题。
- 尊重用户提前终止信号（"stop"、"done"、"proceed"）。
- 如果由于完全覆盖而未提问，输出紧凑的覆盖摘要（所有类别 Clear）然后建议推进。
- 如果达到配额但仍有未解决的高影响类别，明确将其标记为 Deferred 并说明理由。

用于优先级的上下文：{ARGS}

## 执行后检查

**检查扩展 hooks（澄清后）**：
检查项目根目录中是否存在 `.specify/extensions.yml`。
- 如果存在，读取它并查找 `hooks.after_clarify` 键下的条目
- 如果 YAML 无法解析或无效，静默跳过 hook 检查并正常继续
- 过滤掉 `enabled` 明确设置为 `false` 的 hooks。没有 `enabled` 字段的 hooks 默认为启用。
- 对于每个剩余的 hook，**不要**尝试解释或评估 hook `condition` 表达式：
  - 如果 hook 没有 `condition` 字段，或为 null/空，则认为该 hook 可执行
  - 如果 hook 定义了非空 `condition`，跳过该 hook，将条件评估留给 HookExecutor 实现
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

- 如果没有注册任何 hook 或 `.specify/extensions.yml` 不存在，静默跳过