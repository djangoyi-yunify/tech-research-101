---
name: skill-creator
description: 创建新技能、修改和改进现有技能以及衡量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析对技能性能进行基准测试，或优化技能的描述以提高触发准确性时使用。
---

# 技能创建器

用于创建新技能并迭代改进它们的技能。

从高层次来看，创建技能的流程如下：

- 决定你想要技能做什么以及大概如何实现
- 编写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行期间，如果没有定量评估，则起草一些定量评估（如果已经存在一些评估，你可以直接使用或修改，如果你认为需要改变某些内容）。然后向用户解释这些评估（或者如果评估已经存在，解释已经存在的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，让他们查看，也让查看定量指标
- 根据用户对结果的评估反馈重写技能（如果定量基准测试中出现任何明显的缺陷，也要据此重写）
- 重复直到满意为止
- 扩展测试集并尝试更大规模的运行

你使用此技能时的任务是弄清楚用户处于流程的哪个阶段，然后帮助他们推进这些阶段。例如，用户可能说"我想要一个用于 X 的技能"。你可以帮助缩小他们的需求范围，编写初稿，编写测试用例，弄清楚他们想要如何评估，运行所有提示词并重复。

另一方面，如果用户已经拥有技能的初稿，你可以直接进入评估/迭代循环。

当然，你应该始终保持灵活性，如果用户说"我不需要运行大量评估，只是陪我聊聊"，你也可以这样做。

然后，在技能完成后（但同样顺序是灵活的），你还可以运行技能描述优化器，我们有一个单独的脚本来优化技能的触发。

清楚了吗？清楚了。

## 与用户沟通

技能创建器可能被各种熟悉程度的用户使用，他们对编程术语的了解范围很广。如果你没有听说（你怎么可能听说呢，这是最近才开始出现的），现在有一种趋势：克劳德的强大能力激励水管工打开终端，父母和祖父母去谷歌"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索，以理解如何措辞你的沟通！在默认情况下，给你一些概念：

- "evaluation"和"benchmark"是边界情况，但可以用
- 对于"JSON"和"assertion"，你需要看到用户确实知道这些东西是什么的明显线索，然后再使用它们而不解释

如果你不确定，可以简短地解释术语，如果你不确定用户是否能理解，可以随意用简短的定义来澄清。

---

## 创建技能

### 捕获意图

首先了解用户的意图。当前对话可能已经包含了用户想要捕获的工作流程（例如他们说"把这个变成一个技能"）。如果是的话，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的修正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前确认。

1. 这个技能应该让克劳德做什么？
2. 这个技能应该在什么时候触发？（什么用户短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但让用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖关系。在完成这部分之前不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用，通过子代理并行研究，否则内联研究。准备好上下文以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发、做什么。这是主要的触发机制——包括技能做什么以及具体的使用上下文。所有"何时使用"信息放在这里，而不是在正文中。注意：目前克劳德有"触发不足"的倾向——在技能会有用的时候不使用它。为了解决这个问题，请把技能描述写得稍微"激进"一点。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标或想要显示任何类型的企业数据时，务必使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能编写指南

#### 技能的解剖结构

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

技能采用三层加载系统：

1. **元数据**（名称 + 描述）- 始终在上下文中呈现（约 100 字）
2. **SKILL.md 正文** - 技能触发时在上下文中呈现（理想情况下 <500 行）
3. **打包的资源** - 按需加载（无限制，脚本可无需加载即可执行）

这些字数为近似值，如需可自行增加。

**关键模式：**

- 将 SKILL.md 保持在 500 行以内；如接近此限制，应添加额外的层级结构，并明确指示使用该技能的模型接下来应前往何处进行后续操作。
- 在 SKILL.md 中清晰引用文件，并提供何时读取的指导
- 对于大型参考文件（>300 行），应包含目录

**领域组织**：当一个技能支持多个领域/框架时，按变体进行组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅读取相关的参考文件。

#### 不予惊喜原则

不用说，技能绝对不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果对技能进行了描述，其内容不应在意图上让用户感到惊讶。不要配合创建误导性技能或设计用于非法访问、数据窃取或其他恶意活动的技能。不过，像“扮演 XYZ”之类的角色扮演是可以的。

#### 编写模式

在指令中优先使用祈使形式。

**定义输出格式** - 可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例会很有用。你可以这样格式化它们（但如果示例中有"输入"和"输出"，你可能需要稍微调整一下格式）。

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释为什么某些事情很重要，而不是用生硬死板的"MUST"指令。运用心理理论（Theory of Mind），尽量使技能具有通用性，而不是局限在特定的例子上。先写一份初稿，然后用全新的视角审视并改进它。

### 测试用例

写完技能初稿后，设计2-3个真实的测试提示——也就是真实用户实际会说的话。与用户分享：[不必使用完全相同的措辞] "我想尝试这几个测试用例。看起来是否合适，或者你想补充更多？"然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时先不写断言——等到运行进行时再起草断言。

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

请参阅 `references/schemas.md` 获取完整模式（包括 `assertions` 字段，您稍后会添加该字段）。

## 运行和评估测试用例

