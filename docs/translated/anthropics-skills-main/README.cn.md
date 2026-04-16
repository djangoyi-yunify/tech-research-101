> **注意：** 此仓库包含 Anthropic 的 Claude 技能实现。如需了解 Agent Skills 标准，请访问 [agentskills.io](http://agentskills.io)。

# Skills

Skills 是包含指令、脚本和资源的文件夹，Claude 会动态加载它们以提升在专业化任务上的表现。Skills 教会 Claude 如何以可重复的方式完成特定任务，无论是在公司品牌指南下创建文档、使用组织特定的工作流程分析数据，还是自动化个人任务。

更多信息，请参阅：

- [什么是 Skills？](https://support.claude.com/en/articles/12512176-what-are-skills)
- [在 Claude 中使用 Skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [如何创建自定义 Skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [用 Agent Skills 为现实世界的 Agent 赋能](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

# 关于此仓库

此仓库包含展示 Claude 技能系统可能性的 Skills。这些技能涵盖创意应用（艺术、音乐、设计）、技术任务（测试 Web 应用、MCP Server 生成）以及企业工作流程（沟通、品牌等）。

每个 Skill 都自包含在独立的文件夹中，包含一个 `SKILL.md` 文件，其中包含 Claude 使用的指令和元数据。浏览这些 Skills 可以为创建自己的 Skills 获取灵感，或了解不同的模式和方案。

此仓库中的许多 Skills 是开源的（Apache 2.0）。我们还包含了为 [Claude 的文档功能](https://www.anthropic.com/news/create-files) 提供支持的文档创建和编辑 Skills，位于 [`skills/docx`](./skills/docx)、[`skills/pdf`](./skills/pdf)、[`skills/pptx`](./skills/pptx) 和 [`skills/xlsx`](./skills/xlsx) 子文件夹中。这些是源可获取的，而非开源，但我们希望与开发者分享这些作为复杂 Skills 的参考，这些 Skills 在生产级 AI 应用中被积极使用。

## 免责声明

**这些 Skills 仅供演示和教育目的。** 虽然 Claude 中可能提供部分这些功能，但你从 Claude 获得的实现和行为可能与这些 Skills 中展示的内容不同。这些 Skills 旨在说明模式和可能性。在依赖它们完成关键任务之前，务必在自己的环境中彻底测试 Skills。

# Skill Sets

- [./skills](./skills)：创意与设计、开发与技术、企业与沟通以及文档技能的技能示例
- [./spec](./spec)：Agent Skills 规范
- [./template](./template)：Skill 模板

# 在 Claude Code、Claude.ai 和 API 中试用

## Claude Code

你可以通过在 Claude Code 中运行以下命令将此仓库注册为 Claude Code 插件市场：

```
/plugin marketplace add anthropics/skills
```

然后，要安装一组特定的技能：
1. 选择 `Browse and install plugins`
2. 选择 `anthropic-agent-skills`
3. 选择 `document-skills` 或 `example-skills`
4. 选择 `Install now`

或者，也可以直接通过以下方式安装任一 Plugin：

```
/plugin install document-skills@anthropic-agent-skills
/plugin install example-skills@anthropic-agent-skills
```

安装插件后，你只需提及即可使用该 skill。例如，如果你从 marketplace 安装了 `document-skills` 插件，你可以让 Claude Code 执行类似这样的操作："使用 PDF skill 从 `path/to/some-file.pdf` 中提取表单字段"

## Claude.ai

这些示例 skills 在 Claude.ai 的付费计划中均已可用。

要使用此仓库中的任何 skill 或上传自定义 skills，请按照 [在 Claude 中使用 skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b) 中的说明操作。

## Claude API

你可以通过 Claude API 使用 Anthropic 的预构建 skills，并上传自定义 skills。更多详情请参阅 [Skills API 快速入门](https://docs.claude.com/en/api/skills-guide#creating-a-skill)。

# 创建基础 Skill

Skills 创建起来很简单——只需要一个包含 `SKILL.md` 文件的文件夹，文件中包含 YAML frontmatter 和说明。你可以使用此仓库中的 **template-skill** 作为起点：

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

前端配置只需要两个字段：

- `name` - 技能的唯一标识符（小写字母，空格用连字符）
- `description` - 技能功能及使用场景的完整描述

下面的 Markdown 内容包含 Claude 将遵循的指令、示例和指南。更多详细信息，请参阅[如何创建自定义技能](https://support.claude.com/en/articles/12512198-creating-custom-skills)。

# 合作伙伴技能

技能是一种教 Claude 更好地使用特定软件的好方法。当我们看到来自合作伙伴的优秀技能示例时，可能会在这里重点介绍：

- **Notion** - [Notion Skills for Claude](https://www.notion.so/notiondevs/Notion-Skills-for-Claude-28da4445d27180c7af1df7d8615723d0)