---
description: 从自然语言特性描述创建或更新特性规格文档。
handoffs: 
  - label: 构建技术计划
    agent: speckit.plan
    prompt: 为规格文档创建计划。我正在使用...构建
  - label: 澄清规格需求
    agent: speckit.clarify
    prompt: 澄清规格需求
    send: true
---

## 用户输入

```text
$ARGUMENTS
```

# 项目规范执行前检查与钩子处理流程

## 执行前检查

### 扩展钩子检查（在规范之前）

在执行规范之前，系统需要完成以下检查流程：

1. **配置文件定位**：检查项目根目录是否存在 `.specify/extensions.yml` 文件
2. **文件读取与解析**：如果配置文件存在，读取其内容并查找 `hooks.before_specify` 键下的条目
3. **异常处理**：如果 YAML 文件无法解析或格式无效，静默跳过钩子检查流程并继续正常执行后续步骤
4. **钩子过滤规则**：
   - 过滤掉 `enabled` 明确设置为 `false` 的钩子
   - 对于没有 `enabled` 字段的钩子，默认视为已启用

### 条件表达式评估规则

对于每个待处理的钩子，**不要**尝试解释或评估其 `condition` 条件表达式：

- **可执行条件**：如果钩子没有 `condition` 字段，或该字段为 null/空值，则将该钩子标记为可执行
- **需评估条件**：如果钩子定义了非空 `condition`，则跳过该钩子，不进行条件判断，并将条件评估工作委托给 HookExecutor 实现类处理

### 钩子输出规则

对于每个可执行的钩子，根据其 `optional` 标志输出相应的处理信息：

- **可选钩子**（`optional: true`）：
- **必需钩子**（`optional: false` 或未指定）：

```
## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **Mandatory hook** (`optional: false`):

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

- 如果没有注册任何 hook 或 `.specify/extensions.yml` 不存在，则静默跳过

## 概述

用户在触发消息中 `__SPECKIT_COMMAND_SPECIFY__` 之后输入的文本**就是**功能描述。假设你在本次对话中始终可以获取到该描述，即使下面 `{ARGS}` 是字面形式。除非用户输入了空命令，否则不要要求用户重复。

根据上述功能描述，执行以下步骤：

1. **生成简洁的短名称**（2-4 个词）：
   - 分析功能描述，提取最具意义的关键词
   - 创建一个 2-4 词的短名称，准确捕捉功能的核心要点
   - 尽可能使用"动作-名词"格式（如 "add-user-auth"、"fix-payment-bug"）
   - 保留技术术语和缩写（OAuth2、API、JWT 等）
   - 保持简洁但足以让人一眼理解功能含义
   - 示例：
     - "I want to add user authentication" → "user-auth"
     - "Implement OAuth2 integration for the API" → "oauth2-api-integration"
     - "Create a dashboard for analytics" → "analytics-dashboard"
     - "Fix payment processing timeout bug" → "fix-payment-timeout"

2. **创建分支**（可选，通过 hook）：

   如果 `Pre-Execution Checks` 中有 `before_specify` hook 成功运行，它会创建/切换到 git 分支并输出包含 `BRANCH_NAME` 和 `FEATURE_NUM` 的 JSON。请记录这些值作为参考，但分支名**不**决定 spec 目录名称。

   如果用户明确提供了 `GIT_BRANCH_NAME`，请将其传递给 hook，以便分支脚本使用该确切值作为分支名称（绕过所有前缀/后缀生成逻辑）。

3. **创建 spec 功能目录**：

   Spec 默认存放在 `specs/` 目录下，除非用户明确提供 `SPECIFY_FEATURE_DIRECTORY`。

   **`SPECIFY_FEATURE_DIRECTORY` 的解析顺序**：
   1. 如果用户明确提供了 `SPECIFY_FEATURE_DIRECTORY`（例如通过环境变量、参数或配置），直接使用
   2. 否则，在 `specs/` 下自动生成：
      - 检查 `.specify/init-options.json` 中的 `branch_numbering` 配置
      - 如果为 `"timestamp"`：前缀为 `YYYYMMDD-HHMMSS`（当前时间戳）
      - 如果为 `"sequential"` 或不存在：前缀为 `NNN`（扫描 `specs/` 中现有目录后可用的下一个 3 位数字）
      - 构建目录名称：`<前缀>-<短名称>`（例如 `003-user-auth` 或 `20260319-143022-user-auth`）
      - 将 `SPECIFY_FEATURE_DIRECTORY` 设置为 `specs/<目录名称>`

   **创建目录和 spec 文件**：
   - `mkdir -p SPECIFY_FEATURE_DIRECTORY`
   - 将 `templates/spec-template.md` 复制到 `SPECIFY_FEATURE_DIRECTORY/spec.md` 作为起点
   - 将 `SPEC_FILE` 设置为 `SPECIFY_FEATURE_DIRECTORY/spec.md`
   - 将解析后的路径持久化到 `.specify/feature.json`：

```json
{
       "feature_directory": "<resolved feature dir>"
     }
