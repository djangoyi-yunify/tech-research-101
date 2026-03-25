---
描述：通过提出最多5个高度针对性的澄清问题来识别当前功能规格中的未明确之处，并将答案编码回规格中。
交接:
  - 标签: 构建技术计划
    代理: speckit.plan
    提示: 为此规格创建计划。我正在构建...
脚本:
   sh: scripts/bash/check-prerequisites.sh --json --paths-only
   ps: scripts/powershell/check-prerequisites.ps1 -Json -PathsOnly
---

## 用户输入

```text
$ARGUMENTS
```

# No Input Provided

I notice you've shared the clarification workflow specification, but there's no actual user input to process.

To execute this clarification workflow, I need:

1. **The `{SCRIPT}` path** - The script to run from repo root (e.g., `./speckit.sh` or similar)
2. **Feature specification context** - The current spec file content, OR the output from running the script in JSON mode

Could you please provide:

- The script to execute (or confirm the script name/command)
- The feature spec file path or its contents
- Any additional context for prioritization (if applicable)

Once I have these inputs, I can run the clarification workflow as specified.