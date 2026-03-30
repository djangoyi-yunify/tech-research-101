---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及衡量技能绩效。当用户想要从头开始创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析进行技能性能基准测试，或优化技能的描述以获得更好的触发准确性时使用。
---

# Skill Creator（技能创建器）

用于创建新技能并对其进行迭代改进的技能。

从高层次来看，创建技能的过程如下：

- 决定你想要技能做什么，以及它应该如何大致做到
- 撰写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行进行时，如果没有定量评估，则起草一些定量评估（如果已经有一些定量评估，你可以直接使用或修改，如果你认为有什么需要改变的话）。然后向用户解释它们（或者如果它们已经存在，解释已经存在的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，让他们查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（以及如果定量基准测试中出现任何明显的缺陷）
- 重复直到你满意为止
- 扩大测试集，然后更大规模地再试一次

你使用这个技能时的任务是弄清楚用户处于这个过程的哪个阶段，然后加入并帮助他们推进这些阶段。例如，可能他们会说"我想为 X 创建一个技能"。你可以帮助缩小他们的含义，撰写初稿，编写测试用例，弄清楚他们想要如何评估，运行所有提示词，并重复。

另一方面，也许他们已经有技能的初稿。在这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估 just vibe with me"，你也可以这样做。

然后，在技能完成后（但同样，顺序是灵活的），你也可以运行技能描述改进器，我们有一个单独的脚本，用于优化技能的触发。

好的？好的。

## 与用户沟通

技能创建器可能被各种熟悉程度不同的用户使用。如果你没有听说（你怎么能听说，它最近才开始），现在有一种趋势，即 Claude 的力量激励水管工打开终端，父母和祖父母搜索"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索，了解如何措辞你的沟通！在默认情况下，只是给你一些概念：

- "evaluation"和"benchmark"是边界情况，但可以接受
- 对于"JSON"和"assertion"，你希望看到用户严重暗示他们知道这些是什么，然后才能不加解释地使用它们

如果你有疑问，可以简短地解释术语，如果你不确定用户是否理解，可以随意用简短的定义来澄清术语。

---

## 创建技能

### 捕获意图

首先了解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在你完成这部分之前，不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用，通过子代理并行研究，否则内联研究。做好准备，减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发、做什么。这是主要的触发机制——包括技能做什么以及具体的使用上下文。所有"何时使用"信息都放在这里，而不是在正文部分。注：目前 Claude 有"触发不足"的倾向——在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"激进"一点。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。请确保在用户提到仪表板、数据可视化、内部指标或想要显示任何类型的企业数据时使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能写作指南

#### 技能的解剖学

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

1. **元数据**（名称 + 描述）- 始终加载（~100 字）
2. **SKILL.md 正文** - 技能触发时加载（理想 <500 行）
3. **打包的资源** - 按需加载（无限制，脚本可在未加载时执行）

这些字数为近似值，如需要可以超出。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如接近此限制，请添加额外的层级结构，并清晰指示使用该技能的模型接下来应前往何处继续。
- 在 SKILL.md 中清晰引用文件，并提供何时读取的指导
- 对于大型参考文件（>300 行），包含目录

**领域组织**：当一个技能支持多个领域/框架时，按变体组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 只会读取相关的参考文件。

#### 无惊喜原则

这不言自明，但 skill 不得包含恶意软件、利用代码或任何可能危及系统安全的内容。如果描述了某个 skill，其内容不应在意图上让用户感到意外。不要配合创建误导性 skill 或旨在促进未经授权访问、数据外泄或其他恶意活动的 skill。不过，像“扮演 XYZ”这样的角色扮演是可以的。

#### 写作模式

在指令中优先使用祈使句形式。

**定义输出格式** - 你可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有用。你可以这样格式化它们（但如果示例中有 "Input" 和 "Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

试着向模型解释为什么某些东西很重要，而不是使用老套生硬的"MUST"指令。运用心智理论，尝试让技能具有通用性，而不是局限于特定例子。先写一份草稿，然后用全新的视角审视并改进它。

### 测试用例

写完技能初稿后，创建 2-3 个真实的测试提示——即真实用户可能会说的话。与用户分享：[你可以不使用完全相同的措辞]“我想尝试这几个测试用例。你觉得这些合适吗，或者还想补充一些？”然后运行它们。

将测试用例保存到 `evals/evals.json`。先不要写断言——只保存提示词。你会在下一步运行过程中起草断言。

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

完整的 schema（包括 `assertions` 字段，你稍后会添加）请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续的完整流程——请勿中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放在 `<skill-name>-workspace/` 中，作为技能目录的同级目录。在 workspace 内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代内，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——随着进行逐步创建即可。

### 步骤 1：在同一轮次中启动所有运行（使用技能的运行和基线运行）