```

编写实际的已解析目录路径值（例如 `specs/003-user-auth`），而不是字面字符串 `SPECIFY_FEATURE_DIRECTORY`。
这允许下游命令（`__SPECKIT_COMMAND_PLAN__`、`__SPECKIT_COMMAND_TASKS__` 等）无需依赖 git 分支命名约定即可定位 feature 目录。

**重要**：
- 每次 `__SPECKIT_COMMAND_SPECIFY__` 调用只能创建一个 feature
- spec 目录名和 git 分支名是独立的——它们可能相同，但这取决于用户的选择
- spec 目录和文件始终由此命令创建，而非由 hook 创建

4. 加载 `templates/spec-template.md` 以了解所需部分。

5. 遵循此执行流程：
    1. 从参数解析用户描述
       如果为空：ERROR "未提供 feature 描述"
    2. 从描述中提取关键概念
       识别：角色、操作、数据、约束
    3. 对于不明确的方面：
       - 根据上下文和行业标准做出合理推断
       - 仅在以下情况下标记 [NEEDS CLARIFICATION: 具体问题]：
         - 该选择显著影响 feature 范围或用户体验
         - 存在多个具有不同含义的合理解释
         - 不存在合理的默认值
       - **限制：最多 3 个 [NEEDS CLARIFICATION] 标记**
       - 按影响程度优先处理澄清：范围 > 安全/隐私 > 用户体验 > 技术细节
    4. 填写用户场景与测试部分
       如果没有清晰的用户流程：ERROR "无法确定用户场景"
    5. 生成功能需求
       每个需求必须可测试
       对未指定的细节使用合理默认值（在假设部分记录假设）
    6. 定义成功标准
       创建可衡量的、技术无关的结果
       包含定量指标（时间、性能、容量）和定性指标（用户满意度、任务完成度）
       每个标准必须可在无实现细节的情况下验证
    7. 识别关键实体（如果涉及数据）
    8. 返回：SUCCESS（spec 已准备好进行规划）

6. 使用模板结构将规范写入 SPEC_FILE，将占位符替换为从 feature 描述（参数）派生的具体细节，同时保留部分顺序和标题。

7. **规范质量验证**：在编写初始 spec 后，根据质量标准进行验证：

   a. **创建规范质量检查清单**：使用检查清单模板结构在 `SPECIFY_FEATURE_DIRECTORY/checklists/requirements.md` 生成检查清单文件，包含以下验证项：

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
      
      - Items marked incomplete require spec updates before `__SPECKIT_COMMAND_CLARIFY__` or `__SPECKIT_COMMAND_PLAN__`
```

b. **运行验证检查**：对照清单中的每个条目审查规范：
   - 对每个条目，判断其是否通过
   - 记录发现的特定问题（引用相关规范章节）

c. **处理验证结果**：

   - **如果所有条目均通过**：标记清单完成并进入步骤 7

   - **如果条目失败（不包括 [NEEDS CLARIFICATION]）**：
     1. 列出失败的条目及具体问题
     2. 更新规范以解决每个问题
     3. 重新运行验证直到所有条目通过（最多 3 次迭代）
     4. 如果 3 次迭代后仍有失败项，将剩余问题记录在清单备注中并提醒用户

   - **如果存在 [NEEDS CLARIFICATION] 标记**：
     1. 从规范中提取所有 [NEEDS CLARIFICATION: ...] 标记
     2. **限制检查**：如果标记超过 3 个，仅保留最关键的 3 个（按范围/安全性/UX 影响排序），其余根据合理推断处理
     3. 对于每个需要澄清的问题（最多 3 个），按以下格式向用户呈现选项：

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
           - 使用统一的间距，竖线对齐
           - 每个单元格内容两侧应有空格：`| Content |` 而非 `|Content|`
           - 表头分隔符必须至少有 3 个横杠：`|--------|`
           - 测试表格在 markdown 预览中是否正确渲染
        5. 问题按顺序编号（Q1、Q2、Q3 - 最多 3 个）
        6. 在等待响应前将所有问题一起呈现
        7. 等待用户响应所有问题的选择（例如："Q1: A, Q2: Custom - [details], Q3: B"）
        8. 用用户选择或提供的答案替换每个 [NEEDS CLARIFICATION] 标记来更新规格
        9. 所有澄清解决后重新运行验证

   d. **更新清单**：每次验证迭代后，更新清单文件，记录当前的通过/失败状态

