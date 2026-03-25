---
description: 根据自然语言功能描述创建或更新功能规格说明。
handoffs:
  - label: 构建技术计划
    agent: speckit.plan
    prompt: 为规格说明制定计划。我正在构建...
  - label: 明确规格要求
    agent: speckit.clarify
    prompt: 明确规格要求
    send: true
scripts:
  sh: scripts/bash/create-new-feature.sh "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 "{ARGS}"
---

## 用户输入

```text
$ARGUMENTS
```

在继续之前，**必须**考虑用户输入（如果非空）。

## 执行前检查

**检查扩展钩子（规范之前）**：
- 检查项目根目录中是否存在 `.specify/extensions.yml`。
- 如果存在，读取该文件并查找 `hooks.before_specify` 键下的条目
- 如果 YAML 无法解析或无效，静默跳过钩子检查并继续正常执行
- 过滤掉 `enabled` 显式设置为 `false` 的钩子。对于没有 `enabled` 字段的钩子，默认为启用状态
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或该字段为空/null，则将该钩子视为可执行
  - 如果钩子定义了非空的 `condition`，则跳过该钩子，将条件评估留给 HookExecutor 实现
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

- **必须钩子** (`optional: false`):

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

- 如果没有注册任何 hook 或 `.specify/extensions.yml` 不存在，静默跳过

## 大纲

用户在触发消息中 `/speckit.specify` 后输入的文本**就是**功能描述。假设你在此对话中始终可以访问它，即使下面 `{ARGS}` 逐字显示。除非用户提供了空命令，否则不要要求用户重复。

根据该功能描述，执行以下操作：

1. **生成简洁的短名称**（2-4 个单词）：
   - 分析功能描述，提取最有意义的关键词
   - 创建 2-4 个单词的短名称，捕捉功能的本质
   - 尽可能使用动作-名词格式（例如 "add-user-auth"、"fix-payment-bug"）
   - 保留技术术语和缩写（OAuth2、API、JWT 等）
   - 保持简洁但足够描述性，一眼就能理解功能
   - 示例：
     - "I want to add user authentication" → "user-auth"
     - "Implement OAuth2 integration for the API" → "oauth2-api-integration"
     - "Create a dashboard for analytics" → "analytics-dashboard"
     - "Fix payment processing timeout bug" → "fix-payment-timeout"

2. **创建功能分支**，运行脚本并使用 `--short-name`（以及 `--json`）。在顺序模式下，不要传递 `--number` — 脚本会自动检测下一个可用的数字。在时间戳模式下，脚本会自动生成 `YYYYMMDD-HHMMSS` 前缀：

   **分支编号模式**：运行脚本前，检查 `.specify/init-options.json` 是否存在并读取 `branch_numbering` 的值。
   - 如果是 `"timestamp"`，在脚本调用中添加 `--timestamp`（Bash）或 `-Timestamp`（PowerShell）
   - 如果是 `"sequential"` 或不存在，不添加任何额外标志（默认行为）

   - Bash 示例：`{SCRIPT} --json --short-name "user-auth" "Add user authentication"`
   - Bash（时间戳）：`{SCRIPT} --json --timestamp --short-name "user-auth" "Add user authentication"`
   - PowerShell 示例：`{SCRIPT} -Json -ShortName "user-auth" "Add user authentication"`
   - PowerShell（时间戳）：`{SCRIPT} -Json -Timestamp -ShortName "user-auth" "Add user authentication"`

   **重要**：
   - 不要传递 `--number` — 脚本会自动确定正确的下一个数字
   - 始终包含 JSON 标志（`--json` 用于 Bash，`-Json` 用于 PowerShell），以便可以可靠地解析输出
   - 每个功能只能运行此脚本一次
   - JSON 作为输出在终端中提供 - 始终引用它来获取你需要的实际内容
   - JSON 输出将包含 BRANCH_NAME 和 SPEC_FILE 路径
   - 对于参数中的单引号（如 "I'm Groot"），使用转义语法：例如 'I'\''m Groot'（或尽可能使用双引号："I'm Groot"）

3. 加载 `templates/spec-template.md` 了解所需部分。

