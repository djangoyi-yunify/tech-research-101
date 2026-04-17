---
name: skill-creator
description: Create new skills, modify and improve existing skills, and measure skill performance. Use when users want to create a skill from scratch, edit, or optimize an existing skill, run evals to test a skill, benchmark skill performance with variance analysis, or optimize a skill's description for better triggering accuracy.
---

# Skill Creator

用于创建新技能、迭代改进现有技能以及衡量技能效果的技能。

从高层次来看，创建技能的过程如下：

- 确定你希望该技能实现什么功能，以及大致如何实现
- 编写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户从定性和定量两个角度评估结果
  - 在运行进行期间，在后台起草一些定量评估（如果没有现成的）；如果已有一些，你可以直接使用或根据需要修改。然后向用户解释这些评估（如果是已有的，向用户解释现有的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，同时让他们查看定量指标
- 根据用户对结果的评估反馈（以及定量基准测试中出现的明显缺陷）重写技能
- 重复以上步骤直到满意
- 扩展测试集并尝试更大规模的测试

你使用此技能时的任务是弄清楚用户处于这个过程中的哪个阶段，然后帮助他们推进这些阶段。例如，用户可能说"我想制作一个用于 X 的技能"。你可以帮助他们明确需求、撰写草稿、编写测试用例、确定评估方式、运行所有提示词，并循环迭代。

另一方面，如果他们已经有了一份技能草稿，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估，就这样聊聊吧"，你也可以照做。

然后，在技能完成后（顺序同样可以灵活调整），你还可以运行技能描述优化器（我们有专门的处理脚本）来优化技能的触发机制。

清楚了吗？好的。

## 与用户沟通

Skill Creator 可能被各种熟悉程度不同的用户使用，他们对编程术语的了解差异很大。如果你还没听说（你怎么可能听说呢，这是最近才兴起的趋势），现在有一种趋势：Claude 的强大能力激发了水管工打开终端、父母和祖父母去搜索"如何安装 npm"。另一方面，大多数用户可能对电脑比较熟悉。

所以请注意语境线索来理解如何措辞！以默认情况为例，给你一些参考：

- "evaluation"和"benchmark"处于边界，但可以使用
- 对于"JSON"和"assertion"，你需要看到用户确实了解这些术语的明显线索，才能不加解释地使用它们

如果不确定，可以简要解释术语；如果不确定用户是否能理解，可以加一个简短的定义来澄清。

---

## 创建技能

### 明确意图

首先理解用户的意图。当前对话中可能已经包含用户想要捕捉的工作流程（例如，他们说"把这个变成一个技能"）。如果是的话，先从对话历史中提取答案——使用的工具、步骤顺序、用户的纠正、观察到的输入/输出格式。用户可能需要补充空白，并在继续下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 什么时候应该触发这个技能？（用户的哪些表述/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？对于输出可以客观验证的技能（文件转换、数据提取、代码生成、固定工作流程步骤），测试用例很有帮助。对于输出较为主观的技能（写作风格、艺术），通常不需要测试用例。根据技能类型建议合适的默认值，但由用户决定。

### 访谈与研究

主动询问边界情况、输入/输出格式、示例文件、成功标准和依赖项。等到这部分敲定后再编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有的话通过子代理并行研究，否则直接内联研究。做好准备带上下文，以减少用户的负担。

### 编写 SKILL.md

基于用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发，做什么。这是主要的触发机制——包括技能的功能和具体的使用场景。所有"何时使用"的信息都放在这里，不放在正文里。注意：目前 Claude 有一种"触发不足"的倾向——即在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"强硬"一些。例如，不要写成"How to build a simple fast dashboard to display internal Anthropic data."，你可以写成"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能编写指南

#### 技能结构

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description required)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/    - Executable code for deterministic/repetitive tasks
    ├── references/ - Docs loaded into context as needed
    └── assets/     - Files used in output (templates, icons, fonts)
