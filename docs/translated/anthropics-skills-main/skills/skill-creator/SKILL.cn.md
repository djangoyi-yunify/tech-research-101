---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及测量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析对技能性能进行基准测试，或优化技能的描述以提高触发准确性时使用此技能。
---

# 技能创建器

一个用于创建新技能并对其进行迭代改进的技能。

从高层次来看，创建技能的过程如下：

- 决定你希望技能做什么，以及它应该如何大致实现
- 撰写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在运行进行期间，如果还没有定量评估脚本，则起草一些定量评估脚本（如果已经有一些，你可以直接使用或在你觉得需要更改的地方进行修改）。然后向用户解释它们（或者如果已经存在，向用户解释已存在的评估脚本）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，让他们查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈（以及定量基准测试中出现的任何明显缺陷）重写技能
- 重复此过程直到你满意为止
- 扩展测试集并在更大规模上再试一次

你使用此技能时的任务是找出用户处于流程的哪个阶段，然后帮助他们完成这些阶段。例如，用户可能说"我想为 X 创建一个技能"。你可以帮助缩小他们的含义范围，撰写初稿，编写测试用例，确定他们想要如何评估，运行所有提示词，并重复。

另一方面，如果他们已经有一个技能初稿。在这种情况下，你可以直接进入评估/迭代循环部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行大量评估，只需要和我一起感觉一下"，你也可以这样做。

然后，在技能完成后（但同样顺序是灵活的），你还可以运行技能描述改进器，我们有一个单独的脚本来优化技能的触发。

明白了吗？明白了。

## 与用户沟通

技能创建器可能会被各种熟悉程度的用户使用。如果你没有听说（你怎么能听说呢，这是最近才开始的现象），现在有一种趋势：Claude 的力量激励着水管工打开终端，父母和祖父母开始搜索"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索，以理解如何措辞你的沟通！默认情况下，给你一些概念：

- "evaluation"和"benchmark"是边界情况，但可以接受
- 对于"JSON"和"assertion"，你希望在用户明显知道这些术语的含义之前不要使用它们，或者先解释它们

如果你有疑问，可以简要解释术语，如果你不确定用户是否能理解，可以随意用简短的定义来澄清。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这种情况，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的纠正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户措辞/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否正常工作？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但由用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在你完成这部分之前，不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用，通过子代理并行研究，否则内联进行。准备好背景知识以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发、做什么。这是主要的触发机制——包括技能做什么以及具体的使用上下文。所有"何时使用"的信息都放在这里，而不是放在正文部分。注意：目前 Claude 有"触发不足"的倾向——在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"激进"一些。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写成"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标或想要显示任何类型的公司数据时，请务必使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)

### 技能写作指南

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

1. **元数据**（名称 + 描述）- 始终在上下文中显示（约 100 词）
2. **SKILL.md 正文** - 技能触发时在上下文中显示（理想情况下 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可在未加载情况下执行）

以上字数为近似值，如需可适当增加。

**关键模式：**

- 将 SKILL.md 控制在 500 行以内；如接近此限制，需增加额外的层级结构，并清晰指示使用该技能的模型接下来应如何继续操作。
- 在 SKILL.md 中清晰引用文件，并提供阅读时机指导。
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

这不用说，但 skill 必须不包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果描述了 skill 的内容，其意图不应让用户感到意外。不要配合创建误导性 skill 或旨在促进未授权访问、数据泄露或其他恶意活动的请求。不过，像"扮演 XYZ"这样的内容是可以的。

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

**示例模式** - 包含示例很有用。你可以像这样格式化它们（但如果示例中包含 "Input" 和 "Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

试着向模型解释为什么某些东西很重要，而不是使用那些过于刻板的"MUST"指令。换位思考一下，尽量让技能具有通用性，而不是局限于特定例子。先写一份草稿，然后用新的眼光审视并改进它。

### 测试用例

写完技能草案后，设计2-3个真实的测试提示——也就是用户实际会说的话。与用户分享这些用例：「不必使用完全相同的措辞」"这是我想尝试的几个测试用例。看起来合适吗，还是你想再加一些？"

然后运行这些测试。

将测试用例保存到 `evals/evals.json`。先不要写断言——只保存提示词。稍后在你运行测试的过程中再起草断言。

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

请参阅 `references/schemas.md` 获取完整模式（包括你稍后需要添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的完整序列——请不要中途停止。不要使用 `/skill-test` 或任何其他测试技能。

将结果放在 `<skill-name>-workspace/` 中，作为技能目录的同级目录。在工作空间内，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在此之下，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要一次性创建所有目录——随着进行逐步创建即可。

### 步骤 1：在同一回合中启动所有运行（带技能的基准版本）

对于每个测试用例，在同一回合中生成两个子代理——一个带技能，一个不带。这很重要：不要先启动带技能的运行，然后才回来处理基准版本。一次全部启动，这样它们几乎同时完成。