本节是一个连续的流程——请勿中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

请将结果放在 `<skill-name>-workspace/` 中，作为技能目录的同级目录。在工作区中，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在该目录下，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——边执行边创建。

### 步骤 1：在同一回合中启动所有运行（包括带技能的运行和基线运行）

对于每个测试用例，在同一回合中启动两个子代理——一个带有技能，一个不带。这很重要：请勿先启动带技能的运行，稍后再回来启动基线运行。一次全部启动，以便它们几乎同时完成。

**带技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline 运行**（相同 prompt，但 baseline 取决于上下文）：
- **创建新 skill**：完全不使用 skill。使用相同 prompt，不指定 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。编辑前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline subagent 指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言部分暂时留空）。为每个 eval 使用描述性名称——不要只是简单地命名为 "eval-0"。同时使用该名称作为目录名称。如果本次迭代使用了新的或修改过的 eval prompt，需要为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第二步：运行进行时，拟定断言

不要只是等待运行完成——你可以高效地利用这段时间。为每个测试用例拟定定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——在基准测试查看器中阅读时应该清晰明了，让浏览结果的人能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加给需要人工判断的内容。

一旦拟定完断言，就更新 `eval_metadata.json` 文件和 `evals/evals.json`。还要向用户解释他们在查看器中会看到什么——包括定性输出和定量基准测试。

### 第三步：运行完成时，捕获计时数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将这些数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它通过任务通知传递，不会持久化到其他位置。请在到达时处理每个通知，而不是尝试批量处理。

### 第 4 步：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）—— 查看器依赖这些确切的字段名称。对于可以编程检查的断言，请编写并运行脚本而不是凭肉眼判断—— 脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准** — 运行 skill-creator 目录中的聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 查看查看器期望的确切模式。
将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **做一次分析师处理** — 阅读基准数据并揭示聚合统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解要关注的内容——例如无论技能如何都总是通过的断言（无区分度）、高方差的评估（可能不稳定）以及时间/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第2次及之后的迭代，还需要传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作环境/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。当用户点击"提交所有评论"时，反馈将以 `feedback.json` 文件的形式下载。下载后，将 `feedback.json` 复制到工作区目录，以便下一次迭代拾取。

注意：请使用 `generate_review.py` 来创建查看器；无需编写自定义 HTML。

5. **告知用户** 类似这样的话："我已经在浏览器中打开了结果。有两个标签页——'Outputs' 可以让你逐个查看测试用例并留下反馈，'Benchmark' 显示定量比较。完成后，回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给出的任务
- **Output**：技能产生的文件，尽可能以内联方式渲染
- **Previous Output**（第2次及之后迭代）：显示上一次迭代输出的折叠部分
- **Formal Grades**（如果运行了评分）：显示断言通过/失败情况的折叠部分
- **Feedback**：输入时自动保存的文本框
- **Previous Feedback**（第2次及之后迭代）：他们上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的执行率、时间和 token 使用量，包含每次评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键完成。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 第5步：读取反馈

