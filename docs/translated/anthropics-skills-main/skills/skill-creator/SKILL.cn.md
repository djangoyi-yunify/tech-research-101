---
name: skill-creator
description: 创建新技能、修改和完善现有技能，并测量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估测试技能性能，或优化技能描述以提高触发准确性时使用此技能。
---

# 技能创建器

用于创建新技能并迭代改进它们的技能。

从高层次来看，创建技能的过程如下：

- 决定你想要技能做什么以及大致如何实现
- 编写技能草稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在运行进行的同时，如果没有定量评估指标，可以起草一些定量评估（如果已有，可以直接使用或根据需要修改）。然后向用户解释这些评估（或者如果已经存在，解释已有的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（也包括从定量基准测试中发现的明显缺陷）
- 重复直到满意
- 扩展测试集并尝试更大规模的测试

使用此技能时，你的工作是弄清楚用户处于哪个阶段，然后帮助他们推进这些阶段。例如，也许用户会说"我想为 X 创建一个技能"。你可以帮助他们明确需求、编写草稿、编写测试用例、确定评估方式、运行所有提示词，然后重复这个过程。

另一方面，也许他们已经有了一份技能草稿。在这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活，如果用户说"我不需要运行一堆评估，一起随便看看就好"，你也可以那样做。

然后在技能完成后（但同样，顺序是灵活的），你还可以运行技能描述优化器，我们为此准备了专门的脚本，来优化技能的触发机制。

明白了吗？明白了。

## 与用户沟通

技能创建器可能被对编码术语熟悉程度差异很大的用户使用。如果你没听说过（你怎么可能听说过呢，这是最近才开始的趋势），现在有一种趋势是 Claude 的强大能力正在激励水管工打开终端、父母和祖父母去谷歌搜索"如何安装 npm"。另一方面，大多数用户可能对电脑相当熟悉。

因此请注意上下文线索，了解如何措辞！对于默认情况，给你一些参考：

- "evaluation"和"benchmark"属于边界情况，但可以接受
- 对于"JSON"和"assertion"，你需要看到用户明确表示他们知道这些东西是什么，才可以在不解释的情况下使用

如果有疑问，可以简要解释术语；如果不确定用户是否能理解，可以附上简短定义。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前对话中可能已经包含了用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，先从对话历史中提取答案——使用的工具、步骤顺序、用户的更正、观察到的输入/输出格式。用户可能需要填补空白，并在继续下一步之前确认。

1. 这个技能应该让 Claude 能够做什么？
2. 什么时候应该触发这个技能？（什么用户用语/上下文）
3. 期望的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？对于输出可以客观验证的技能（文件转换、数据提取、代码生成、固定工作流程步骤），测试用例很有帮助。对于输出主观的技能（写作风格、艺术），通常不需要。根据技能类型建议适当的默认值，但让用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。等这部分完善后再编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有子代理可用则通过子代理并行研究，否则内联研究。准备好上下文以减少用户负担。

### 编写 SKILL.md

基于用户访谈，填写以下组件：

- **name**: 技能标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包括技能的功能以及具体的使用场景。所有"何时使用"的信息都放在这里，不放在正文里。注意：目前 Claude 有"触发不足"的倾向——即在有用的时候不使用技能。为了解决这个问题，请把技能描述写得稍微"积极主动"一些。例如，不要写"How to build a simple fast dashboard to display internal Anthropic data."，你可以写"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
- **技能的其他部分 :)**

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

#### 渐进式披露（Progressive Disclosure）

技能使用三级加载系统：

1. **元数据**（名称 + 描述）- 始终在上下文中（~100 词）
2. **SKILL.md 正文** - 在技能触发时加载到上下文中（理想情况下 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

这些字数统计是近似值，如有需要可以超出。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如果接近此限制，请添加额外的层级结构，并提供清晰的指引，说明使用该技能的模型接下来应该去哪里继续跟进。
- 从 SKILL.md 中清晰地引用文件，并提供关于何时读取它们的指导
- 对于大型参考文件（>300 行），请包含目录