```

#### 渐进式披露

技能采用三级加载系统：

1. **元数据**（名称 + 描述）- 始终在上下文中显示（约 100 词）
2. **SKILL.md 正文** - 技能触发时在上下文中显示（理想情况下 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

以上字数仅为近似值，如需要可适当增加。

**关键模式：**
- 保持 SKILL.md 在 500 行以内；如果接近此限制，应添加额外的层级结构，并提供清晰的指引，说明使用该技能的模型接下来应前往何处继续跟进。
- 在 SKILL.md 中清晰引用文件，并说明何时应读取它们
- 对于大型参考文件（>300 行），应包含目录

**领域组织**：当技能支持多个领域/框架时，按变体进行组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅读取相关的引用文件。

#### 避免意外原则

这不用说，但技能不得包含恶意软件、利用代码或任何可能危害系统安全的内容。技能的描述内容不应在意图上让用户感到意外。不要配合创建误导性技能或旨在促进未授权访问、数据窃取或其他恶意活动的技能请求。不过，像“扮演 XYZ 角色”这样的内容是可以的。

#### 编写模式

在指令中推荐使用祈使句。

**定义输出格式** - 可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**Examples 模式** - 包含示例会很有帮助。你可以这样格式化它们（但如果示例中有 "Input" 和 "Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为何重要，而非使用沉重的、刻板的 MUST（必须）。运用心智理论，努力让技能具有通用性，而不是局限于特定案例。先写一份草稿，然后用新鲜的眼光审视并改进。

### 测试用例

编写完技能草案后，设计 2-3 个现实的测试提示——真实用户可能会说的话类型。将其分享给用户：[不必使用完全相同的措辞] "这是我想尝试的几个测试用例。您觉得合适吗，或者想要添加更多？" 然后运行它们。

将测试用例保存到 `evals/evals.json`。暂不编写断言——只有提示。你会在下一步运行期间起草断言。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's task prompt",
      "expected_output": "Description of expected result",
      "files": []
    }
  ]
}
```

有关完整 schema（包括你稍后要添加的 `assertions` 字段），请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续的过程——不要中途停止。不要使用 `/skill-test` 或任何其他测试 skill。

将结果放在 `<skill-name>-workspace/` 目录中，作为 skill 目录的同级目录。在 workspace 内，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在每个迭代目录下，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——按需创建即可。

### 步骤 1：同一轮中启动所有运行（带 skill 和 baseline）

对于每个测试用例，在同一轮中启动两个子 agent——一个带有 skill，一个不带。这很重要：不要先启动带 skill 的运行，然后再回来运行 baseline。要一次性启动所有任务，这样它们能几乎同时完成。

**带 skill 的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基线运行**（相同提示词，但基线取决于上下文）：

- **创建新技能**：完全不使用技能。相同提示词，无技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：旧版本。编辑之前，对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将基线子代理指向快照。保存到 `old_skill/outputs/`。

为每个测试用例编写一个 `eval_metadata.json`（断言可以暂时留空）。为每个评估取一个描述性名称——基于其测试内容，而不仅仅是 "eval-0"。目录也使用此名称。如果此迭代使用新的或修改过的评估提示词，为每个新的评估目录创建这些文件——不要假设它们会从前面的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第 2 步：趁运行进行中，起草断言

不要只是等待运行完成——你可以利用这段时间做一些有效的工作。为每个测试用例起草定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请审查并解释它们检查的内容。

好的断言应该是客观可验证的，并且有描述性的名称——它们应该在基准查看器中清晰可读，这样查看结果的人能立即理解每个断言检查的内容。主观技能（如写作风格、设计质量）更适合定性评估——不要将断言强行应用于需要人工判断的内容。

起草完成后，更新 `eval_metadata.json` 文件和 `evals/evals.json` 中的断言。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### 第 3 步：运行完成后，捕获计时数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录下的 `timing.json` 文件中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——数据通过任务通知传递，不会被持久化到其他地方。在每个通知到达时立即处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行评分** — 生成一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录下的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以编程检查的断言，编写并运行脚本而不是人工目测——脚本更快、更可靠，并且可以在迭代中复用。

2. **聚合到基准测试** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参见 `references/schemas.md` 了解 viewer 期望的确切 schema。

将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **进行分析师审查** — 阅读 benchmark 数据，挖掘汇总统计可能隐藏的模式。参见 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论是否使用 skill 都总是通过的断言（无区分度）、高方差的评估（可能不稳定），以及 time/token 权衡。

4. **启动 viewer**，同时加载定性输出和定量数据。

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第2次及之后的迭代，还需要传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**Cowork / 无显示环境：** 如果 `webbrowser.open()` 不可用或环境没有显示设备，使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会以 `feedback.json` 文件形式下载。下载后，将 `feedback.json` 复制到工作目录中，以便下一次迭代使用。

