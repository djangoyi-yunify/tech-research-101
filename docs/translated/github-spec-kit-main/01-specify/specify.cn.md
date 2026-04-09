---
description: 从自然语言功能描述创建或更新功能规格说明。
handoffs:
  - label: 构建技术方案
    agent: speckit.plan
    prompt: 为规格说明制定计划。我正在构建...
  - label: 明确规格需求
    agent: speckit.clarify
    prompt: 明确规格说明需求
    send: true
---

## 用户输入

```text
$ARGUMENTS
```

我注意到您提供的是关于扩展钩子检查的工作流程说明，但没有提供具体的用户输入或任务请求。

由于我无法访问文件系统，无法检查项目根目录中是否存在 `.specify/extensions.yml` 文件。

请提供您想要我执行的具体任务，例如：

- 您希望我翻译的文档内容
- 需要我分析或处理的具体文件
- 其他您需要帮助的任务

有了具体的内容后，我会按照您提供的检查流程（考虑用户输入、检查扩展钩子）来执行相应的任务。

```
## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **必选 hook** (`optional: false`):

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

- 如果没有注册钩子或 `.specify/extensions.yml` 不存在，则静默跳过

## 概述

触发消息中用户在 `/speckit.specify` 后输入的文本**就是**功能描述。假设你在此对话中始终可以访问它，即使下面出现字面量 `{ARGS}`。除非用户提供了空命令，否则不要要求用户重复。

给定该功能描述，执行以下操作：

1. **生成简洁的短名称**（2-4个单词）：
   - 分析功能描述并提取最有意义的关键词
   - 创建一个2-4个单词的短名称，捕捉功能的核心
   - 尽可能使用动作-名词格式（例如，"add-user-auth"、"fix-payment-bug"）
   - 保留技术术语和缩写（OAuth2、API、JWT等）
   - 保持简洁但足以让人一眼理解该功能
   - 示例：
     - "I want to add user authentication" → "user-auth"
     - "Implement OAuth2 integration for the API" → "oauth2-api-integration"
     - "Create a dashboard for analytics" → "analytics-dashboard"
     - "Fix payment processing timeout bug" → "fix-payment-timeout"

2. **创建分支**（可选，通过钩子）：

   如果上述预检查中成功运行了 `before_specify` 钩子，它将已创建/切换到 git 分支并输出包含 `BRANCH_NAME` 和 `FEATURE_NUM` 的 JSON。请记下这些值以供参考，但分支名称**不会**决定规格目录名称。

   如果用户明确提供了 `GIT_BRANCH_NAME`，请将其传递给钩子，以便分支脚本使用的确切值作为分支名称（绕过所有前缀/后缀生成）。

3. **创建规格功能目录**：

   规格生活在默认的 `specs/` 目录下，除非用户明确提供 `SPECIFY_FEATURE_DIRECTORY`。

   **`SPECIFY_FEATURE_DIRECTORY` 的解析顺序**：
   1. 如果用户明确提供了 `SPECIFY_FEATURE_DIRECTORY`（例如，通过环境变量、参数或配置），则按原样使用它
   2. 否则，在 `specs/` 下自动生成：
      - 检查 `.specify/init-options.json` 中的 `branch_numbering`
      - 如果是 `"timestamp"`：前缀为 `YYYYMMDD-HHMMSS`（当前时间戳）
      - 如果是 `"sequential"` 或不存在：前缀为 `NNN`（扫描 `specs/` 中现有目录后的下一个可用3位数字）
      - 构建目录名称：`<prefix>-<short-name>`（例如，`003-user-auth` 或 `20260319-143022-user-auth`）
      - 将 `SPECIFY_FEATURE_DIRECTORY` 设置为 `specs/<directory-name>`

   **创建目录和规格文件**：
   - `mkdir -p SPECIFY_FEATURE_DIRECTORY`
   - 将 `templates/spec-template.md` 复制到 `SPECIFY_FEATURE_DIRECTORY/spec.md` 作为起点
   - 将 `SPEC_FILE` 设置为 `SPECIFY_FEATURE_DIRECTORY/spec.md`
   - 将解析后的路径持久化到 `.specify/feature.json`：

```json
{
       "feature_directory": "<resolved feature dir>"
     }
