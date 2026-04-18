---
description: 使用计划模板执行实现规划工作流，以生成设计工件。
handoffs:
  - label: Create Tasks
    agent: speckit.tasks
    prompt: 将计划拆分为任务
    send: true
  - label: Create Checklist
    agent: speckit.checklist
    prompt: 为以下领域创建检查清单...
scripts:
  sh: scripts/bash/setup-plan.sh --json
  ps: scripts/powershell/setup-plan.ps1 -Json
---

## 用户输入

```text
$ARGUMENTS
```

在继续之前，你**必须**考虑用户输入（如果非空）。

## 执行前检查

**检查扩展钩子（在计划之前）**：
- 检查项目根目录中是否存在 `.specify/extensions.yml` 文件。
- 如果存在，读取该文件并查找 `hooks.before_plan` 键下的条目。
- 如果 YAML 无法解析或无效，静默跳过钩子检查并正常继续。
- 筛选出 `enabled` 明确设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为已启用。
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或该字段为 null/空，则将该钩子视为可执行。
  - 如果钩子定义了非空 `condition`，则跳过该钩子，将条件评估留给 HookExecutor 实现。
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

- **强制钩子** (`optional: false`)：

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

- 如果没有注册任何钩子，或 `.specify/extensions.yml` 不存在，静默跳过

## 大纲

1. **设置**：从仓库根目录运行 `{SCRIPT}` 并解析 JSON 获取 FEATURE_SPEC、IMPL_PLAN、SPECS_DIR、BRANCH。对于参数中的单引号（如 "I'm Groot"），使用转义语法：例如 `'I'\''m Groot'`（或尽可能使用双引号："I'm Groot"）。

2. **加载上下文**：读取 FEATURE_SPEC 和 `/memory/constitution.md`。加载 IMPL_PLAN 模板（已复制）。

3. **执行计划工作流**：按照 IMPL_PLAN 模板的结构执行：
   - 填写技术上下文（将未知项标记为 "NEEDS CLARIFICATION"）
   - 从 constitution 填写 Constitution Check 部分
   - 评估门控（如有违规且无正当理由则报 ERROR）
   - 阶段 0：生成 research.md（解决所有 NEEDS CLARIFICATION）
   - 阶段 1：生成 data-model.md、contracts/、quickstart.md
   - 阶段 1：通过运行 agent 脚本更新 agent 上下文
   - 设计完成后重新评估 Constitution Check

4. **停止并报告**：Phase 2 规划完成后命令结束。报告 branch、IMPL_PLAN 路径和生成的产物。

5. **检查扩展钩子**：报告后，检查项目根目录中是否存在 `.specify/extensions.yml`。
   - 如果存在，读取该文件并查找 `hooks.after_plan` 键下的条目
   - 如果 YAML 无法解析或格式无效，静默跳过钩子检查并正常继续
   - 筛选掉 `enabled` 明确设为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为启用。
   - 对于每个剩余的钩子，**不要**尝试解释或求值钩子的 `condition` 表达式：
     - 如果钩子没有 `condition` 字段，或其为 null/空，则视该钩子为可执行
     - 如果钩子定义了非空 `condition`，则跳过该钩子，将 condition 求值留给 HookExecutor 实现
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

- **强制钩子** (`optional: false`):

```
## Extension Hooks

       **Automatic Hook**: {extension}
       Executing: `/{command}`
       EXECUTE_COMMAND: {command}
```

- 如果没有注册任何 hooks 或 `.specify/extensions.yml` 不存在，则静默跳过

## 阶段

### 阶段 0：概述与研究

1. **从上述技术上下文中提取未知内容**：
   - 对于每个需要澄清的内容 → 研究任务
   - 对于每个依赖项 → 最佳实践任务
   - 对于每个集成 → 模式任务

2. **生成并分派研究代理**：

```text
For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
```

3. **整合研究结果** 到 `research.md`，使用以下格式：
   - Decision: [选择方案]
   - Rationale: [选择原因]
   - Alternatives considered: [其他评估方案]

**输出**: research.md（所有 NEEDS CLARIFICATION 已解决）

### 阶段 1：设计与契约

**前置条件：** `research.md` 已完成

1. **从功能规格中提取实体** → `data-model.md`：
   - 实体名称、字段、关系
   - 需求中的验证规则
   - 适用的状态转换

2. **定义接口契约**（如果项目有外部接口）→ `/contracts/`：
   - 识别项目向用户或其他系统暴露的接口
   - 记录适合该项目类型的契约格式
   - 示例：库的公共 API、CLI 工具的命令模式、Web 服务的端点、解析器的语法定义、应用程序的 UI 契约
   - 如果项目是纯内部项目（构建脚本、一次性工具等）则跳过

3. **更新 Agent 上下文**：
   - 在 `__CONTEXT_FILE__` 的 `<!-- SPECKIT START -->` 和 `<!-- SPECKIT END -->` 标记之间，更新计划引用以指向步骤 1 创建的计划文件（IMPL_PLAN 路径）

**输出**: data-model.md、/contracts/*、quickstart.md、更新后的 agent 上下文文件

## 关键规则

- 文件系统操作用绝对路径；文档和 agent 上下文文件中的引用用项目相对路径
- 门控检查失败或未解决的澄清问题应报错