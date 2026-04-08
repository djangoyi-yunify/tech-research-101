---
description: 根据交互式或提供的原则输入创建或更新项目章程，确保所有依赖模板保持同步。
handoffs: 
  - label: Build Specification
    agent: speckit.specify
    prompt: 根据更新后的章程实现功能规范。我想要构建...
---

## 用户输入

```text
$ARGUMENTS
```

# 预执行检查

## 执行前检查

**检查扩展钩子（在章程更新之前）**：

- 检查项目根目录是否存在 `.specify/extensions.yml`
- 如果存在，读取该文件并查找 `hooks.before_constitution` 键下的条目
- 如果 YAML 无法解析或无效，静默跳过钩子检查并继续正常执行
- 过滤掉 `enabled` 明确设置为 `false` 的钩子。没有 `enabled` 字段的钩子默认视为已启用
- 对于每个剩余的钩子，**不要**尝试解释或评估钩子的 `condition` 表达式：
  - 如果钩子没有 `condition` 字段，或该字段为空/空值，则将该钩子视为可执行
  - 如果钩子定义了非空的 `condition`，跳过该钩子，将条件评估留给 HookExecutor 实现
- 对于每个可执行的钩子，根据其 `optional` 标志输出以下内容：
  - **可选钩子** (`optional: true`)：

```
## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **必需 hook**（`optional: false`）：

```
## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
```

I'll help you update the project constitution. Let me start by checking the current state of the constitution file.

```bash
cat .specify/memory/constitution.md 2>/dev/null || echo "FILE_NOT_FOUND"
```

Let me also check if there's a template to work from:

```bash
cat .specify/templates/constitution-template.md 2>/dev/null || echo "TEMPLATE_NOT_FOUND"
```

```
## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
```

- **必需钩子** (`optional: false`):

```
## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
```

如果没有注册任何 hooks 或 `.specify/extensions.yml` 不存在，则静默跳过