**领域组织**：当一个技能支持多个领域/框架时，按变体组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅读取相关的参考文件。

#### 避免意外原则

这点不言自明，技能（skills）不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果技能有描述，其内容不应在意图上让用户感到意外。不要配合创建具有误导性的技能，或设计用于未经授权访问、数据窃取或其他恶意活动的技能。不过，类似"扮演 XYZ"的角色扮演类技能是可以的。

#### 编写模式

在指令中优先使用祈使句。

**定义输出格式** —— 可以这样实现：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有用。你可以这样格式化它们（但如果示例中包含"Input"和"Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为何重要，而不是使用生硬老套的 MUST 指令。运用心智理论，努力让技能具有通用性，而不是过于局限于特定示例。先写一份初稿，然后用新的眼光审视并改进它。

### 测试用例

撰写完技能初稿后，设计 2-3 个真实的测试提示——即真实用户可能会说的话。将其分享给用户：[不必使用完全相同的措辞] "这是我想尝试的几个测试用例。你觉得这些合适吗，或者你想添加更多？"然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时不要编写断言——只保存提示。你会在下一步中，在运行进行时起草断言。

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

请参阅 `references/schemas.md` 获取完整 schema（包括你稍后会添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续流程——不要中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放在 skill 目录的同级目录 `<skill-name>-workspace/` 中。在 workspace 内，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），每个迭代内部，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要一开始就创建所有目录——边走边创建。

### 步骤 1：在同一轮中启动所有运行（使用技能和不使用技能）

对于每个测试用例，在同一轮中启动两个 subagent——一个使用技能，一个不使用。这很重要：不要先启动使用技能的运行，然后再回过头来启动 baseline。同时启动所有内容，以便它们几乎同时完成。

**使用技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同提示词，但 baseline 取决于上下文）：

- **创建新 skill**：完全不使用 skill。使用相同提示词，不指定 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。在编辑之前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline subagent 指向这个快照版本。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言部分暂时留空即可）。为每个 eval 取一个描述性的名称，基于其测试内容——不要简单地命名为 "eval-0"。目录名称也使用这个名称。如果本次迭代使用了新的或修改过的 eval 提示词，需要在每个新 eval 目录下创建这些文件——不要假设它们会从上一次迭代继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### Step 2: While runs are in progress, draft assertions

不要只是等待运行完成——你可以有效地利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言是客观可验证的，并且具有描述性的名称——它们在基准查看器中应该清晰易读，这样人们在查看结果时能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）最好通过定性评估——不要将断言强加于需要人类判断的事物。

一旦起草完成，用断言更新 `eval_metadata.json` 文件和 `evals/evals.json`。还要向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### Step 3: As runs complete, capture timing data

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知传递，不会被持久化存储到其他地方。在通知到达时立即处理，不要尝试批量处理。

### 步骤 4：评分、汇总并启动查看器

所有运行完成后：

1. **对每次运行进行评分** — 启动一个 grader 子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录中的 `grading.json`。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以通过程序检查的断言，应编写并运行脚本而不是肉眼检查——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **汇总到基准测试** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，带有 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参阅 `references/schemas.md` 以了解 viewer 期望的确切 schema。

请将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **进行分析师审查** — 阅读 benchmark 数据，找出 aggregate 统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论 skill 如何都总是通过的断言（非区分性）、高方差的评估（可能不稳定）以及 time/token 的权衡。

4. **启动 viewer**，同时加载定性输出和定量数据。

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 轮及之后的迭代，还需要传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**Cowork / 无头环境：** 如果 `webbrowser.open()` 不可用，或者环境中没有显示器，可以使用 `--static <output_path>` 来生成一个独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈内容会下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作目录中，供下一轮迭代使用。

注意：请使用 generate_review.py 来创建查看器，无需编写自定义 HTML。

