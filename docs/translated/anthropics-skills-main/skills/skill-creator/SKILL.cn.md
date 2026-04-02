---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及测量技能绩效。当用户想要从头开始创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析进行技能性能基准测试，或优化技能描述以提高触发准确性时使用。
---

# Skill Creator

用于创建新技能并对其进行迭代改进的技能。

从高层次来看，创建技能的过程如下：

- 决定你想要技能做什么以及大致应该如何做
- 编写技能的初稿
- 创建几个测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行的同时，如果没有现成的定量评估，则起草一些定量评估（如果有现成的评估，你可以直接使用或修改，如果你觉得需要改变某些内容）。然后向用户解释这些评估（如果已经存在，则解释已有的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，让他们查看，也让用户查看定量指标
- 根据用户对评估结果的反馈（以及定量基准测试中出现的任何明显缺陷）重写技能
- 重复直到你满意为止
- 扩大测试集并尝试更大规模的运行

你使用此技能时的任务是弄清楚用户处于流程的哪个阶段，然后介入并帮助他们完成这些阶段。例如，用户可能说"我想为 X 创建一个技能"。你可以帮助缩小他们的意思范围，编写初稿，编写测试用例，弄清楚他们想要如何评估，运行所有提示词，并重复。

另一方面，如果用户已经有关于技能的初稿。在这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活，如果用户说"我不需要运行一堆评估，只需要和我一起讨论"，你也可以这样做。

然后，在技能完成之后（但同样，顺序是灵活的），你还可以运行技能描述优化器，我们有一个单独的脚本，用于优化技能的触发。

明白了？明白了。

## 与用户沟通

Skill Creator 可能会被各种熟悉程度的用户使用，这些用户对编程术语的了解程度各不相同。如果你没有听说（你怎么能呢，这是最近才开始的现象），现在有一种趋势，Claude 的强大能力激励着水管工打开终端，父母和祖父母开始谷歌"如何安装 npm"。另一方面，大部分用户可能相当精通电脑。

所以请注意上下文线索，以理解如何措辞你的沟通！以默认情况为例，给你一些概念：

- "evaluation" 和 "benchmark" 是边界情况，但可以使用
- 对于 "JSON" 和 "assertion"，在你使用它们而没有解释之前，需要看到用户确实知道这些术语的认真线索

如果你有疑问，可以简要解释术语，如果你不确定用户是否理解，可以随时用简短的定义来澄清术语。

---

## 创建技能

### 捕捉意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这变成一个技能"）。如果是这种情况，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在继续下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但由用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在你完成这部分之前，先不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有的话通过子代理并行研究，否则 inline 进行。准备好背景知识以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发、做什么。这是主要的触发机制——包括技能做什么以及具体的使用上下文。所有"何时使用"的信息放在这里，而不是放在正文部分。注意：目前 Claude 有一种"触发不足"的倾向——在技能有用时不使用它。为了解决这个问题，请把技能描述写得稍微"激进"一些。例如，不要写成"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写成"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标或想要显示任何类型的公司数据时，务必使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：必需工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能编写指南

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

1. **元数据**（名称 + 描述）- 始终在上下文中（~100词）
2. **SKILL.md 正文** - 技能触发时在上下文中（理想<500行）
3. **捆绑资源** - 按需加载（无限制，脚本可执行无需加载）

这些词数为近似值，如有需要可以超出。

**关键模式：**

- SKILL.md 保持在 500 行以内；如接近此限制，添加额外的层级结构，并清楚指引使用该技能的模型接下来应该去哪里继续。
- 在 SKILL.md 中清晰标注文件，并提供何时读取的指导
- 对于大型参考文件（>300行），包含目录

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

#### 不应令人惊讶的原则

这不言而喻，但 skill 绝不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果描述了 skill 的内容，其意图不应让用户感到意外。不要配合创建误导性 skill 或旨在促进未授权访问、数据窃取或其他恶意活动的 skill。不过，“扮演 XYZ”这类的是可以的。

#### 编写模式

指令中建议使用祈使句形式。

**定义输出格式** - 你可以这样做：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例会很有用。你可以这样格式化它们（但如果示例中包含 "Input" 和 "Output"，你可能需要稍微调整一下）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 编写风格

尝试向模型解释事情为什么重要，而不是使用生硬刻板的"MUST"。运用心智理论，努力让技能具有通用性，而不是局限于特定案例。先写一份草稿，然后用全新的眼光审视并改进它。

### 测试用例

在撰写技能草稿后，设计 2-3 个真实的测试提示——即真实用户可能会说的话。与用户分享：[你不必使用完全相同的措辞]"我想尝试这几个测试用例。看起来是否合适，或者你想补充更多？"然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时不要写断言——只保存提示即可。在运行过程中，你将在下一步起草断言。

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