注意：请使用 generate_review.py 来创建查看器，无需编写自定义 HTML。

5. **告知用户** 类似这样的话："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给定任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（第2次及之后的迭代）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：一个文本框，输入时自动保存
- **Previous Feedback**（第2次及之后的迭代）：上一次迭代时的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的通过率、耗时和 token 使用量，以及每次评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键完成。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤5：读取反馈

当用户告诉你完成后，读取 `feedback.json`：

```json
{
  "reviews": [
    {"run_id": "eval-0-with_skill", "feedback": "the chart is missing axis labels", "timestamp": "..."},
    {"run_id": "eval-1-with_skill", "feedback": "", "timestamp": "..."},
    {"run_id": "eval-2-with_skill", "feedback": "perfect, love this", "timestamp": "..."}
  ],
  "status": "complete"
}
```

空反馈意味着用户认为没问题。把改进重点放在用户有具体抱怨的测试用例上。完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 skill

这是循环的核心环节。你已经运行了测试用例，用户也审查了结果，现在需要根据他们的反馈来优化 skill。

### 如何思考改进

1. **从反馈中进行泛化。** 这里的大局是：我们正在尝试创建可以被使用数百万次（也许字面上就是如此，甚至更多）的 skill，应用于各种不同的 prompt。你和用户之所以反复迭代少数几个示例，是因为这样速度更快。用户对这些示例了如指掌，可以快速评估新的输出。但如果你和用户共同开发的 skill 只适用于这些示例，那它就毫无用处。与其做出一些容易过拟合的小改动，或者设置过于严格的 MUST 约束，不如尝试拓展思路，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，说不定就能找到绝佳的解决方案。

2. **保持 prompt 精简。** 删除那些没有发挥作用的内容。一定要阅读转录文本，而不仅仅是最终输出——如果看起来 skill 让模型浪费大量时间做毫无成效的事情，你可以尝试删除导致这种行为的 skill 部分，然后观察结果。

3. **解释为什么。** 努力解释你要求模型执行每个操作背后的**原因**。现在的 LLM 非常聪明，它们有很好的心智理论，给定一个好的框架后，可以超越机械指令，真正推动事情进展。即使用户的反馈很简短或带有挫败感，也要努力真正理解任务，理解用户为什么这样写，理解他们实际写了什么，然后将这些理解传达给指令。如果你发现自己用全大写写 ALWAYS 或 NEVER，或者使用过于僵化的结构，这就是一个黄色警示信号——如果可能的话，重新措辞并解释推理过程，让模型理解你请求的事项为何重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意是否有子代理独立编写了相似的辅助脚本或采用相同的多步骤方法来处理某些事情。如果 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明 skill 应该打包这个脚本。编写一次，放入 `scripts/`，然后告诉 skill 使用它。这样可以节省每次调用时重新发明轮子的成本。

这个任务非常重要（我们正在尝试每年创造数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，真正深入思考。建议先写一份修订草案，然后再重新审视并改进。真正尽最大努力进入用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进 skill 后：

1. 将改进应用到 skill
2. 将所有测试用例重新运行到新的 `iteration-<N+1>/` 目录，包括基线运行。如果你创建的是新的 skill，基线始终是 `without_skill`（无 skill）——这在迭代过程中保持不变。如果你是在改进现有 skill，根据你的判断决定什么是合理的基线：用户最初使用的原始版本，还是上一个迭代版本。
3. 使用 `--previous-workspace` 指向上一迭代目录来启动 reviewer
4. 等待用户审查并告知完成
5. 阅读新的反馈，再次改进，重复

持续进行直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级功能：盲测比较

当你想要对两个版本的 skill 进行更严格的比较时（例如用户问"新版本真的更好吗？"），可以使用盲测比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的 agent，但不告诉它哪个是哪个，让它评判质量。然后分析获胜者为何获胜。

这是可选功能，需要子代理，大多数用户不需要。人类审查循环通常已经足够。

---

## 描述优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。创建或改进 skill 后，主动提供优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合 should-trigger 和 should-not-trigger。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I understand the workflow you're describing for creating evaluation sets. However, I notice you've provided the methodology (Steps 1-3) but not the key details I need to execute this:

1. **What skill is this for?** - I need the skill name and description to create appropriate queries
2. **The HTML template** - I can't read `assets/eval_review.html` directly
3. **Additional context** - What comes after Step 3?