4. 按以下执行流程操作：

   1. 从输入中解析用户描述
      如果为空：错误 "No feature description provided"
   2. 从描述中提取关键概念
      识别：参与者、动作、数据、约束
   3. 对于不明确的方面：
      - 根据上下文和行业标准做出合理猜测
      - 仅在以下情况下标记 [NEEDS CLARIFICATION: 具体问题]：
        - 该选择显著影响功能范围或用户体验
        - 存在多种合理解释但含义不同
        - 没有合理的默认值存在
      - **限制：最多 3 个 [NEEDS CLARIFICATION] 标记**
      - 按影响程度优先处理澄清：范围 > 安全/隐私 > 用户体验 > 技术细节
   4. 填写用户场景和测试部分
      如果没有清晰的用户流程：错误 "Cannot determine user scenarios"
   5. 生成功能需求
      每个需求必须是可测试的
      对未指定的细节使用合理的默认值（在假设部分记录假设）
   6. 定义成功标准
      创建可衡量的、技术无关的结果
      包括定量指标（时间、性能、容量）和定性衡量（用户满意度、任务完成度）
      每个标准必须无需实现细节即可验证
   7. 识别关键实体（如果涉及数据）
   8. 返回：SUCCESS（规范已准备好进行规划）

5. 使用模板结构将规范写入 SPEC_FILE，用从功能描述（参数）中派生的具体细节替换占位符，同时保持部分顺序和标题不变。

6. **规范质量验证**：编写初始规范后，根据质量标准进行验证：

   a. **创建规范质量检查清单**：使用检查清单模板结构在 `FEATURE_DIR/checklists/requirements.md` 生成检查清单文件，包含以下验证项目：

```markdown
# Specification Quality Checklist: [FEATURE NAME]
      
      **Purpose**: Validate specification completeness and quality before proceeding to planning
      **Created**: [DATE]
      **Feature**: [Link to spec.md]
      
      ## Content Quality
      
      - [ ] No implementation details (languages, frameworks, APIs)
      - [ ] Focused on user value and business needs
      - [ ] Written for non-technical stakeholders
      - [ ] All mandatory sections completed
      
      ## Requirement Completeness
      
      - [ ] No [NEEDS CLARIFICATION] markers remain
      - [ ] Requirements are testable and unambiguous
      - [ ] Success criteria are measurable
      - [ ] Success criteria are technology-agnostic (no implementation details)
      - [ ] All acceptance scenarios are defined
      - [ ] Edge cases are identified
      - [ ] Scope is clearly bounded
      - [ ] Dependencies and assumptions identified
      
      ## Feature Readiness
      
      - [ ] All functional requirements have clear acceptance criteria
      - [ ] User scenarios cover primary flows
      - [ ] Feature meets measurable outcomes defined in Success Criteria
      - [ ] No implementation details leak into specification
      
      ## Notes
      
      - Items marked incomplete require spec updates before `/speckit.clarify` or `/speckit.plan`
```

b. **运行验证检查**: 根据每个检查项审查规范：

- 对每个检查项，确定其是否通过
- 记录发现的具体问题（引用相关规范章节）

c. **处理验证结果**:

- **如果所有检查项通过**: 标记检查清单完成，并进入步骤7

- **如果有检查项失败（不包括 [NEEDS CLARIFICATION]）**:
  1. 列出失败的检查项和具体问题
  2. 更新规范以解决每个问题
  3. 重新运行验证直到所有检查项通过（最多3次迭代）
  4. 如果3次迭代后仍然失败，将剩余问题记录在检查清单备注中并警告用户

- **如果仍有 [NEEDS CLARIFICATION] 标记**:
  1. 从规范中提取所有 [NEEDS CLARIFICATION: ...] 标记
  2. **限制检查**: 如果标记超过3个，仅保留3个最关键的（按范围/安全性/用户体验影响排序），其余的做出合理猜测
  3. 对于每个需要澄清的项目（最多3个），按以下格式向用户呈现选项：

```markdown
## Question [N]: [Topic]
           
           **Context**: [Quote relevant spec section]
           
           **What we need to know**: [Specific question from NEEDS CLARIFICATION marker]
           
           **Suggested Answers**:
           
           | Option | Answer | Implications |
           |--------|--------|--------------|
           | A      | [First suggested answer] | [What this means for the feature] |
           | B      | [Second suggested answer] | [What this means for the feature] |
           | C      | [Third suggested answer] | [What this means for the feature] |
           | Custom | Provide your own answer | [Explain how to provide custom input] |
           
           **Your choice**: _[Wait for user response]_
```

4. **CRITICAL - 表格格式化**：确保 markdown 表格正确格式化：
   - 使用一致的间距，竖线对齐
   - 每个单元格内容周围应有空格：`| Content |` 而不是 `|Content|`
   - 标题分隔符至少需要 3 个破折号：`|--------|`
   - 测试表格在 markdown 预览中是否正确渲染