对于每个测试用例，在同一轮次中启动两个子代理——一个使用技能，一个不使用。这一点很关键：不要先启动使用技能的运行，然后再回来启动基线运行。要一次性全部启动，这样它们大致会同时完成。

**使用技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基线运行**（相同的提示词，但基线取决于上下文）：

- **创建新技能**：完全没有技能。使用相同的提示词，不使用技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：旧版本。在编辑之前，先对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将基线子代理指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言暂时可以为空）。根据测试内容为每个评估赋予描述性名称——不仅仅是 "eval-0"。也使用此名称作为目录名。如果此迭代使用了新的或修改过的评估提示，请为每个新评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第二步：运行进行时，起草断言

不要只是等待运行完成——你可以利用这段时间高效工作。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们应该在基准测试查看器中清晰可读，以便有人浏览结果时能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要强制为需要人工判断的内容添加断言。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### 第三步：运行完成时，捕获计时数据

当每个子代理任务完成时，你会收到一条包含 `total_tokens` 和 `duration_ms` 的通知。立即将这些数据保存到运行目录的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——数据通过任务通知传递，不会其他地方保存。请在收到每个通知时立即处理，不要尝试批量处理。

### 第 4 步：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分** — 生成一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录的 `grading.json`。grading.json 的 expectations 数组必须使用字段 `text`、`passed` 和 `evidence`（而不是 `name`/`met`/`details` 或其他变体）—— 查看器依赖于这些确切的字段名称。对于可以编程检查的断言，请编写并运行脚本而不是凭肉眼判断 — 脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准中** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器所需的具体模式。将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **进行分析师审查** — 阅读基准数据并揭示聚合统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"分析基准结果"部分）了解需要关注的内容——例如无论技能如何都总是通过的断言（非区分性）、高方差评估（可能是 flaky 的），以及 time/token 权衡。

4. **启动查看器**，同时提供定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代 2+，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协同办公/无头环境：** 如果 `webbrowser.open()` 不可用或环境无显示，请使用 `--static <output_path>` 来写入独立的 HTML 文件而非启动服务器。当用户点击"Submit All Reviews"时，反馈将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下次迭代读取。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告知用户** 类似如下："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个点击查看每个测试用例并留下反馈，'Benchmark' 显示定量对比。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能产生的文件，尽可能内联渲染
- **Previous Output**（迭代 2+）：显示上次迭代输出的折叠部分
- **Formal Grades**（如果运行了分级）：显示断言通过/失败的折叠部分
- **Feedback**：输入时自动保存的文本框
- **Previous Feedback**（迭代 2+）：用户上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的正确率、耗时和 token 使用量，以及每个评估的细分和分析师观察。

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

空的反馈意味着用户认为没有问题。将你的改进集中在用户有具体抱怨的测试用例上：

完成 viewer server 后将其关闭：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 提升技能

这是循环的核心。你已经运行了测试用例，用户也已经查看了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中归纳。** 这里总体来说，我们正在尝试创建可以跨多种不同提示词使用数百万次（也许真的是这样，甚至更多）的技能。你和用户之所以在少数示例上反复迭代，是因为这样能更快地推进工作。用户对这些示例了如指掌，能够快速评估新输出。但如果你和用户共同开发的技能只能处理这些示例，那就毫无用处。与其进行繁琐的过拟合修改，或者设置强制性限制性的 MUST 条款，不如尝试分支出去，使用不同的隐喻，或者推荐不同的工作模式。尝试的成本相对较低，也许你就能找到很棒的方案。

2. **保持提示词简洁。** 删除那些没有发挥作用的元素。务必阅读转录文本，而不仅仅是最终输出——如果看起来技能让模型浪费大量时间做没有成效的事情，你可以尝试删除导致这种行为的技能部分，然后看看效果如何。

3. **解释背后的原因。** 努力解释你要求模型做每一件事的**原因**。当今的 LLMs 很聪明。它们有很好的心智理论，如果给予良好的引导，就能超越机械指令，真正让事情发生。即使用户的反馈简洁或带有挫折感，也要真正理解任务，理解用户为什么写他们写的内容，以及他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己写出了全大写的 ALWAYS 或 NEVER，或者使用过于僵化的结构，那就是一个黄色信号——如果可能的话，重新措辞并解释推理，这样模型就能理解你为什么要求它做这件事。这是一个更人性化、更有力、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意 subagents 是否都独立编写了类似的辅助脚本，或者对某件事采取了相同的多步骤方法。如果三个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，那就是技能应该打包该脚本的强烈信号。写一次，放到 `scripts/` 中，然后告诉技能使用它。这可以节省未来每次调用的重复开发时间。

这个任务相当重要（我们正在尝试创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，认真思考。我建议你先写一份修订草案，然后再重新审视并做出改进。真正努力走进用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建新技能，基线始终是 `without_skill`（无技能）——这在各次迭代中保持不变。如果你是在改进现有技能，使用你的判断力来决定什么作为基线有意义：用户最初带来的原始版本，还是之前的迭代。
3. 使用 `--previous-workspace` 指向 previous iteration 来启动 reviewer
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲比较