**带技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基线运行**（相同的 prompt，但基线取决于上下文）：

- **创建新技能**：完全不使用技能。使用相同的 prompt，不指定技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：使用旧版本。编辑前，先对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让基线 subagent 指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言暂时可以为空）。为每个评估使用描述性名称 —— 不要只写 "eval-0"。目录名称也使用这个名称。如果本次迭代使用了新的或修改过的评估 prompt，需要为每个新的评估目录创建这些文件 —— 不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行时起草断言

不要只是等待运行完成——你可以利用这段时间做一些有意义的事情。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们应该在基准测试查看器中清晰可读，这样人们在浏览结果时能立即理解每个断言检查的是什么。主观技能（写作风格、设计质量）最好用定性方式评估——不要把需要人工判断的东西强制加上断言。

一旦起草完成，请更新 `eval_metadata.json` 文件和 `evals/evals.json` 中的断言。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准测试。

### 步骤 3：运行完成后，捕获计时数据

当每个子代理任务完成时，你会收到一条包含 `total_tokens` 和 `duration_ms` 的通知。请立即将这些数据保存到运行目录中的 `timing.json` 文件中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它们通过任务通知传递，不会其他地方持久化。在收到通知时立即处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **为每次运行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录的 `grading.json`。grading.json 的 expectations 数组必须使用字段 `text`、`passed` 和 `evidence`（而不是 `name`/`met`/`details` 或其他变体）—— 查看器依赖这些确切的字段名。对于可以编程检查的断言，编写并运行脚本而不是凭肉眼检查——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解 viewer 期望的确切 schema。将每个 with_skill 版本放在其 baseline 版本之前。

3. **进行分析师审查** — 读取 benchmark 数据，揭示聚合统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论技能如何都总是通过的断言（非区分性）、高方差的评估（可能不稳定）以及 time/token 权衡。

4. **启动 viewer**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代2+，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下一次迭代可以读取。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告诉用户** 类似这样的话："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个查看每个测试用例并留下反馈，'Benchmark' 显示定量对比。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（迭代2+）：折叠部分，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠部分，显示断言的通过/失败情况
- **Feedback**：一个文本框，输入时自动保存
- **Previous Feedback**（迭代2+）：用户上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置通过率、耗时和 token 使用量，包含每个评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键完成。完成后，点击"Submit All Reviews"会将所有反馈保存到 `feedback.json`。

### 步骤5：读取反馈

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

空的反馈意味着用户认为没有问题。将你的改进集中在用户有具体抱怨的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 提升技能

这是循环的核心。你已经运行了测试用例，用户也已经审查了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中总结规律。** 这里的大局是我们正在尝试创建可以被使用数百万次（也许是 literal，甚至更多，谁知道呢）的技能，跨越许多不同的提示。你和用户之所以一遍又一遍地迭代少数几个例子，是因为这样可以帮助加快速度。用户对这些例子了如指掌，能够快速评估新的输出。但如果你和用户共同开发的技能只对这些例子有效，那它就毫无用处。与其做一些繁琐的过度拟合修改，或者设置过于严格的 MUST 约束，如果是一些顽固的问题，你可以尝试分支出去，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你就会找到很棒的解决方案。

2. **保持提示简洁。** 删除那些没有发挥作用的东西。一定要阅读转录稿，而不仅仅是最终输出——如果看起来技能正在让模型浪费大量时间做无效的事情，你可以尝试删除让模型做这些事情的技能部分，然后看看会发生什么。

3. **解释为什么。** 努力解释你要求模型做每一件事的**原因**。今天的 LLM 很聪明。它们有很好的心智理论，如果给它们一个好的框架，就能超越死板的指令，真正让事情发生。即使用户的反馈简短或带有挫折感，也要努力真正理解任务，理解用户为什么写他们写的东西，以及他们实际上写了什么，然后将这种理解传递到指令中。如果你发现自己在写全大写的 ALWAYS 或 NEVER，或者使用超级僵化的结构，这是一个黄色警告信号——如果可能的话，重新措辞并解释推理，这样模型就能理解你要求它做的事情为什么重要。这是一个更人性化、更有力、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录稿，注意子代理是否都独立编写了类似的辅助脚本，或者对某件事情采取了相同的多步骤方法。如果所有 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明技能应该捆绑该脚本。写一次，放到 `scripts/` 中，然后告诉技能使用它。这可以节省未来每次调用的重复开发工作。

这个任务相当重要（我们正在尝试在这里创造数十亿美元的年经济价值！），你的思考时间不是瓶颈；慢慢来，仔细思考。我建议先写一份修订草案，然后再重新审视它并做出改进。真正努力进入用户的头脑，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建一个新技能，基线始终是 `without_skill`（无技能）——这在迭代之间保持不变。如果你正在改进现有技能，使用你的判断力来确定什么作为基线有意义：用户最初带来的原始版本，还是之前的迭代。
3. 使用 `--previous-workspace` 指向前一个迭代来启动审查器
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲比

