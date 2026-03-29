> **注意：** 此仓库包含 Anthropic 为 Claude 开发的技能实现。关于 Agent Skills 标准，请访问 [agentskills.io](http://agentskills.io)。

# 技能

技能是指令、脚本和资源的集合，Claude 可动态加载这些内容以提升在特定任务中的表现。技能教会 Claude 如何以可重复的方式完成具体任务，无论是根据公司品牌指南创建文档、使用组织特定的工作流程分析数据，还是自动化个人任务。

更多信息请查看：
- [什么是技能？](https://support.claude.com/en/articles/12512176-what-are-skills)
- [在 Claude 中使用技能](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [如何创建自定义技能](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [使用 Agent Skills 为现实世界的代理赋能](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

# 关于此仓库

此仓库包含展示 Claude 技能系统可能性的技能示例。这些技能涵盖创意应用（艺术、音乐、设计）到技术任务（测试 Web 应用、MCP 服务器生成）再到企业工作流程（通信、品牌等）。

每个技能都独立存放于各自文件夹中，包含 `SKILL.md` 文件，其中包含 Claude 使用的指令和元数据。您可以浏览这些技能以获取灵感，或了解不同的模式和方案。

此仓库中的许多技能都是开源的（Apache 2.0）。我们还包含了为 [Claude 的文档功能](https://www.anthropic.com/news/create-files) 提供支持的文档创建和编辑技能，位于 [`skills/docx`](./skills/docx)、[`skills/pdf`](./skills/pdf)、[`skills/pptx`](./skills/pptx) 和 [`skills/xlsx`](./skills/xlsx) 子文件夹中。这些是源码可用而非开源的，但我们希望与开发者分享这些作为复杂技能的参考，这些技能在生产级 AI 应用中被积极使用。

## 免责声明

**这些技能仅供演示和教育目的。** 虽然 Claude 可能提供其中部分功能，但您从 Claude 获得的实现和行为可能与这些技能中展示的内容不同。这些技能旨在说明模式和可能性。在依赖这些技能完成关键任务之前，请务必在您自己的环境中进行彻底测试。

# 技能集
- [./skills](./skills)：创意与设计、开发与技术、企业与通信以及文档技能的技能示例
- [./spec](./spec)：Agent Skills 规范
- [./template](./template)：技能模板

# 在 Claude Code、Claude.ai 和 API 中试用

## Claude Code

您可以通过在 Claude Code 中运行以下命令将此仓库注册为 Claude Code 插件市场：

```
/plugin marketplace add anthropics/skills
```

然后，安装特定技能集：

1. 选择 `Browse and install plugins`
2. 选择 `anthropic-agent-skills`
3. 选择 `document-skills` 或 `example-skills`
4. 选择 `Install now`

或者，直接通过以下方式安装任一插件：

```
/plugin install document-skills@anthropic-agent-skills
/plugin install example-skills@anthropic-agent-skills
```

安装插件后，您只需提及该技能即可使用。例如，如果您从市场安装了 `document-skills` 插件，您可以请求 Claude Code 执行类似这样的操作："使用 PDF 技能从 `path/to/some-file.pdf` 中提取表单字段"

## Claude.ai

这些示例技能已在 Claude.ai 的付费计划中提供。

要使用本仓库中的任何技能或上传自定义技能，请按照 [在 Claude 中使用技能](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b) 中的说明进行操作。

## Claude API

您可以使用 Anthropic 的预置技能，并通过 Claude API 上传自定义技能。详见 [Skills API 快速入门](https://docs.claude.com/en/api/skills-guide#creating-a-skill)。

# 创建基础技能

技能的创建非常简单——只需一个包含 YAML frontmatter 和说明的 `SKILL.md` 文件即可。您可以使用本仓库中的 **template-skill** 作为起点：

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

# 前言

frontmatter 只需要两个字段：

- `name` - 你的技能的唯一标识符（小写，空格用连字符）
- `description` - 技能的功能完整描述及使用场景

下方的 Markdown 内容包含 Claude 将遵循的指令、示例和指南。更多详情请参阅[如何创建自定义技能](https://support.claude.com/en/articles/12512198-creating-custom-skills)。

# 合作伙伴技能

技能是帮助 Claude 提升特定软件使用能力的有效方式。当我们发现合作伙伴提供的优秀技能示例时，可能会在此进行展示：

- **Notion** - [适用于 Claude 的 Notion 技能](https://www.notion.so/notiondevs/Notion-Skills-for-Claude-28da4445d27180c7af1df7d8615723d0)