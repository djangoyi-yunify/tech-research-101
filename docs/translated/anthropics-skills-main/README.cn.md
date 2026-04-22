> **注意：** 此仓库包含 Anthropic 为 Claude 开发的 skills 实现。如需了解 Agent Skills 标准，请访问 [agentskills.io](http://agentskills.io)。

# Skills
Skills 是由指令、脚本和资源组成的文件夹，Claude 会动态加载它们以提升在特定任务上的表现。Skills 教会 Claude 如何以可重复的方式完成特定任务，无论是按照公司品牌规范创建文档、使用组织特定的工作流分析数据，还是自动化个人任务。

更多信息，请查看：
- [什么是 skills？](https://support.claude.com/en/articles/12512176-what-are-skills)
- [在 Claude 中使用 skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [如何创建自定义 skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [用 Agent Skills 为真实世界的 agents 赋能](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

# 关于此仓库

此仓库包含展示 Claude skills 系统可能性的 skills。这些 skills 涵盖创意应用（艺术、音乐、设计）、技术任务（测试 Web 应用、MCP server 生成）以及企业工作流（沟通、品牌等）。

每个 skill 都独立存在于自己的文件夹中，包含一个 `SKILL.md` 文件，其中包含 Claude 使用的指令和元数据。浏览这些 skills 以获取灵感，或了解不同的模式和 approaches。

此仓库中的许多 skills 都是开源的（Apache 2.0）。我们还包含了为 [Claude 的文档功能](https://www.anthropic.com/news/create-files)提供支持的文档创建和编辑 skills，存放在 [`skills/docx`](./skills/docx)、[`skills/pdf`](./skills/pdf)、[`skills/pptx`](./skills/pptx) 和 [`skills/xlsx`](./skills/xlsx) 子文件夹中。这些是 source-available（非开源），但我们希望与开发者分享，作为在生产 AI 应用中实际使用的更复杂 skills 的参考。

## 免责声明

**这些 skills 仅供演示和教育目的。** 虽然 Claude 的部分功能可能与此类似，但你从 Claude 获得的实现和行为可能与这些 skills 中展示的内容不同。这些 skills 旨在说明模式和可能性。在依赖它们完成关键任务之前，请务必在自己的环境中彻底测试。

# Skill Sets
- [./skills](./skills)：创意与设计、开发与技术、企业与沟通以及文档 Skills 的示例
- [./spec](./spec)：Agent Skills 规范
- [./template](./template)：Skill 模板

# 在 Claude Code、Claude.ai 和 API 中试用

## Claude Code
你可以通过在 Claude Code 中运行以下命令将此仓库注册为 Claude Code Plugin marketplace：

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

安装插件后，只需提及技能即可使用。例如，如果你从 marketplace 安装了 `document-skills` 插件，你可以让 Claude Code 执行类似这样的操作："使用 PDF 技能从 `path/to/some-file.pdf` 中提取表单字段"

## Claude.ai

这些示例技能已在 Claude.ai 的付费计划中全部可用。

要使用此仓库中的任何技能或上传自定义技能，请按照[在 Claude 中使用技能](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b)中的说明操作。

## Claude API

你可以通过 Claude API 使用 Anthropic 的预构建技能，以及上传自定义技能。更多详情请参阅 [Skills API 快速入门](https://docs.claude.com/en/api/skills-guide#creating-a-skill)。

# 创建基础技能

技能的创建非常简单——只需一个包含 YAML frontmatter 和指令的 `SKILL.md` 文件的文件夹。你可以使用此仓库中的 **template-skill** 作为起点：

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

frontmatter 只需要两个字段：

- `name` - 技能的唯一标识符（小写，空格用连字符）
- `description` - 对技能功能和使用场景的完整描述

下面的 markdown 内容包含 Claude 将遵循的指令、示例和指南。更多详情，请参阅[如何创建自定义技能](https://support.claude.com/en/articles/12512198-creating-custom-skills)。

# 合作伙伴技能

技能是教 Claude 如何更好地使用特定软件的好方法。当我们发现来自合作伙伴的优秀技能示例时，可能会在此处重点推荐一些：

- **Notion** - [Claude 的 Notion 技能](https://www.notion.so/notiondevs/Notion-Skills-for-Claude-28da4445d27180c7af1df7d8615723d0)