```

写入实际的已解析目录路径值（例如 `specs/003-user-auth`），而不是字面字符串 `SPECIFY_FEATURE_DIRECTORY`。
这样下游命令（`/speckit.plan`、`/speckit.tasks` 等）就可以定位特性目录，而无需依赖 git 分支名称约定。

**重要提示**：
- 每次 `/speckit.specify` 调用只能创建一个特性
- 特性目录名称和 git 分支名称是独立的——它们可能相同，但这取决于用户的选择
- 特性目录和文件始终由此命令创建，绝不是由钩子创建

4. 加载 `templates/spec-template.md` 以了解所需部分。

5. 按照此执行流程：
   1. 从参数中解析用户描述
      如果为空：错误 "未提供特性描述"
   2. 从描述中提取关键概念
      识别：参与者、动作、数据、约束
   3. 对于不明确的方面：
      - 根据上下文和行业标准做出合理推断
      - 仅在以下情况标记 [NEEDS CLARIFICATION：具体问题]：
        - 选择会显著影响特性范围或用户体验
        - 存在多种合理的解释方式，且各自有不同的影响
        - 没有合理的默认值
      - **限制：最多 3 个 [NEEDS CLARIFICATION] 标记**
      - 按影响程度优先处理澄清：范围 > 安全/隐私 > 用户体验 > 技术细节
   4. 填写用户场景和测试部分
      如果没有清晰的用户流程：错误 "无法确定用户场景"
   5. 生成功能需求
      每个需求必须是可测试的
      对于未指定的细节使用合理的默认值（在假设部分记录假设）
   6. 定义成功标准
      创建可衡量的、技术无关的结果
      包括定量指标（时间、性能、数量）和定性衡量（用户满意度、任务完成度）
      每个标准都必须在没有实现细节的情况下可验证
   7. 识别关键实体（如果涉及数据）
   8. 返回：成功（规格已准备好进行规划）

6. 使用模板结构将规格写入 SPEC_FILE，用从特性描述（参数）中派生的具体细节替换占位符，同时保留部分顺序和标题。

7. **规格质量验证**：在编写初始规格后，根据质量标准进行验证：

   a. **创建规格质量检查清单**：在 `SPECIFY_FEATURE_DIRECTORY/checklists/requirements.md` 处使用检查清单模板结构生成检查清单文件，包含以下验证项：

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

b. **运行验证检查**: 根据每个检查清单项审查规范：

- 对每个项目确定其是否通过
- 记录发现的具体问题（引用相关规范章节）

c. **处理验证结果**：

- **如果所有项目通过**：标记检查清单完成并进入步骤 7

- **如果项目失败（不包括 [NEEDS CLARIFICATION]）**：
  1. 列出失败的项目和具体问题
  2. 更新规范以解决每个问题
  3. 重新运行验证直到所有项目通过（最多 3 次迭代）
  4. 如果 3 次迭代后仍然失败，在检查清单备注中记录剩余问题并警告用户

- **如果 [NEEDS CLARIFICATION] 标记仍然存在**：
  1. 从规范中提取所有 [NEEDS CLARIFICATION: ...] 标记
  2. **限制检查**：如果标记超过 3 个，仅保留 3 个最关键的（按范围/安全/用户体验影响排序），其余的做合理假设
  3. 对每个需要澄清的内容（最多 3 个），按以下格式向用户呈现选项：

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

4. **重要 - 表格格式**：确保 markdown 表格格式正确：
   - 使用一致的管道符对齐间距
   - 每个单元格内容两侧应有空格：`| Content |` 而非 `|Content|`
   - 标题分隔符至少需要 3 个破折号：`|--------|`
   - 测试表格在 markdown 预览中是否正确渲染
5. 按顺序编号问题（Q1、Q2、Q3 - 最多 3 个）
6. 在等待响应之前将所有问题一起呈现
7. 等待用户回复所有问题的选择（例如，"Q1: A, Q2: Custom - [details], Q3: B"）
8. 用用户选择或提供的答案替换每个 [NEEDS CLARIFICATION] 标记来更新规格说明
9. 解决所有澄清问题后重新运行验证

   d. **更新检查清单**：每次验证迭代后，使用当前的通过/失败状态更新检查清单文件

8. **向用户报告完成情况**，包括：
   - `SPECIFY_FEATURE_DIRECTORY` — 特性目录路径
   - `SPEC_FILE` — 规格说明文件路径
   - 检查清单结果摘要
   - 下一阶段的准备状态（`/speckit.clarify` 或 `//speckit.plan`）