当用户告诉你完成后，读取 `feedback.json`：`

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

空的反馈意味着用户认为没问题。将改进重点放在用户有具体抱怨的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是整个流程的核心。你已经运行了测试用例，用户也已经审核了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进方向

1. **从反馈中总结规律。** 我们正在做的事情的宏观目标是创建可以被使用数百万次（也许真的是这样，甚至更多，谁知道呢）的技能，应用于各种不同的提示词。你和用户之所以在少数几个例子上反复迭代，是因为这样可以加快速度。用户对这些例子了如指掌，能够快速评估新的输出。但如果你和用户共同开发的技能只能处理这些例子，那它就毫无用处。与其做一些容易过拟合的琐碎改动，或者设置过于严格的限制性规则，不如尝试采用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你就能找到绝佳的解决方案。

2. **保持提示词精简。** 移除那些没有发挥作用的内容不仅要看最终输出，还要仔细阅读转录文本——如果看起来技能在让模型浪费大量时间做无意义的事情，可以尝试删除导致这种结果的技能部分，然后观察效果。

3. **解释背后的原因。** 尽你所能解释你要求模型做每一件事的**原因**。当今的大语言模型非常聪明。它们有很好的心智理论，如果给予适当的引导，就能超越机械指令，真正发挥作用。即使用户的反馈很简短或带有情绪，也要尝试真正理解任务，理解用户为什么写他们写的东西，理解他们实际写了什么，然后将这种理解融入到指令中。如果你发现自己满篇都在用大写字母写"总是"或"绝不"，或者使用过于僵化的结构，这是一个黄色警告信号——如果可能的话，重新措辞并解释推理，这样模型才能理解你为什么要求它做这件事。这是一个更人性化、更强大、更有效的方法。

4. **寻找测试用例中的重复工作。** 阅读测试运行的转录文本，注意子代理是否独立编写了类似的辅助脚本，或者对某件事采取了相同的多步骤方法。如果三个测试用例都导致子代理写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，说明技能应该打包这个脚本。只写一次，放到 `scripts/` 目录下，然后告诉技能去使用它。这可以让未来每一次调用都无需重复造轮子。

这个任务相当重要（我们正试图在这里创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，仔细思考。我建议先写一稿改进方案，然后用全新的眼光审视它并做出改进。真正设身处地为用户想想，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建新技能，基线始终是 `without_skill`（无技能）——这在所有迭代中保持不变。如果你是在改进现有技能，使用你的判断力来决定什么是合理的基线：用户最初带来的原始版本，还是上一个迭代版本。
3. 使用 `--previous-workspace` 指向上一个迭代来启动审核器
4. 等待用户审核并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复这个过程

继续下去直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲测对比

在某些情况下，如果你想更严格地比较技能的两个版本（例如，用户问"新版本真的更好吗？"），可以使用盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详细信息。基本思路是：把两个输出交给一个独立的代理，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审核循环通常就足够了。

---

## 描述优化

SKILL.md 前言中的描述字段是决定 Claude 是否调用技能的主要机制。在创建或改进技能后，主动提出优化描述以提高触发准确性。

### 第一步：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的情况。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I understand you're providing instructions for creating an evaluation set for a skill, but I notice you've included the workflow instructions without specifying the actual skill I need to work with.

To proceed, I need:

1. **Skill name** — What is the skill called? (e.g., "Excel File Processing", "PDF Text Extraction", etc.)

2. **Skill description** — What does this skill do? What are its capabilities?

3. **Any specific requirements** — Are there particular features, edge cases, or capabilities you'd like me to focus on for the eval queries?

Once you provide these details, I can create realistic **should-trigger** and **should-not-trigger** queries following the guidelines you outlined.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（当前会话使用的模型），以便触发测试与用户的实际体验一致。

在运行过程中，定期查看输出以向用户更新当前是第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 预留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会在训练集和测试集上重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——按测试分数而非训练分数选择，以避免过拟合。

### 技能触发机制的工作原理

了解触发机制有助于设计更好的评估查询。技能会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 会根据该描述决定是否咨询某个技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务上咨询技能——简单的一步式查询（如"读取此 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以用基本工具直接处理它们。复杂、多步骤或专业化的查询在描述匹配时能可靠地触发技能。

这意味着你的评估查询应该足够实质，让 Claude 真正能从咨询技能中受益。像"读取文件 X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description` 并更新技能的 SKILL.md 前端信息。向用户展示修改前后的对比并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，打包技能并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，引导用户到最终的 `.skill` 文件路径，以便他们安装。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，一些机制有所不同。以下是需要调整的地方：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，先阅读 skill 的 SKILL.md，然后按照其说明自己完成测试提示。逐个执行。这不像独立子代理那样严格（你编写了这个 skill 同时也在运行它，所以你有完整的上下文），但这是一个有用的健全性检查——而且人工审查步骤可以弥补这一点。跳过基线运行——直接使用 skill 按要求完成任务。

**审查结果**：如果无法打开浏览器（例如，Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），完全跳过浏览器审查器。相反，直接在对话中呈现结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告诉用户位置，以便他们可以下载和检查。inline 询问反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖基线比较，没有子代理就没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，询问反馈——只是中间没有浏览器审查器。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此部分。

**盲对比**：需要子代理。跳过。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境下都可以工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能是在要求你更新现有 skill，而不是创建新的。在这种情况下：
- **保留原始名称**。注意 skill 的目录名和 `name` 前端字段——保持不变。例如，如果已安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置**。已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——由于权限问题，直接写入可能会失败。

---

## Cowork 特定说明

如果在 Cowork，需要了解的主要事项是：

- 你有子代理，所以主要工作流程（并行生成测试用例、运行基线、评分等）都可以正常运作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，因此在生成评估查看器时，使用 `--static <output_path>` 编写一个独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 由于某种原因，Cowork 设置似乎不鼓励 Claude 在运行测试后生成评估查看器，所以再次重申一下：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器，让人工在你自己修改 skill 并尝试纠正之前查看示例，使用 `generate_review.py`（不要自己写定制的 html 代码）。提前道歉，但我还是要大写强调：在自己评估输入之前，先生成评估查看器。你需要尽快让人工看到它们！
- 反馈工作方式不同：由于没有运行的服务器，查看器的"提交所有审查"按钮将下载 `feedback.json` 作为文件。然后你可以从中读取它（你可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 `claude -p` 通过子进程，而不是浏览器，但请等到你完全完成 skill 并且用户同意它已经完善后再进行。
- **更新现有 skill**：用户可能是在要求你更新现有 skill，而不是创建新的。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` —— 如何针对输出评估断言
- `agents/comparator.md` —— 如何在两个输出之间进行盲 A/B 对比
- `agents/analyzer.md` —— 如何分析为什么一个版本击败了另一个

references/ 目录有额外的文档：
- `references/schemas.md` —— evals.json、grading.json 等的 JSON 结构

---

为了强调，这里再次重复核心循环：

- 确定 skill 的内容
- 起草或编辑 skill
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你 和用户都满意
- 打包最终 skill 并返回给用户

请将步骤添加到你的 TodoList 中，以确保不会忘记。如果在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"放入你的 TodoList 中，以确保它发生。

祝你好运！