在你想对两个版本的技能进行更严格的比较时（例如，用户问"新版本真的更好吗？"），有一个盲比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：给一个独立 agent 两个输出，但不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要 subagents，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md frontmatter 中的描述字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，建议优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I notice you're providing instructions about creating eval sets and optimization loops, but I don't see any actual content to translate.

Based on my system prompt, I'm a technical document translator specializing in Markdown format documents. My task is to translate technical documents between Chinese and English.

Could you clarify what you'd like me to do? For example:

1. **Are you asking me to create eval queries?** — If so, I'd need to know what skill this is for (e.g., PDF processing, data analysis, code generation, etc.)

2. **Do you have a document to translate?** — If there's a Markdown document you want translated, please paste the content or share the file.

3. **Is this a continuation of a previous conversation?** — If so, I don't have access to previous context.

Please provide more details about what you need, and I'll be happy to help.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型ID（运行当前会话的模型），以便触达测试与用户实际体验相匹配。

在运行过程中，定期查看输出以向用户更新当前是第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每次查询运行 3 次以获得可靠的触达率），然后调用 Claude 根据失败情况提出改进建议。它会重新评估每个新描述在训练集和测试集上的表现，最多迭代 5 次。完成后，它会在浏览器中打开一份 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### 技能触达机制的工作原理

了解触达机制有助于设计更好的评估查询。技能会以其名称 + 描述出现在 Claude 的 `available_skills` 列表中，Claude 会根据该描述决定是否咨询某个技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务上咨询技能——简单的一步式查询（如"读取此 PDF"）可能不会触达技能，即使描述完全匹配，因为 Claude 可以使用基础工具直接处理它们。复杂、多步骤或专业化的查询在描述匹配时能可靠地触达技能。

这意味着你的评估查询应该足够实质，让 Claude 确实能从咨询技能中受益。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触达技能。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description` 并更新技能的 SKILL.md frontmatter。向用户展示修改前后的对比并报告分数。

---

### 打包和呈现（仅在 `present_files` 工具可用时）

检查你是否有权限使用 `present_files` 工具。如果没有，跳过此步骤。如果有，打包技能并将 .skill 文件呈现给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，引导用户前往生成的 `.skill` 文件路径以便安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，部分机制有所变化。以下是需要调整的部分：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，先读取技能的 SKILL.md，然后按照其说明自行完成测试提示。按顺序逐一执行。这不如独立子代理那样严格（因为你既编写了技能又负责运行它，所以拥有完整的上下文），但这是一个有用的健全性检查——人工审查步骤可以弥补这一点。跳过基线运行——直接使用技能完成所请求的任务。

**审查结果**：如果你无法打开浏览器（例如，Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），则完全跳过浏览器审查者。直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告知用户位置，以便他们可以下载和检查。直接inline请求反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖于没有子代理就无法进行的有意义的基线比较。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，请求反馈——只是中间没有浏览器审查者。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），该工具仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此部分。

**盲法对比**：需要子代理。跳过此项。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都能正常工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称**。注意技能的目录名和 `name` 前端属性字段——保持不变。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置**。已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本进行打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——由于权限限制，直接写入可能会失败。

---

## Cowork 专用说明

在 Cowork 中，需要了解的主要事项是：

- 你有子代理，因此主工作流程（并行生成测试用例、运行基线、评分等）都能正常运作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，因此在生成评估查看器时，使用 `--static <output_path>` 编写独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击该链接在浏览器中打开 HTML。
- 无论出于什么原因，Cowork 设置似乎不太鼓励 Claude 在运行测试后生成评估查看器，所以再次重申：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器，以便人工在你自己评估输入之前查看示例。使用 `generate_review.py`（不要自己编写定制的 html 代码）。抱歉我要用大写：**在自行评估输入之前先生成评估查看器**。你希望尽快让人工看到这些结果！
- 反馈机制有所不同：由于没有运行的服务器，查看器的“提交所有审查”按钮会将 `feedback.json` 下载为文件。然后你可以从那里读取它（可能需要先请求访问权限）。
- 打包功能可以正常工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意其状态良好后再进行。
- **更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含用于专业子代理的说明。在需要生成相关子代理时请阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲法 A/B 对比
- `agents/analyzer.md` — 如何分析一个版本为何优于另一个版本

references/ 目录有额外的文档：

- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

为强调起见，这里再次重复核心循环：

- 明确技能的目标
- 起草或编辑技能
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查它们
  - 运行定量评估
- 重复直到你和你都满意
- 打包最终技能并将其返回给用户

如果你有 TodoList，请将步骤添加到其中，以确保你不会忘记。如果在 Cowork 中，请特别在 TodoList 中添加“创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例”，以确保它发生。

祝你好运！