5. **告知用户**类似以下内容："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个查看测试用例并留下反馈，'Benchmark' 显示定量对比。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（第 2 轮及之后）：折叠区域，显示上一轮迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：文本框，随用户输入自动保存
- **Previous Feedback**（第 2 轮及之后）：上一轮迭代的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的通过率、耗时和 token 使用量，包含每个评估的详细分解和分析师观察。

导航通过上一页/下一页按钮或方向键完成。完成后，用户点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

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

空的反馈意味着用户认为没有问题。将你的改进重点放在用户有具体抱怨的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 提升技能

这是循环的核心。你已经运行了测试用例，用户也审查了结果，现在需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中提炼通用规律。** 这里的大局是我们试图创建能够被使用百万次（也许字面意义上，甚至更多）的技能，应用于各种不同的 prompt。你和用户在这里反复迭代只有几个例子，因为这样速度更快。用户对这些例子了如指掌，评估新输出对他们来说很快。但如果你和用户共同开发的技能只能用于这些例子，那它就毫无用处。与其进行繁琐的过拟合修改，或者过度限制性的 MUST 要求，如果存在一些顽固问题，你可以尝试拓展思路，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会找到很棒的解决方案。

2. **保持 prompt 精简。** 删除那些没有发挥作用的内容。务必阅读转录文本，而不仅仅是最终输出——如果看起来技能让模型浪费大量时间做低效的事情，你可以尝试删除导致模型这样做的技能部分，然后观察结果。

3. **解释为什么。** 尽力解释你要求模型做的每件事背后的**原因**。当今的 LLM 很聪明。它们具有良好的心智理论，给定一个好的框架，就能超越呆板的指令，真正发挥作用。即使用户的反馈简短或带有情绪，也要努力理解任务，理解用户为什么这样写，理解他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己用全大写的 ALWAYS 或 NEVER，或者使用过于僵化的结构，那是黄旗信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求的事情为什么重要。这是一个更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意是否所有子代理都独立编写了类似的辅助脚本或采用了相同的多步骤方法。如果三个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这是一个强烈的信号，表明技能应该捆绑这个脚本。只写一次，放到 `scripts/` 目录，然后告诉技能使用它。这可以节省每次未来调用时重新发明轮子的成本。

这个任务相当重要（我们正试图在这里创造每年数十亿美元的经济价值！）你的思考时间不是瓶颈；花时间好好琢磨。我建议你先写一份修订草案，然后重新审视并改进。真正尽力走进用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进技能之后：

1. 将你的改进应用到技能中
2. 重新运行所有测试用例到新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建一个新技能，基线始终是 `without_skill`（无技能）——这在各次迭代中保持不变。如果你正在改进现有技能，使用你的判断来决定什么作为基线有意义：用户最初带来的原始版本，还是之前的迭代版本。
3. 使用 `--previous-workspace` 指向之前的迭代来启动审查器
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切都看起来很好）
- 你没有取得有意义的进展

---

## 高级：盲测对比

当你想要对技能的两个版本进行更严格的比较时（例如，用户问"新版本真的更好吗？"），有一个盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的代理，不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md 封面中的 description 字段是决定 Claude 是否调用技能的主要机制。在创建或改进技能后，建议优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I'd be happy to help create an eval set for a skill, but I need more information. The instructions reference a specific skill's name and description, but I don't have that context yet.

Could you please provide:

1. **Skill name** - What is the skill called?
2. **Skill description** - What does this skill do? (This is crucial for creating appropriate should-trigger and should-not-trigger queries)

Alternatively, if you already have a skill file or description file in your workspace, let me know where to find it.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用 system prompt 中的 model ID（驱动当前会话的模型），使触发测试与用户实际体验相匹配。

运行时定期 tail 输出，向用户更新当前迭代次数和得分情况。

这会自动处理完整的优化循环。它将评估集划分为 60% 训练集和 40% 留出测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会对每个新描述在训练集和测试集上重新评估，迭代最多 5 次。完成后，它会在浏览器中打开 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### Skill 触发机制的工作原理