5. 按顺序为问题编号（Q1、Q2、Q3 - 最多 3 个）
6. 在等待响应之前一起呈现所有问题
7. 等待用户回复所有问题的选择（例如，"Q1: A, Q2: Custom - [详情], Q3: B"）
8. 通过用用户选择或提供的内容替换每个 [NEEDS CLARIFICATION] 标记来更新规格说明
9. 在所有澄清解决后重新运行验证

   d. **更新清单**：每次验证迭代后，使用当前的通过/失败状态更新清单文件

7. 报告完成情况，包括分支名称、清单文件路径、清单结果，以及下一阶段的准备情况（`/speckit.clarify` 或 `/speckit.plan`）。

8. **检查扩展钩子**：报告完成后，检查项目根目录中是否存在 `.specify/extensions.yml`。
   - 如果存在，请读取该文件并查找 `hooks.after_specify` 键下的条目
   - 如果 YAML 无法解析或无效，静默跳过钩子检查并正常继续
   - 筛选掉 `enabled` 明确设为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为已启用。
   - 对于每个剩余的钩子，**不要**尝试解释或评估钩子的 `condition` 表达式：
     - 如果钩子没有 `condition` 字段，或该字段为 null/空，则该钩子可执行
     - 如果钩子定义了非空的 `condition`，跳过该钩子，将条件评估留给 HookExecutor 实现
   - 对于每个可执行的钩子，根据其 `optional` 标志输出以下内容：
     - **可选钩子**（`optional: true`）：

```
## Extension Hooks

       **Optional Hook**: {extension}
       Command: `/{command}`
       Description: {description}

       Prompt: {prompt}
       To execute: `/{command}`
```

- **强制钩子** (`optional: false`):

```
## Extension Hooks

       **Automatic Hook**: {extension}
       Executing: `/{command}`
       EXECUTE_COMMAND: {command}
```

- 如果没有注册任何 hook 或者 `.specify/extensions.yml` 不存在，静默跳过

**注意：** 脚本会在写入之前创建并检出新分支，同时初始化规范文件。

## 快速指南

- 关注用户需要**什么**以及**为什么**。
- 避免提及如何实现（不要涉及技术栈、API、代码结构）。
- 面向业务相关方，而非开发人员。
- 不要创建任何嵌入在规范中的检查清单。这将作为单独的指令。

### 部分要求

- **强制性部分**：每个功能都必须完成
- **可选部分**：仅在与功能相关时才包含
- 当某部分不适用时，完全删除（不要保留为"N/A"）

### AI 生成规范

根据用户提示创建规范时：

1. **做出明智的推断**：使用上下文、行业标准和常见模式来填补空白
2. **记录假设**：在假设部分记录合理的默认值
3. **限制澄清数量**：最多使用 3 个 [需要澄清] 标记 —— 仅用于影响功能范围或用户体验的关键决策：
   - 存在多种合理解释且含义不同
   - 没有任何合理的默认值
4. **优先处理澄清**：范围 > 安全/隐私 > 用户体验 > 技术细节
5. **像测试人员一样思考**：每个模糊的需求都应该在"可测试且明确"的检查项上失败
6. **常见需要澄清的方面**（仅在无合理默认值时）：
   - 功能范围和边界（包含/排除特定用例）
   - 用户类型和权限（如果存在多种冲突解释）
   - 安全/合规要求（当法律/财务影响重大时）

**合理默认值示例**（无需询问）：

- 数据保留：该领域的行业标准实践
- 性能目标：除非另有说明，否则使用标准网页/移动应用预期
- 错误处理：用户友好的消息和适当的回退方案
- 认证方式：网页应用使用标准会话或 OAuth2
- 集成模式：使用项目适当的模式（网页服务用 REST/GraphQL，库用函数调用，工具用 CLI 参数等）

### 成功标准指南

成功标准必须：

1. **可衡量**：包含具体指标（时间、百分比、数量、比率）
2. **技术无关**：不提及框架、语言、数据库或工具
3. **以用户为中心**：从用户/业务角度描述结果，而非系统内部
4. **可验证**：无需了解实现细节即可测试/验证

**好的示例**：

- "用户可以在 3 分钟内完成结账"
- "系统支持 10,000 个并发用户"
- "95% 的搜索在 1 秒内返回结果"
- "任务完成率提高 40%"

**差的示例**（面向实现）：

- "API 响应时间低于 200ms"（太技术化，使用"用户即时看到结果"）
- "数据库可处理 1000 TPS"（实现细节，使用面向用户的指标）
- "React 组件高效渲染"（框架特定）
- "Redis 缓存命中率超过 80%"（技术特定）