> **注意：** 此仓库包含 Anthropic 为 Claude 开发的技能实现。如需了解 Agent Skills 标准，请访问 [agentskills.io](http://agentskills.io)。

# 技能

技能是指包含指令、脚本和资源的文件夹，Claude 可以动态加载这些内容以提升在特定任务上的表现。技能教会 Claude 如何以可重复的方式完成具体任务，无论是按照公司品牌指南创建文档、使用组织特定的工作流程分析数据，还是实现个人任务自动化。

如需更多信息，请查看：

- [什么是技能？](https://support.claude.com/en/articles/12512176-what-are-skills)
- [在 Claude 中使用技能](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [如何创建自定义技能](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [使用 Agent Skills 为现实世界的智能体赋能](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

# 关于此仓库

此仓库展示了 Claude 技能系统的各种可能性。这些技能涵盖创意应用（艺术、音乐、设计）、技术任务（测试 Web 应用、MCP 服务器生成）以及企业工作流（通信、品牌等）。

每个技能都独立存放在各自的文件夹中，包含 `SKILL.md` 文件，其中包含 Claude 使用的指令和元数据。您可以浏览这些技能以获取灵感，了解不同的模式和实现方法。

此仓库中的许多技能都是开源的（Apache 2.0）。我们还包含了支撑 [Claude 文档功能](https://www.anthropic.com/news/create-files)的文档创建和编辑技能，存放在 [`skills/docx`](./skills/docx)、[`skills/pdf`](./skills/pdf)、[`skills/pptx`](./skills/pptx) 和 [`skills/xlsx`](./skills/xlsx) 子文件夹中。这些是源代码可用而非开源，但我们希望与开发者分享这些复杂技能的参考实现，因为它们在生产级 AI 应用中已被实际使用。

## 免责声明

**这些技能仅供演示和教育目的。** 虽然 Claude 可能具备其中部分功能，但您从 Claude 获得的实现和行为可能与这些技能中展示的内容不同。这些技能旨在说明模式和可能性。在依赖这些技能完成关键任务之前，请务必在您自己的环境中进行充分测试。

# 技能集

- [./skills](./skills)：创意与设计、开发与技术、企业与通信以及文档技能的示例
- [./spec](./spec)：Agent Skills 规范
- [./template](./template)：技能模板

# 在 Claude Code、Claude.ai 和 API 中试用

## Claude Code

您可以通过在 Claude Code 中运行以下命令将此仓库注册为 Claude Code 插件市场：

```
/plugin marketplace add anthropics/skills
```

然后，安装特定的技能集：

1. 选择 `Browse and install plugins`
2. 选择 `anthropic-agent-skills`
3. 选择 `document-skills` 或 `example-skills`
4. 选择 `Install now`

或者，直接安装任一 Plugin：

```
/plugin install document-skills@anthropic-agent-skills
/plugin install example-skills@anthropic-agent-skills
```

安装插件后，你只需提及它就可以使用该技能。例如，如果你从市场安装 `document-skills` 插件，你可以让 Claude Code 执行类似以下操作：

“使用 PDF 技能从 `path/to/some-file.pdf` 中提取表单字段”

## Claude.ai

这些示例技能在 Claude.ai 的付费计划中都已可用。

要使用本仓库中的任何技能或上传自定义技能，请遵循 [在 Claude 中使用技能](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b) 中的说明。

## Claude API

你可以通过 Claude API 使用 Anthropic 预置的技能，并上传自定义技能。详见 [技能 API 快速入门](https://docs.claude.com/en/api/skills-guide#creating-a-skill)。

# 创建基础技能

技能的创建非常简单——只需一个文件夹，包含一个带有 YAML frontmatter 和说明的 `SKILL.md` 文件。你可以使用本仓库中的 **template-skill** 作为起点：

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

# 合作伙伴技能

技能是教会 Claude 如何更好地使用特定软件的好方法。当我们看到合作伙伴提供的优秀示例技能时，可能会在这里重点介绍其中一些：

- **Notion** - [Notion Skills for Claude](https://www.notion.so/notiondevs/Notion-Skills-for-Claude-28da4445d27180c7af1df7d8615723d0)