理解触发机制有助于设计更好的评估查询。Skill 会以名称 + 描述的形式出现在 Claude 的 `available_skills` 列表中，Claude 根据描述决定是否需要咨询某个 skill。需要注意的是，Claude 只会在处理不了的任务上咨询 skill——简单的一次性查询（如"读取此 PDF"）可能不会触发 skill，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理。复杂的、多步骤的或专业化的查询在描述匹配时会可靠地触发 skill。

这意味着评估查询应该足够实质化，使 Claude 真正受益于咨询 skill。像"读取文件 X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发 skill。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，并更新 skill 的 SKILL.md frontmatter。向用户展示前后的变化并报告分数。

---

### 打包和呈现（仅在 `present_files` 工具可用时）

检查是否可以使用 `present_files` 工具。如果不能，跳过此步骤。如果可以，打包 skill 并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，将用户导向生成的 `.skill` 文件路径，以便他们可以安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有 subagent，部分机制有所不同。以下是需要调整的内容：

**运行测试用例**：没有 subagent 意味着无法并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后按照其说明自行完成测试提示。逐个执行。这不如独立的 subagent 那样严格（你既编写了 skill 也在运行它，所以你有完整的上下文），但这是一个有用的合理性检查——人类审查步骤可以弥补这一点。跳过基线运行——直接使用 skill 完成请求的任务。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），完全跳过浏览器审查者。直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告知他们位置，以便他们可以下载检查。在行内请求反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖于基线比较，而在没有 subagent 的情况下这些比较没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，请求反馈——只是中间没有浏览器审查者。如果有文件系统，你仍然可以在迭代目录中组织结果。

**描述优化**：本节需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，跳过此部分。

**盲测比较**：需要 subagent。跳过。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名称和 `name` frontmatter 字段——不要更改它们。例如，如果已安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 专用说明

如果你在 Cowork 中，需要了解的主要内容是：

- 你有 subagent，所以主要工作流程（并行生成测试用例、运行基线、评分等）都可以工作。（但是，如果遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，所以在生成 eval viewer 时，使用 `--static <output_path>` 编写独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 出于某种原因，Cowork 配置似乎让 Claude 不愿意在运行测试后生成 eval viewer，所以再次重申：无论你在 Cowork 还是 Claude Code 中，运行测试后都应始终生成 eval viewer 供人类查看示例，然后再自己修订 skill 并尝试修正，使用 `generate_review.py`（不要自己编写花哨的 html 代码）。提前道歉，但我还是要用大写：**在评估输入之前先生成 EVAL VIEWER**！你想尽快把它们展示给人类！
- 反馈的工作方式不同：由于没有运行的服务器，viewer 的 "Submit All Reviews" 按钮会将 `feedback.json` 下载为文件。然后你可以从中读取（你可能需要先请求访问权限）。
- 打包功能可以正常工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 制作并获得用户同意后再进行。
- **更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。按照上面 claude.ai 部分中的更新指南操作。

---

## 参考文件

agents/ 目录包含专用 subagent 的说明。在需要生成相应 subagent 时阅读它们。

- `agents/grader.md` —— 如何针对输出进行断言评估
- `agents/comparator.md` —— 如何在两个输出之间进行盲测 A/B 比较
- `agents/analyzer.md` —— 如何分析为什么一个版本击败了另一个版本

references/ 目录有额外的文档：
- `references/schemas.md` —— evals.json、grading.json 等的 JSON 结构

---

再次强调核心循环：

- 确定 skill 的主题
- 起草或编辑 skill
- 运行可访问 skill 的 claude 处理测试提示
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终的 skill 并返回给用户。

请将这些步骤添加到你的 TodoList 中，以确保不会忘记。如果你在 Cowork 中，请特别将“在 TodoList 中添加'创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类审查测试用例'”，以确保它发生。

祝你好运！