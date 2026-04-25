# 实施计划：[功能]

**分支**：`[###-feature-name]` | **日期**：[DATE] | **规格**：[link]
**输入**：`/specs/[###-feature-name]/spec.md` 中的功能规格

**注意**：此模板由 `__SPECKIT_COMMAND_PLAN__` 命令填充。详见 `.specify/templates/plan-template.md` 中的执行工作流程。

## 摘要

[从功能规格中提取：主要需求 + 研究中的技术方案]

## 技术上下文

<!--
  需要操作：替换本节内容为项目的技术细节。
  这里的结构仅作为指导，以引导迭代过程。
-->

**语言/版本**：[例如，Python 3.11、Swift 5.9、Rust 1.75 或需要澄清]
**主要依赖**：[例如，FastAPI、UIKit、LLVM 或需要澄清]
**存储**：[如果适用，例如 PostgreSQL、CoreData、文件或 不适用]
**测试**：[例如，pytest、XCTest、cargo test 或需要澄清]
**目标平台**：[例如，Linux 服务器、iOS 15+、WASM 或需要澄清]
**项目类型**：[例如，库/CLI/Web 服务/移动应用/编译器/桌面应用 或需要澄清]
**性能目标**：[领域特定，例如 1000 req/s、10k lines/sec、60 fps 或需要澄清]
**约束条件**：[领域特定，例如 <200ms p95、<100MB 内存、离线可用 或需要澄清]
**规模/范围**：[领域特定，例如 10k 用户、1M LOC、50 个界面 或需要澄清]

## 宪法检查

*闸门：必须在阶段 0 研究之前通过。阶段 1 设计后重新检查。*

[根据宪法文件确定闸门]

## 项目结构

### 文档（本功能）

```text
specs/[###-feature]/
├── plan.md              # This file (__SPECKIT_COMMAND_PLAN__ command output)
├── research.md          # Phase 0 output (__SPECKIT_COMMAND_PLAN__ command)
├── data-model.md        # Phase 1 output (__SPECKIT_COMMAND_PLAN__ command)
├── quickstart.md        # Phase 1 output (__SPECKIT_COMMAND_PLAN__ command)
├── contracts/           # Phase 1 output (__SPECKIT_COMMAND_PLAN__ command)
└── tasks.md             # Phase 2 output (__SPECKIT_COMMAND_TASKS__ command - NOT created by __SPECKIT_COMMAND_PLAN__)
```

### 源代码（仓库根目录）

<!--
  行动要求：用此功能的具体布局替换下方的占位符树状结构。删除未使用的选项，并用真实路径（如 apps/admin、packages/something）扩展选中的结构。交付的计划不得包含选项标签。 -->

```text
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**结构决策**：[记录所选结构并引用上文捕获的真实目录]

## 复杂度跟踪

> **仅在 Constitution Check 有违规且需要正当理由时填写**

| 违规 | 为何需要 | 更简单替代方案被驳回原因 |
|------|----------|--------------------------|
| [例如，第4个项目] | [当前需求] | [为何3个项目不足] |
| [例如，仓储模式] | [具体问题] | [为何直接数据库访问不足] |