对于你想对技能的两个版本进行更严格比较的情况（例如，用户问"新版本真的更好吗？"），有一个盲比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：给一个独立的代理两个输出，不告诉它哪个是哪个，让它判断质量。然后分析赢家为什么赢。

这是可选的，需要子代理，大多数用户不需要。人类审查循环通常就足够了。

---

## 描述优化

SKILL.md 前言中的描述字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，建议优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的情况。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I understand the process you're describing for creating eval sets. However, I need to know which skill we're creating the eval queries for.

Could you please provide:

1. **The skill name** (e.g., "PDF Extraction", "Excel Formulas", etc.)
2. **The skill description** — what does this skill do?

Once you share those, I can:

- Create 8-10 **should-trigger** queries with varied phrasings (formal, casual, implicit references to the skill, edge cases)
- Create 8-10 **should-not-trigger** queries that are tricky near-misses (genuinely ambiguous cases where another tool would be more appropriate)
- Help you review them using the HTML template

What skill are we working with?

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（为当前会话提供支持的模型），以便触发测试与用户的实际体验相匹配。

在运行过程中，定期查看输出以向用户更新当前是第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 预留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会对训练集和测试集重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON（根据测试分数而非训练分数选择，以避免过拟合）。

### 技能触发机制的工作原理

了解触发机制有助于设计更好的评估查询。技能会以其名称和描述出现在 Claude 的 `important thing to know is` 列表中，Claude 会根据该描述决定是否咨询技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务上咨询技能——简单的一步式查询（如"读取此 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以直接使用基本工具处理它们。复杂的、多步骤的或专业的查询在描述匹配时能够可靠地触发技能。

这意味着你的评估查询应该足够实质，以至于 Claude 实际上会从咨询技能中受益。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description` 并更新技能的 SKILL.md 前置元数据。向用户展示修改前后的对比并报告分数。

---

### 打包和演示（仅在 `present_files` 工具可用时）

检查是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，请打包技能并向用户演示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

# 技能开发工作流程指南

打包完成后，引导用户前往生成的 `.skill` 文件路径以便安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，某些机制会有所变化。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。读取技能的 SKILL.md，然后按照其说明自行完成测试提示。逐一执行。这比独立子代理的验证方式略宽松（因为你既编写了技能又在运行它，拥有完整的上下文），但这是一个有用的合理性检查——人工审查步骤可以弥补这一点。跳过基线运行——直接使用技能完成任务即可。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示功能，或你在远程服务器上），则完全跳过浏览器审查器。相反，直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），请将其保存到文件系统并告知用户所在位置，以便他们下载检查。inline 方式征求反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖于没有子代理就没有意义的基线比较。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能、重新运行测试用例、征求反馈——只是中间没有浏览器审查器。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此步骤。

**盲对比**：需要子代理。跳过此步骤。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称**。注意技能的目录名称和 `name` 前端字段——保持不变。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置**。已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里进行编辑，然后从副本进行打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——由于权限限制，直接写入可能会失败。

---

## Cowork 专用说明

如果在 Cowork 中，主要需要了解以下几点：

- 你有子代理，因此主要工作流程（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，因此在生成评估查看器时，使用 `--static <output_path>` 编写独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以在浏览器中打开 HTML。
- 无论出于什么原因，Cowork 配置似乎不鼓励 Claude 在运行测试后生成评估查看器，所以再次重申：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器供人工查看示例，然后再自行评估输入并尝试进行修正。使用 `generate_review.py`（不要自己编写定制的 HTML 代码）。抱歉，但我需要大写强调：在自行评估输入之前，先生成评估查看器。你希望尽快让人工看到这些内容！
- 反馈工作方式不同：由于没有运行的服务器，查看器的"提交所有审查"按钮将下载 `feedback.json` 作为文件。你可以从那里读取它（可能需要先请求访问权限）。
- 打包功能可以正常工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意其状态良好后再进行。
- **更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` —— 如何针对输出评估断言
- `agents/comparator.md` —— 如何在两个输出之间进行盲 A/B 对比
- `agents/analyzer.md` —— 如何分析一个版本为何优于另一个版本

references/ 目录包含其他文档：

- `references/schemas.md` —— evals.json、grading.json 等的 JSON 结构

---

为强调起见，这里再次重复核心循环：

- 明确技能的功能
- 起草或编辑技能
- 使用可访问技能的 claude 运行测试提示
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你和你都满意
- 打包最终技能并将其返回给用户

如果你有 TodoList，请添加步骤以确保不会忘记。如果在 Cowork 中，请特别在 TodoList 中添加"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"，以确保它发生。

祝你好运！