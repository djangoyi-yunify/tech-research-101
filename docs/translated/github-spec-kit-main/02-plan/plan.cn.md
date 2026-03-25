---
description: 使用计划模板执行实施规划工作流，生成设计工件。
handoffs:
  - label: 创建任务
    agent: speckit.tasks
    prompt: 将计划分解为任务
    send: true
  - label: 创建检查清单
    agent: speckit.checklist
    prompt: 为以下领域创建检查清单...
scripts:
  sh: scripts/bash/setup-plan.sh --json
  ps: scripts/powershell/setup-plan.ps1 -Json
agent_scripts:
  sh: scripts/bash/update-agent-context.sh __AGENT__
  ps: scripts/powershell/update-agent-context.ps1 -AgentType __AGENT__
---

## 用户输入

```text
$ARGUMENTS
```

## 执行前检查

**检查扩展钩子（在规划之前）**：

- 检查项目根目录下是否存在 `.specify/extensions.yml` 文件。
- 如果存在，请读取该文件并查找 `hooks.before_plan` 键下的条目。
- 如果 YAML 无法解析或无效，请静默跳过钩子检查并继续正常执行。
- 筛选掉 `enabled` 显式设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认为启用。
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或者为 null/空，则将该钩子视为可执行。
  - 如果钩子定义了非空的 `condition`，请跳过该钩子，将条件评估留给 HookExecutor 实现。
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

- **强制钩子**（`optional: false`）：

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

- 如果没有注册任何钩子或 `.specify/extensions.yml` 不存在，则静默跳过

## 大纲

1. **设置**：从仓库根目录运行 `{SCRIPT}` 并解析 JSON 以获取 FEATURE_SPEC、IMPL_PLAN、SPECS_DIR、BRANCH。对于参数中的单引号（如 "I'm Groot"），请使用转义语法：例如 'I'\''m Groot'（或尽可能使用双引号："I'm Groot"）。

2. **加载上下文**：读取 FEATURE_SPEC 和 `/memory/constitution.md`。加载 IMPL_PLAN 模板（已复制）。

3. **执行计划工作流**：按照 IMPL_PLAN 模板的结构执行：
   - 填写技术上下文（将未知项标记为"需要澄清"）
   - 根据宪法填写宪法检查部分
   - 评估门禁（如果违规无法解释则报错）
   - 阶段 0：生成 research.md（解决所有需要澄清的问题）
   - 阶段 1：生成 data-model.md、contracts/、quickstart.md
   - 阶段 1：通过运行代理脚本更新代理上下文
   - 设计后重新评估宪法检查

4. **停止并报告**：命令在阶段 2 规划后结束。报告分支、IMPL_PLAN 路径和生成的工件。

5. **检查扩展钩子**：报告后，检查项目根目录是否存在 `.specify/extensions.yml`。
   - 如果存在，读取该文件并查找 `hooks.after_plan` 键下的条目
   - 如果无法解析 YAML 或格式无效，静默跳过钩子检查并正常继续
   - 过滤掉 `enabled` 明确设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为启用。
   - 对于每个剩余的钩子，**不要**尝试解释或求值钩子的 `condition` 表达式：
     - 如果钩子没有 `condition` 字段，或该字段为空/空值，则将该钩子视为可执行
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

- **必选钩子** (`可选: 否`):

```
## Extension Hooks

       **Automatic Hook**: {extension}
       Executing: `/{command}`
       EXECUTE_COMMAND: {command}
```

- 如果没有注册任何 hooks 或 `.specify/extensions.yml` 不存在，静默跳过

## 阶段

### 阶段 0：概要与研究

1. **从上面的技术上下文中提取未知内容**：
   - 每个需要澄清的事项 → 研究任务
   - 每个依赖项 → 最佳实践任务
   - 每个集成 → 模式任务

2. **生成并调度研究代理**：

```text
For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
```

3. 在 `research.md` 中**整合发现**，使用以下格式：
   - Decision: [选择了什么]
   - Rationale: [为什么选择]
   - Alternatives considered: [还评估了哪些]

**输出**: 包含所有待澄清问题已解决的 research.md

### 阶段 1：设计与契约

**前置条件：** `research.md` 完成

1. **从功能规范中提取实体** → `data-model.md`：
   - 实体名称、字段、关系
   - 需求中的验证规则
   - 状态转换（如适用）

2. **定义接口契约**（如果项目有外部接口） → `/contracts/`：
   - 识别项目向用户或其他系统暴露的接口
   - 记录适合项目类型的契约格式
   - 示例：库的公共 API、CLI 工具的命令模式、Web 服务的端点、解析器的语法、应用的用户界面契约
   - 如果项目是纯内部（构建脚本、一次性工具等）则跳过

3. **更新 Agent 上下文**：
   - 运行 `{AGENT_SCRIPT}`
   - 这些脚本会检测正在使用的 AI Agent
   - 更新相应的 Agent 特定上下文文件
   - 只添加当前计划中的新技术
   - 保留标记之间的手动添加内容

**输出**: data-model.md, /contracts/*, quickstart.md, Agent 特定文件

## 关键规则

- 使用绝对路径
- 遇到门控失败或未解决的澄清问题时**报错**