Could you provide:
- The skill name and description?
- Any specific edge cases or contexts you want the eval queries to cover?
- The contents of `assets/eval_review.html` if you can share it?

Alternatively, if you've already been working on this skill and there's prior context, let me know and I can continue from there.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的 model ID（驱动当前会话的模型），使触发测试与用户实际体验相符。

运行时，定期 tail 输出，向用户更新当前迭代次数和分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每次查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会重新评估每个新描述在训练集和测试集上的表现，迭代最多 5 次。完成后，会在浏览器中打开 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### Skill 触发机制的工作原理

理解触发机制有助于设计更好的评估查询。Skill 会以名称 + 描述的形式出现在 Claude 的 `available_skills` 列表中，Claude 根据描述决定是否使用某个 skill。需要知道的重要一点是，Claude 只会在处理不了的任务上咨询 skill——简单、单步的查询（如"读取这个 PDF"）可能不会触发 skill，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理它们。复杂的、多步骤的或专门的查询，当描述匹配时能够可靠地触发 skill。

这意味着你的评估查询应该足够实质，让 Claude 真正受益于咨询 skill。像"读取文件 X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发 skill。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，并更新 skill 的 SKILL.md frontmatter。向用户展示前后对比并报告分数。

---

### 打包和呈现（仅当 `present_files` 工具可用时）

检查是否可以使用 `present_files` 工具。如果不能，跳过此步骤。如果可以，打包 skill 并向用户呈现 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，将用户引导到生成的 `.skill` 文件路径，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（草拟 → 测试 → 审核 → 改进 → 重复），但由于 Claude.ai 没有 subagent，某些机制会有所不同。以下是需要调整的内容：

**运行测试用例**：没有 subagent 意味着无法并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后按照其说明自行完成测试提示。一次做一个。这不如独立 subagent 那样严谨（因为你既编写了 skill 也在运行它，所以拥有完整的上下文），但这是一个有用的完整性检查——人工审核步骤可以弥补这一点。跳过基线运行——直接使用 skill 完成请求的任务。

**审核结果**：如果你无法打开浏览器（例如 Claude.ai 的 VM 没有显示器，或者你在远程服务器上），完全跳过浏览器审核器。直接在对话中呈现结果。对于每个测试用例，展示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告诉用户位置，以便他们下载检查。 Inline 方式请求反馈："看起来怎么样？有什么想改的吗？"

**基准测试**：跳过定量基准测试——它依赖于基线比较，在没有 subagent 的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill、重新运行测试用例、请求反馈——只是中间没有浏览器审核器。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果你在 Claude.ai 上，跳过此部分。

**盲测比较**：需要 subagent。跳过此部分。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名称和 `name` frontmatter 字段——保持不变。例如，如果安装的 skill 是 `research-helper`，输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑之前复制到可写位置。** 已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 专用说明

在 Cowork 中，你需要知道的主要内容：

- 你有 subagent，所以主要工作流程（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，所以在生成 eval viewer 时，使用 `--static <output_path>` 来编写独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在自己的浏览器中打开 HTML。
- 出于某种原因，Cowork 配置似乎不鼓励 Claude 在运行测试后生成 eval viewer，所以再次重申：无论你在 Cowork 还是 Claude Code 中，运行测试后都应始终生成 eval viewer 供人工查看示例，然后再自己修订 skill 并尝试进行更正，使用 `generate_review.py`（不要自己编写花哨的 html 代码）。提前致歉，但我还是要大写：**在评估输入之前生成 EVAL VIEWER**！你想让它们尽快出现在用户面前！
- 反馈工作方式不同：由于没有运行中的服务器，viewer 的"Submit All Reviews"按钮将下载 `feedback.json` 作为文件。你可以从那里读取它（可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 `claude -p` 通过 subprocess，而不是浏览器，但请等到你完全完成 skill 并且用户同意它状态良好后再保存它。
- **更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。请遵循上面 Claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专门 subagent 的说明。在需要生成相关 subagent 时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何对两个输出进行盲测 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本胜出

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

再次强调核心循环：

- 了解 skill 的主题
- 起草或编辑 skill
- 运行可以访问 skill 的 claude 处理测试提示
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审核它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终的 skill 并返回给用户。

如果你的任务清单中有这样的功能，请添加步骤以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审核测试用例"添加到你的 TodoList 中以确保它发生。

祝你好运！