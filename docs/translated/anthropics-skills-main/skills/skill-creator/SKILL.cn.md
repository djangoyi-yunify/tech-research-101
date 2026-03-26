---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及衡量技能绩效。当用户想要从零开始创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析对技能性能进行基准测试，或优化技能的描述以提高触发准确性时使用此技能。
---

# 技能创建器

用于创建新技能并对其进行迭代改进的技能。

从高层次来看，创建技能的过程如下：

- 决定你想要技能做什么，以及它应该如何大致完成
- 撰写技能的初稿
- 创建几个测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行进行时，如果没有定量评估，则起草一些（如果已经有一些，你可以直接使用或修改，如果你觉得需要改变它们的话）。然后向用户解释它们（或者如果已经存在，解释已经存在的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供他们查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（以及如果定量基准测试中出现的明显缺陷）
- 重复直到你满意为止
- 扩展测试集，然后更大规模地尝试

你使用这个技能时的任务是弄清楚用户在这个过程的哪个阶段，然后帮助他们推进这些阶段。例如，可能他们会说"我想为 X 创建一个技能"。你可以帮助缩小他们的含义，撰写初稿，编写测试用例，找出他们想要如何评估，运行所有提示词，并重复。

另一方面，也许他们已经有一个技能初稿了。在这种情况下，你可以直接进入评估/迭代循环部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估，只管和我一起看看"，你也可以这样做。

然后，在技能完成后（但同样，顺序是灵活的），你还可以运行技能描述改进器，我们有一个单独的脚本来优化技能的触发。

明白了吗？明白了。

## 与用户沟通

技能创建器可能会被各种熟悉程度的用户使用，这些用户对编程术语的熟悉程度范围很广。如果你没有听说（你怎么能呢，这是最近才开始流行的），现在有一种趋势，即 Claude 的力量激励着水管工打开终端，父母和祖父母开始搜索"如何安装 npm"。另一方面，大多数用户可能相当精通电脑。

所以请注意上下文线索，以理解如何措辞你的沟通！在默认情况下，只是给你一些概念：

- "evaluation"和"benchmark"是边缘词，但可以接受
- 对于"JSON"和"assertion"，你需要看到用户 serious clues 知道这些东西是什么，然后在不解释的情况下使用它们

如果你有疑问，可以简要解释术语，如果你不确定用户是否能理解，可以随意用简短的定义来澄清术语。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的纠正、观察到的输入/输出格式。用户可能需要填补空白，并应该在进入下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否正常工作？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在你完成这部分之前，先不要写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用，通过子代理并行研究，否则 inline 进行。做好准备，减少用户的负担。

### 撰写 SKILL.md

根据用户访谈，填写以下组件：

- **name**: 技能标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包括技能做什么以及具体的触发上下文。所有"何时使用"的信息都放在这里，而不是在正文部分。注意：目前 Claude 有"触发不足"的倾向——在技能会有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"激进"一点。例如，不要写"How to build a simple fast dashboard to display internal Anthropic data."，你可以写"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能写作指南

#### 技能剖析

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

1. **元数据**（名称 + 描述）- 始终显示在上下文中（约 100 词）
2. **SKILL.md 正文** - 技能触发时显示在上下文中（理想情况下 <500 行）
3. **打包资源** - 按需加载（无限量，脚本可执行而不必加载）

以上字数仅为近似值，如有需要可适当超出。

**关键模式：**
- 将 SKILL.md 保持在 500 行以内；如接近此限制，需增加一层层级结构，并清晰指示使用该技能的模型接下来应如何继续操作。
- 在 SKILL.md 中清晰引用文件，并提供读取时机的指导。
- 对于大型参考文件（>300 行），需包含目录。

**领域组织**：当一个技能支持多个领域/框架时，按变体进行组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 只读取相关的参考文件。

#### 不意外原则

这不用说，但技能绝对不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果对技能进行描述，其内容不应该让用户对其意图感到意外。不要配合创建误导性技能或设计用于未授权访问、数据外泄或其他恶意活动的请求。不过，像「扮演 XYZ」这类内容是可以的。

#### 编写模式

在指令中优先使用祈使句形式。

**定义输出格式** - 你可以这样做：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例格式** - 包含示例很有用。你可以这样格式化它们（但是如果示例中有 "Input" 和 "Output"，你可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释为什么某些内容很重要，而不是使用生硬刻板的 MUST 命令。运用心智理论，努力让技能具有通用性，而不是局限于特定示例。先写一份草稿，然后用全新的视角审视并改进它。

### 测试用例

在撰写技能初稿之后，设计 2-3 个现实的测试提示——即真实用户可能会说的话。与用户分享这些用例：[你可以不使用完全相同的措辞] "我想尝试这几个测试用例。看起来是否合适，还是你想添加更多？" 然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时先不写断言——暂时只保存提示内容。你会在下一步运行过程中起草断言。

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

请参阅 `references/schemas.md` 了解完整 schema（包括 `assertions` 字段，您稍后会添加）。

