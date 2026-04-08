---
description: 将现有任务转换为可操作的、依赖关系排序的 GitHub issues，基于可用的设计工件。
tools: ['github/github-mcp-server/issue_write']
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

## 用户输入

```text
$ARGUMENTS
```

## 执行前检查

**检查扩展钩子（在任务转问题转换之前）**：
- 检查项目根目录中是否存在 `.specify/extensions.yml`
- 如果存在，请读取该文件并查找 `hooks.before_taskstoissues` 键下的条目
- 如果 YAML 无法解析或无效，静默跳过钩子检查并继续正常执行
- 筛选掉 `enabled` 明确设置为 `false` 的钩子。对于没有 `enabled` 字段的钩子，默认视为已启用
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子的 `condition` 表达式：
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

- **必需钩子** (`optional: false`):

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

- 如果没有注册任何 hook 或 `.specify/extensions.yml` 不存在，则静默跳过

## 大纲

1. 从仓库根目录运行 `{SCRIPT}` 并解析 FEATURE_DIR 和 AVAILABLE_DOCS 列表。所有路径必须为绝对路径。对于参数中的单引号，如"I'm Groot"，请使用转义语法：例如 'I'\''m Groot'（或尽可能使用双引号："I'm Groot"）。
2. 从执行的脚本中，提取 **tasks** 的路径。
3. 通过运行以下命令获取 Git remote：

```bash
git config --get remote.origin.url
```

> [!注意]
> 仅当远程仓库是 GITHUB URL 时才继续执行后续步骤

1. 对于列表中的每个任务，使用 GitHub MCP 服务器在 Git 远程仓库对应的仓库中创建一个新 Issue。

> [!注意]
> 切勿在与远程 URL 不匹配的仓库中创建 Issue

## 执行后检查

**检查扩展钩子（任务转 Issue 之后）**：

检查项目根目录是否存在 `.specify/extensions.yml`。
- 如果存在，读取该文件并查找 `hooks.after_taskstoissues` 键下的条目
- 如果 YAML 无法解析或无效，静默跳过钩子检查并继续正常执行
- 过滤掉 `enabled` 明确设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为已启用。
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子的 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或该字段为 null/空，则将该钩子视为可执行
  - 如果钩子定义了非空的 `condition`，跳过该钩子，将条件评估交给 HookExecutor 实现
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

- **必需钩子** (`optional: false`)：

```
## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
```

- 如果没有注册任何 hook，或者 `.specify/extensions.yml` 文件不存在，则静默跳过