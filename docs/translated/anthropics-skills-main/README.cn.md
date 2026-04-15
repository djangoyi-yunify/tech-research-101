> **注意：** 此仓库包含 Anthropic 为 Claude 开发的技能实现。如需了解 Agent Skills 标准，请参阅 [agentskills.io](http://agentskills.io)。

# Skills

Skills 是由指令、脚本和资源组成的文件夹，Claude 可以动态加载这些内容来提升特定任务的表现。Skills 教会 Claude 如何以可重复的方式完成特定任务，无论是按照公司品牌规范创建文档、使用组织特定的工作流程分析数据，还是自动化个人任务。

如需更多信息，请参阅：

- [什么是 Skills？](https://support.claude.com/en/articles/12512176-what-are-skills)
- [在 Claude 中使用 Skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [如何创建自定义 Skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [用 Agent Skills 为真实世界的 Agent 赋能](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

# 关于此仓库

此仓库包含展示 Claude Skills 系统可能性的 Skills。这些 Skills 涵盖创意应用（艺术、音乐、设计）、技术任务（测试 Web 应用、MCP 服务器生成）以及企业工作流程（沟通、品牌等）。

每个 Skill 都独立存在于各自的文件夹中，包含一个 `SKILL.md` 文件，其中包含 Claude 使用的指令和元数据。浏览这些 Skills 可以为您的自定义 Skills 提供灵感，或帮助您理解不同的模式和方案。

此仓库中的许多 Skills 都是开源的（Apache 2.0）。我们还包含了为 [Claude 的文档功能](https://www.anthropic.com/news/create-files)提供支持的文档创建和编辑 Skills，位于 [`skills/docx`](./skills/docx)、[`skills/pdf`](./skills/pdf)、[`skills/pptx`](./skills/pptx) 和 [`skills/xlsx`](./skills/xlsx) 子文件夹中。这些是源码可用（source-available）而非开源的，但我们希望与开发者分享，作为在实际 AI 应用中使用的更复杂 Skills 的参考。

## 免责声明

**这些 Skills 仅供演示和教育目的。** 虽然 Claude 可能会提供其中某些功能，但您从 Claude 获得的实现和行为可能与这些 Skills 中展示的内容不同。这些 Skills 旨在说明模式和可能性。在将这些 Skills 用于关键任务之前，请务必在您自己的环境中进行全面测试。

# Skill 集

- [./skills](./skills)：创意与设计、开发与技术、企业与沟通以及文档 Skills 的示例
- [./spec](./spec)：Agent Skills 规范
- [./template](./template)：Skill 模板

# 在 Claude Code、Claude.ai 和 API 中试用

## Claude Code

您可以通过在 Claude Code 中运行以下命令将此仓库注册为 Claude Code 插件市场：

```
/plugin marketplace add anthropics/skills
```

然后，要安装特定的技能集：
1. 选择 `Browse and install plugins`
2. 选择 `anthropic-agent-skills`
3. 选择 `document-skills` 或 `example-skills`
4. 选择 `Install now`

或者，也可以直接通过以下方式安装任一 Plugin：

```
/plugin install document-skills@anthropic-agent-skills
/plugin install example-skills@anthropic-agent-skills
```

安装插件后，你只需提及它即可使用该 skill。例如，如果你从 marketplace 安装了 `document-skills` 插件，你可以让 Claude Code 执行类似这样的操作："使用 PDF skill 从 `path/to/some-file.pdf` 中提取表单字段"

## Claude.ai

这些示例 skills 已在 Claude.ai 的付费计划中可用。

要使用本仓库中的任何 skill 或上传自定义 skills，请参阅[在 Claude 中使用 skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b)中的说明。

## Claude API

你可以通过 Claude API 使用 Anthropic 的预构建 skills，并上传自定义 skills。更多详情请参阅 [Skills API 快速入门](https://docs.claude.com/en/api/skills-guide#creating-a-skill)。

# 创建 Basic Skill

Skills 的创建非常简单——只需一个包含 `SKILL.md` 文件的文件夹，文件中包含 YAML frontmatter 和说明。你可以使用本仓库中的 **template-skill** 作为起点：

```markdown
---
name: my-skill-name
description: A clear description of what this skill does and when to use it
---

# My Skill Name

[Add your instructions here that Claude will follow when this skill is active]

## Examples
- Example usage 1
- Example usage 2

## Guidelines
- Guideline 1
- Guideline 2
```

前置元数据仅需两个字段：

- `name` - 技能的唯一标识符（小写，空格用连字符替代）
- `description` - 对技能功能及使用场景的完整描述

下面的 Markdown 内容包含 Claude 将遵循的说明、示例和指南。更多详情，请参阅[如何创建自定义技能](https://support.claude.com/en/articles/12512198-creating-custom-skills)。

# 合作伙伴技能

技能是教授 Claude 如何更好地使用特定软件的绝佳方式。当我们看到合作伙伴们创建的优秀技能示例时，会在此处重点介绍其中的一部分：

- **Notion** - [Notion Skills for Claude](https://www.notion.so/notiondevs/Notion-Skills-for-Claude-28da4445d27180c7af1df7d8615723d0)