完整的模式定义请参阅 `references/schemas.md`（包括稍后需要添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的操作流程——请勿中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放在技能目录同级的 `<skill-name>-workspace/` 中。在工作空间内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代内，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不需要预先创建所有目录——随着操作进行再创建即可。

### 步骤 1：在同一回合内启动所有运行（使用技能的版本和基准版本）

对于每个测试用例，在同一回合内生成两个子代理——一个配备技能，一个不配备技能。这点很重要：请勿先启动使用技能的运行，然后再回头启动基准版本。一次全部启动，这样它们能几乎同时完成。

**使用技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基准运行**（相同的提示词，但基准取决于上下文）：

- **创建新技能**：完全没有技能。使用相同的提示词，无技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：旧版本。编辑前，先对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将基准子代理指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言暂时可以为空）。给每个评估一个描述性名称——不要只写 "eval-0"。目录名称也使用这个名称。如果这次迭代使用了新的或修改的评估提示词，要为每个新的评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行时起草断言

不要只是等待运行完成——你可以高效地利用这段时间。为每个测试用例起草定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们在基准测试查看器中应该清晰易读，以便有人一眼就能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要把需要人工判断的东西强制加上断言。

一旦起草完断言，请更新 `eval_metadata.json` 文件和 `evals/evals.json`。还要向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### 步骤 3：运行完成时，捕获计时数据

当每个子代理任务完成时，你会收到一条包含 `total_tokens` 和 `duration_ms` 的通知。立即将这些数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它们通过任务通知传递，不会持久化到其他位置。收到每个通知后立即处理，不要尝试批量处理。

### 第 4 步：评分、聚合并启动查看器

所有运行完成后：

1. **为每次运行评分**——生成一个评分子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录下的 `grading.json`。grading.json 的 expectations 数组必须使用字段 `text`、`passed` 和 `evidence`（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名称。对于可以程序化检查的断言，编写并运行脚本而不是人工检查——脚本更快、更可靠，并且可以在多次迭代中复用。

2. **聚合到基准**——从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及均值 ± 标准差和 delta 值。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器期望的精确模式。

将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **进行分析师审查** — 阅读基准数据并揭示聚合统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results" 部分）了解需要关注的内容——例如无论技能如何都会通过的断言（无区分度）、高方差评估（可能不稳定）以及时间/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代 2+，还需传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作环境 / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下次迭代拾取。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告知用户** 类似这样的话："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个查看测试用例并留下反馈，'Benchmark' 显示定量对比。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给定的任务
- **Output**：技能产生的文件，尽可能内联渲染
- **Previous Output**（迭代 2+）：折叠部分，显示上次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠部分，显示断言的通过/失败情况
- **Feedback**：文本框，输入时自动保存
- **Previous Feedback**（迭代 2+）：用户上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的执行率、耗时和 token 使用量，包含每个评估的细分和分析师观察结果。

导航通过 prev/next 按钮或方向键完成。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

当用户告知完成后，读取 `feedback.json`：

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

空反馈表示用户认为没有问题。请专注于改进那些用户有具体投诉的测试用例：

完成之后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是循环的核心。你已经运行了测试用例，用户已经审核了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中总结规律。** 这里的大局是，我们正在尝试创建可以被使用数百万次（可能 literally，甚至更多，谁知道呢）的技能，跨越许多不同的提示。你和用户之所以在少数例子上反复迭代，是因为这样可以更快地推进。用户对这些例子了如指掌，能够快速评估新输出。但如果你和用户共同开发的技能只能用于这些例子，那就没有用了。与其做一些繁琐的过拟合修改，或者过度限制性的 MUSTs，如果存在一些顽固的问题，你可以尝试分支出去，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你就能找到很棒的方案。

2. **保持提示简洁。** 移除那些没有作用的元素。确保阅读转录稿，而不仅仅是最终输出——如果看起来技能正在让模型浪费大量时间做 unproductive 的事情，你可以尝试删除导致这种情况的技能部分，然后看看会发生什么。

3. **解释原因。** 努力解释你要求模型做每件事的 **原因**。当今的 LLMs 很聪明。它们有很好的心智理论，给定一个好的框架，就能超越机械指令，真正让事情发生。即使用户的反馈很简短或很沮丧，也要努力真正理解任务，理解用户为什么写他们写的内容，以及他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己在全大写写 ALWAYS 或 NEVER，或者使用超级 rigid 的结构，这是一个 yellow flag——如果可能的话，重新措辞并解释推理，这样模型就能理解你要求的事情为什么重要。这是一个更人性化、更有力量、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录稿，注意子代理是否都独立编写了类似的辅助脚本，或对某件事情采取了相同的多步骤方法。如果所有 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明技能应该捆绑那个脚本。写一次，放到 `scripts/` 里，然后告诉技能使用它。这可以节省未来每次调用的重复劳动。

