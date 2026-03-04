# [PROJECT_NAME] 宪章
<!-- 示例：Spec Constitution, TaskFlow Constitution 等 -->

## 核心原则

### [PRINCIPLE_1_NAME]
<!-- 示例：I. Library-First -->
[PRINCIPLE_1_DESCRIPTION]
<!-- 示例：每个功能始于独立 library；Library 必须自包含、可独立测试、有文档；需明确目的——不存在仅因组织架构而设立的 library -->

### [PRINCIPLE_2_NAME]
<!-- 示例：II. CLI Interface -->
[PRINCIPLE_2_DESCRIPTION]
<!-- 示例：每个 library 通过 CLI 暴露功能；文本输入/输出协议：stdin/args → stdout, errors → stderr；支持 JSON + 人类可读格式 -->

### [PRINCIPLE_3_NAME]
<!-- 示例：III. Test-First (NON-NEGOTIABLE) -->
[PRINCIPLE_3_DESCRIPTION]
<!-- 示例：TDD 强制执行：编写测试 → 用户批准 → 测试失败 → 然后实现；严格遵循 Red-Green-Refactor 循环 -->

### [PRINCIPLE_4_NAME]
<!-- 示例：IV. Integration Testing -->
[PRINCIPLE_4_DESCRIPTION]
<!-- 示例：需集成测试的关注领域：新 library 契约测试、契约变更、服务间通信、共享 schema -->

### [PRINCIPLE_5_NAME]
<!-- 示例：V. Observability, VI. Versioning & Breaking Changes, VII. Simplicity -->
[PRINCIPLE_5_DESCRIPTION]
<!-- 示例：文本 I/O 确保可调试性；需结构化日志；或：MAJOR.MINOR.BUILD 格式；或：保持简单，YAGNI 原则 -->

## [SECTION_2_NAME]
<!-- 示例：附加约束、安全要求、性能标准等 -->

[SECTION_2_CONTENT]
<!-- 示例：技术栈要求、合规标准、部署策略等 -->

## [SECTION_3_NAME]
<!-- 示例：开发 Workflow、Review 流程、质量门禁等 -->

[SECTION_3_CONTENT]
<!-- 示例：Code review 要求、测试门禁、部署审批流程等 -->

## 治理
<!-- 示例：宪章效力高于所有其他实践；修正案需文档、批准及迁移计划 -->

[GOVERNANCE_RULES]
<!-- 示例：所有 PR/review 必须验证合规性；复杂性必须有正当理由；使用 [GUIDANCE_FILE] 获取运行时开发指导 -->

**版本**：[CONSTITUTION_VERSION] | **批准日期**：[RATIFICATION_DATE] | **最后修订**：[LAST_AMENDED_DATE]
<!-- 示例：版本：2.1.1 | 批准日期：2025-06-13 | 最后修订：2025-07-16 -->