## 运行和评估测试用例

本节是一个连续的过程——请不要中途停止。不要使用 `/skill-test` 或任何其他测试技能。

将结果放在 `<skill-name>-workspace/` 中，作为 skill 目录的同级目录。在 workspace 中，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代中，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——随着进行逐步创建。

### 第一步：在同一回合中生成所有运行（使用 skill 的和基线的）

对于每个测试用例，在同一回合中生成两个子代理——一个带有 skill，一个不带。这很重要：不要先生成使用 skill 的运行，然后再回来生成基线的。一次全部启动，以便它们同时完成。

**使用 skill 的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基线运行**（相同提示词，但基线取决于上下文）：

- **创建新技能**：没有任何技能。使用相同提示词，不指定技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：旧版本。编辑前，先对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让基线子代理指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言暂时可以留空）。为每个评估取一个描述性名称——不要只是用 "eval-0"。目录名称也使用这个名称。如果本次迭代使用了新的或修改过的评估提示词，要为每个新的评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行中起草断言

不要仅仅等待运行完成——你可以利用这段时间来高效地工作。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们在基准查看器中应该清晰易读，这样人们在浏览结果时能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将需要人工判断的内容强制添加断言。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### 步骤 3：运行完成时，捕获计时数据

当每个子代理任务完成时，你会收到一条包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "task_name": {
    "total_tokens": 12345,
    "duration_ms": 12345
  }
}
```

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它通过任务通知传递，不会保存在其他任何地方。在收到每个通知时立即处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分**——生成一个评分子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用字段 `text`、`passed` 和 `evidence`（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖于这些确切的字段名称。对于可以编程检查的断言，编写并运行脚本而不是肉眼检查——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合成 benchmark**——从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，并带有平均值±标准差和 delta 值。如果手动生成 `benchmark.json`，请参阅 `references/schemas.md` 查看查看器期望的确切 schema。将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **进行分析师审查** — 阅读基准数据并揭示聚合统计可能隐藏的模式。参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解要查找的内容——例如无论技能如何都始终通过的断言（无区分度）、高方差评估（可能不稳定）以及时间/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代 2+，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协同办公/无头环境：** 如果 `webbrowser.open()` 不可用或环境无显示，使用 `--static <output_path>` 来生成独立的 HTML 文件而非启动服务器。当用户点击"Submit All Reviews"时，反馈内容将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，供下一次迭代使用。

注意：请使用 generate_review.py 来创建查看器，无需编写自定义 HTML。

5. **告知用户** 类似这样："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个查看测试用例并留下反馈，'Benchmark' 显示定量对比。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（迭代 2+）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评级）：折叠区域，显示断言的通过/失败情况
- **Feedback**：自动保存的文本框，输入时即时保存
- **Previous Feedback**（迭代 2+）：用户上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：各配置的执行率、耗时和 token 使用量，包含每次评估的细分和分析师观察结果。

通过 prev/next 按钮或方向键导航。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

当用户完成时，读取 `feedback.json`：

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

空的反馈意味着用户认为没问题。将你的改进集中在用户有具体投诉的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是循环的核心。你已经运行了测试用例，用户已经审查了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中归纳总结。** 这里的大局观是，我们正在尝试创建可以用于数百万次（也许是字面意思，甚至是更多，谁知道呢）的技能，跨越许多不同的提示。你和用户之所以在这里反复迭代少数几个示例，是因为这样可以帮助加快速度。用户对这些示例了如指掌，能够快速评估新的输出。但如果你和用户共同开发的技能只适用于这些示例，那它就毫无用处。与其进行繁琐的过拟合修改，或者使用强制性的MUST规则，不如尝试分支出去，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会发现很棒的东西。

2. **保持提示简洁。** 移除那些没有发挥作用的东西。确保阅读了转录稿，而不仅仅是最终输出——如果看起来技能让模型浪费了大量时间做没有成效的事情，你可以尝试删除导致这种情况的部分技能，然后看看会发生什么。

3. **解释原因。** 努力解释你要求模型做每件事情的**原因**。当今的LLM很聪明。它们有很好的心智理论，如果给予良好的引导，就可以超越机械指令，真正发挥作用。即使用户的反馈简短或带有挫败感，也要努力真正理解任务，理解用户为什么写他们写的内容，以及他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己在全大写使用ALWAYS或NEVER，或者使用超级僵化的结构，那就是一个黄色信号——如果可能的话，重新措辞并解释推理，让模型理解你要求的事情为什么重要。这是一个更人性化、更有力、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录稿，注意是否有子代理独立编写了类似的辅助脚本或采取了相同的多步骤方法。如果所有3个测试用例都导致子代理编写了`create_docx.py`或`build_chart.py`，这就是一个强烈的信号，表明技能应该捆绑该脚本。写一次，放到`scripts/`中，然后告诉技能使用它。这可以节省未来每次调用的重复开发时间。

这个任务非常重要（我们正在尝试创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，仔细思考。我建议先写一份修订草案，然后重新审视它并做出改进。真正努力进入用户的头脑，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的`iteration-<N+1>/`目录中，包括基线运行。如果你正在创建一个新技能，基线始终是`without_skill`（无技能）——它在迭代中保持不变。如果你正在改进现有技能，请自行判断什么作为基线有意义：用户最初使用的原始版本，还是之前的迭代。
3. 使用`--previous-workspace`指向前一个迭代来启动审查器
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲测比较

对于你想要更严格地比较技能两个版本的情况（例如，用户问"新版本实际上更好吗？"），有一个盲测比较系统。阅读`agents/comparator.md`和`agents/analyzer.md`了解详情。基本思路是：给一个独立的代理两个输出，不告诉它哪个是哪个，让它判断质量。然后分析为什么胜出的那个赢了。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md前言中的描述字段是决定Claude是否会调用技能的主要机制。在创建或改进技能后，提供优化描述以提高触发准确性的服务。

### 第一步：生成触发评估查询

创建20个评估查询——混合应该触发和不应该触发的类型。保存为JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I understand the workflow you've outlined for creating evaluation sets for skills. However, I notice you haven't specified which skill we're creating the eval set for.

Could you please provide:

1. **The skill name** (e.g., "PDF Parser", "Excel Formatter", etc.)
2. **The skill description** - what the skill does

Once I have this information, I can:

- Review the eval queries you've prepared
- Help identify gaps in coverage (intent variations, edge cases)
- Suggest improvements for the should-not-trigger cases to make them more challenging/nearmiss
- Help format everything for the HTML review template

What's the skill you're working on?

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（当前会话使用的模型），以便触发测试与用户实际体验一致。

运行时，定期尾部查看输出，向用户更新当前是第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会在训练集和测试集上重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### 技能触发机制的工作原理

了解触发机制有助于设计更好的评估查询。技能会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 会根据该描述决定是否咨询技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务上咨询技能——简单的一步式查询（如"读取此 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以使用基本工具直接处理它们。复杂、多步骤或专业的查询在描述匹配时能够可靠地触发技能。

这意味着您的评估查询应该足够实质，以至于 Claude 确实能从咨询技能中受益。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md frontmatter。向用户展示之前和之后的情况，并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查您是否有权访问 `present_files` 工具。如果没有，请跳过此步骤。如果有，请打包技能并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，引导用户获取生成的 `.skill` 文件路径，以便安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，部分机制有所不同。需要调整的部分如下：

**运行测试用例**：没有子代理意味着无法并行执行。读取技能的 SKILL.md，然后按照其说明自行完成测试提示。每次执行一个。这不如独立子代理严格（你编写了技能，也在运行它，所以你拥有完整的上下文），但这是一个有用的健全性检查——人工审查步骤可以弥补这一点。跳过基准测试——直接使用技能完成任务。

**审查结果**：如果你无法打开浏览器（例如，Claude.ai 的虚拟机没有显示器，或你在远程服务器上），完全跳过浏览器审查。直接在对话中呈现结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告知用户位置，以便他们下载和检查。inline 询问反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，询问反馈——只是中间没有浏览器审查。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，跳过此部分。

**盲对比**：需要子代理。跳过。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都可以工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称**。注意技能的目录名称和 `name` 前端字段——保持不变。例如，如果已安装的技能是 `research-helper`，输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置**。已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先暂存到 `/tmp/`**，然后复制到输出目录——由于权限问题，直接写入可能会失败。

---

## Cowork 专用说明

在 Cowork 中，需要了解的主要事项是：

- 你有子代理，所以主要工作流程（并行生成测试用例、运行基准测试、评分等）都能正常运作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，所以在生成评估查看器时，使用 `--static <output_path>` 写入独立的 HTML 文件。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 由于某种原因，Cowork 设置似乎不利于 Claude 在运行测试后生成评估查看器，所以再次重申：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器，让人工在修改技能并尝试纠正之前查看示例，使用 `generate_review.py`（不要自己编写定制的 html 代码）。预先道歉，但我会大写：**在评估输入之前先生成评估查看器**！你希望尽快让人工看到这些内容！
- 反馈工作方式不同：由于没有运行的服务器，查看器的"提交所有审查"按钮会将 `feedback.json` 下载为文件。然后你可以从中读取（可能需要先请求访问权限）。
- 打包可以正常工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过子进程使用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意技能已经完善后再进行。
- **更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专业子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲 A/B 对比
- `agents/analyzer.md` — 如何分析一个版本击败另一个版本的原因

references/ 目录有额外的文档：

- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

为了强调，再重复一次核心循环：

- 确定技能的功能
- 起草或编辑技能
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你和你满意的用户
- 打包最终技能并返回给用户

如果你有 TodoList，请添加步骤以确保你不会忘记。如果你在 Cowork 中，请特别在 TodoList 中写下"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"，以确保它发生。

祝你好运！