这个任务相当重要（我们正在尝试每年创造数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，仔细考虑。我建议写一份修订草案，然后重新审视它并做出改进。真正努力进入用户的头脑，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建新技能，基线总是 `without_skill`（无技能）——这在所有迭代中保持不变。如果你正在改进现有技能，请运用你的判断力，判断什么作为基线有意义：用户带来的原始版本，还是之前的迭代。
3. 使用 `--previous-workspace` 指向前一个迭代来启动审核器
4. 等待用户审核并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲比较

对于你想对技能的两个版本进行更严格比较的情况（例如，用户问"新版本真的更好吗？"），有一个盲比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：把两个输出交给一个独立的代理，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审核循环通常就足够了。

---

## 描述优化

SKILL.md 前言中的描述字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，主动提出优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I don't see a specific skill name or description in your message. This appears to be a process template for creating evaluation sets, but the actual skill information is missing.

Could you please provide:

1. **The skill name** (e.g., "PDF text extraction", "Excel formulas", "Image resizing", etc.)
2. **The skill description** — what the skill does, its capabilities, and when it should be triggered

Once I have this information, I can create the eval set with:

- **8-10 should-trigger queries** — realistic, specific queries that match the skill's purpose, with varied phrasings (formal/casual), implicit skill mentions, edge cases, and some that compete with similar skills
- **8-10 should-not-trigger queries** — near-misses with shared keywords but different actual intent, genuinely tricky cases that test the boundary of when the skill should vs. shouldn't trigger

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（为当前会话提供支持的模型），以便触发测试与用户的实际体验相匹配。

在运行过程中，定期查看输出，向用户更新当前处于第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 预留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它重新评估每个新描述在训练集和测试集上的表现，最多迭代 5 次。完成后，它会在浏览器中打开一份 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### 技能触发的工作原理

了解触发机制有助于设计更好的评估查询。技能会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 会根据该描述决定是否咨询技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务时咨询技能——简单的一次性查询（如"读取此 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理它们。复杂、多步骤或专业化的查询在描述匹配时能可靠地触发技能。

这意味着你的评估查询应该足够实质化，以便 Claude 确实能从咨询技能中受益。简单的查询（如"读取文件 X"）是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md frontmatter。向用户展示修改前后的对比，并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查是否有权访问 `present_files` 工具。如果没有，请跳过此步骤。如果有，打包技能并将 .skill 文件展示给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，引导用户前往生成的 `.skill` 文件路径，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，某些机制有所不同。需要调整的内容如下：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，请先阅读技能的 SKILL.md，然后按照其说明自己完成测试提示。按顺序逐个执行。这不如独立子代理那样严格（因为你既编写了技能又在运行它，所以拥有完整的上下文），但这是一个有用的健全性检查——人工审查步骤可以弥补这一点。跳过基线运行——直接使用技能按照请求完成任务。

**审查结果**：如果你无法打开浏览器（例如，Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），则完全跳过浏览器审查器。相反，直接在对话中呈现结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），请将其保存到文件系统并告知他们位置，以便他们可以下载和检查。inline 寻求反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖没有子代理就无法进行的基线比较。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，寻求反馈——只是中间没有浏览器审查器。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此部分。

**盲比**：需要子代理。跳过此部分。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的地方都可以工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能是在要求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称。** 记录技能的目录名称和 `name` 前端字段——保持不变。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里进行编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——由于权限限制，直接写入可能会失败。

---

## Cowork 专用说明

在 Cowork 中，主要需要了解以下几点：

- 你有子代理，所以主要工作流程（并行生成测试用例、运行基线、评分等）都可以工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也可以。）
- 你没有浏览器或显示器，所以在生成评估查看器时，使用 `--static <output_path>` 编写独立的 HTML 文件，而不是启动服务器。然后提供一个链接，让用户可以在浏览器中打开 HTML。
- 无论如何，Cowork 设置似乎不利于 Claude 在运行测试后生成评估查看器，所以再次重申：无论你是在 Cowork 还是 Claude Code 中，运行测试后都应始终生成评估查看器，让人工在你自己修订技能并尝试纠正之前查看示例，使用 `generate_review.py`（不要自己编写定制的 html 代码）。提前道歉，但我还是要大写强调：在自己评估输入之前先生成评估查看器！你希望让人工尽快看到它们！
- 反馈工作方式不同：由于没有运行的服务器，查看器的"提交所有评论"按钮将下载 `feedback.json` 作为文件。你可以从那里读取它（可能需要先请求访问）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意其状态良好后再进行。
- **更新现有技能**：用户可能是在要求你更新现有技能，而不是创建新技能。请遵循上面 Claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专业子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本击败了另一个版本

references/ 目录包含附加文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

为了强调，这里再次重复核心循环：

- 确定技能是关于什么的
- 起草或编辑技能
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你 和用户都满意
- 打包最终技能并将其返回给用户

如果你有待办事项列表，请添加步骤以确保你不会忘记。如果你是在 Cowork 中，请特别在待办事项列表中放入"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"，以确保它发生。

祝你好运！