8. **向用户报告完成**，内容包括：
   - `SPECIFY_FEATURE_DIRECTORY` — 功能目录路径
   - `SPEC_FILE` — 规格文件路径
   - 清单结果摘要
   - 下一阶段的准备状态（`__SPECKIT_COMMAND_CLARIFY__` 或 `__SPECKIT_COMMAND_PLAN__`）

9. **检查扩展钩子**：报告完成后，检查项目根目录是否存在 `.specify/extensions.yml`。
   - 如果存在，读取它并查找 `hooks.after_specify` 键下的条目
   - 如果 YAML 无法解析或无效，静默跳过钩子检查并正常继续
   - 过滤掉 `enabled` 明确设置为 `false` 的钩子。对于没有 `enabled` 字段的钩子，默认视为已启用。
   - 对于每个剩余的钩子，**不要**尝试解释或评估钩子 `condition` 表达式：
     - 如果钩子没有 `condition` 字段，或为 null/空，则视为可执行
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

**注意：** 分支创建由 `before_specify` hook（git extension）处理。Spec 目录和文件创建始终由此核心命令处理。

## 快速指南

- 聚焦用户需要**什么**以及**为什么**
- 避免 HOW（如何实现），不涉及技术栈、API、代码结构
- 为业务相关方编写，而非开发者
- 不要在 spec 中嵌入任何检查清单，那将是独立的命令

### 章节要求

- **必填部分**：每个功能都必须完成
- **可选部分**：仅在功能相关时包含
- 当某章节不适用时，完全删除（不要留"N/A"）

### AI 生成规范

从用户提示创建规范时：

1. **做出合理推断**：使用上下文、行业标准和常见模式填补空白
2. **记录假设**：在假设部分记录合理的默认值
3. **限制澄清标记**：最多 3 个 [需要澄清] 标记 - 仅用于以下关键决策：
   - 显著影响功能范围或用户体验
   - 存在多种合理解释且有不同影响
   - 没有任何合理的默认值
4. **优先澄清顺序**：范围 > 安全/隐私 > 用户体验 > 技术细节
5. **像测试人员一样思考**：每个模糊需求都应无法通过"可测试且明确"检查项
6. **常见需要澄清的领域**（仅在无合理默认值时使用）：
   - 功能范围和边界（包含/排除特定用例）
   - 用户类型和权限（如果存在多种冲突解释）
   - 安全/合规要求（当具有法律/财务影响时）

**合理默认值示例**（无需询问）：

- 数据保留：该领域的行业标准实践
- 性能目标：除非另有说明，否则使用标准 Web/移动应用期望
- 错误处理：用户友好的消息和适当的回退方案
- 认证方法：Web 应用的标准会话或 OAuth2
- 集成模式：使用项目适当的模式（Web 服务用 REST/GraphQL，库用函数调用，工具用 CLI 参数等）

### 成功标准指南

成功标准必须：

1. **可衡量**：包含具体指标（时间、百分比、数量、速率）
2. **技术无关**：不提及框架、语言、数据库或工具
3. **用户导向**：从用户/业务角度描述结果，而非系统内部
4. **可验证**：无需了解实现细节即可测试/验证

**好的示例**：

- "用户可在 3 分钟内完成结账"
- "系统支持 10,000 并发用户"
- "95% 的搜索在 1 秒内返回结果"
- "任务完成率提升 40%"

**差的示例**（面向实现）：

- "API 响应时间低于 200ms"（过于技术化，应使用"用户即时看到结果"）
- "数据库可处理 1000 TPS"（实现细节，应使用面向用户的指标）
- "React 组件高效渲染"（框架特定）
- "Redis 缓存命中率超过 80%"（技术特定）