9. **检查扩展钩子**：报告完成后，检查项目根目录中是否存在 `.specify/extensions.yml`。
   - 如果存在，读取该文件并查找 `hooks.after_specify` 键下的条目
   - 如果无法解析 YAML 或该文件无效，静默跳过钩子检查并继续正常执行
   - 过滤掉 `enabled` 显式设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为启用。
   - 对于每个剩余的钩子，**不要**尝试解释或求值钩子的 `condition` 表达式：
     - 如果钩子没有 `condition` 字段，或者该字段为 null/空，则将钩子视为可执行
     - 如果钩子定义了非空的 `condition`，则跳过该钩子，将条件求值留给 HookExecutor 实现
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

- **必须的 hook** (`optional: false`)：

```
## Extension Hooks

       **Automatic Hook**: {extension}
       Executing: `/{command}`
       EXECUTE_COMMAND: {command}
```

- 如果没有注册任何钩子或 `.specify/extensions.yml` 不存在，静默跳过

**注意：** 分支创建由 `before_specify` 钩子处理（git 扩展）。规范目录和文件创建始终由此核心命令处理。

## 快速指南

- 关注用户需要什么（**WHAT**）以及为什么（**WHY**）
- 避免如何实现（HOW）（不涉及技术栈、API、代码结构）
- 面向业务利益相关者，而非开发者
- **不要创建任何嵌入规范中的检查清单**。这将是一个单独的 命令

### 章节要求

- **必填章节**：每个功能都必须完成
- **可选章节**：仅在与功能相关时包含
- 当某章节不适用时，完全删除（不要留作"N/A"）

### AI 生成指南

根据用户提示创建规范时：

1. **做出合理推断**：利用上下文、行业标准和常见模式来填补空白
2. **记录假设**：在"假设"章节中记录合理的默认值
3. **限制澄清需求**：最多使用 3 个 [需要澄清] 标记 —— 仅用于影响重大的关键决策：
   - 显著影响功能范围或用户体验
   - 存在多种合理解读且各有不同影响
   - 没有任何合理的默认值
4. **优先处理澄清需求**：范围 > 安全/隐私 > 用户体验 > 技术细节
5. **像测试人员一样思考**：每个模糊的需求都应该在"可测试且明确"的检查项中失败
6. **常见需要澄清的领域**（仅在无合理默认值时）：
   - 功能范围和边界（包含/排除特定用例）
   - 用户类型和权限（如果存在多种冲突解读）
   - 安全/合规要求（当涉及法律/财务重大影响时）

**合理默认值示例**（无需询问）：

- 数据保留：遵循该领域的行业标准实践
- 性能目标：除非另有说明，否则使用标准 Web/移动应用预期
- 错误处理：用户友好的消息和适当的回退机制
- 认证方式：Web 应用使用标准会话机制或 OAuth2
- 集成模式：使用项目适当的模式（Web 服务用 REST/GraphQL，库用函数调用，工具用 CLI 参数等）

### 成功标准指南

成功标准必须：

1. **可衡量**：包含具体指标（时间、百分比、数量、比率）
2. **技术无关**：不提及框架、语言、数据库或工具
3. **以用户为中心**：从用户/业务角度描述成果，而非系统内部
4. **可验证**：无需了解实现细节即可测试/验证

**良好示例**：

- "用户可在 3 分钟内完成结账"
- "系统支持 10,000 名并发用户"
- "95% 的搜索在 1 秒内返回结果"
- "任务完成率提升 40%"

**不良示例**（以实现为中心）：

- "API 响应时间低于 200ms"（过于技术化，应使用"用户即时看到结果"）
- "数据库可处理 1000 TPS"（实现细节，应使用面向用户的指标）
- "React 组件高效渲染"（特定于框架）
- "Redis 缓存命中率超过 80%"（特定于技术）