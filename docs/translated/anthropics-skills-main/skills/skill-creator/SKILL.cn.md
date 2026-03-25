---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及衡量技能绩效。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、对技能性能进行基准测试并进行方差分析，或优化技能的描述以提高触发准确性时使用此技能。
---

# Skill Creator

用于创建新技能并对其进行迭代改进的技能。

从高层次来看，创建技能的流程如下：

- 确定你想要技能做什么以及大致如何做
- 起草技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行的同时，如果没有定量评估，就起草一些定量评估（如果已经有一些，你可以直接使用或修改，如果你觉得需要改变什么）。然后向用户解释这些评估（如果已经存在，就解释已经存在的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，让他们查看，也让用户查看定量指标
- 根据用户对结果的评估反馈重写技能（如果定量基准测试中出现任何明显的缺陷，也要据此重写）
- 重复直到满意为止
- 扩展测试集并尝试更大规模地运行

你使用此技能时的任务是找出用户处于流程的哪个阶段，然后帮助他们推进这些阶段。例如，用户可能说"我想为 X 创建一个技能"。你可以帮助缩小他们想要的范围，撰写初稿，编写测试用例，确定他们想要如何评估，运行所有提示词，并重复。

另一方面，如果他们已经有一个技能初稿。在这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行大量评估，只是和我一起凭感觉来"，你也可以这样做。

然后，在技能完成后（但同样，顺序是灵活的），你还可以运行技能描述优化器，我们有一个单独的脚本来做这件事，以优化技能的触发。

清楚了吗？清楚了。

## 与用户沟通

Skill Creator 可能会被各种熟悉程度的用户使用，这些用户对编程术语的熟悉程度差异很大。如果你没听说（而且你怎么能听说呢，这是最近才开始的现象），现在有一种趋势：Claude 的强大能力激励水管工打开终端，父母和祖父母去谷歌"如何安装 npm"。另一方面，大多数用户可能相当精通电脑。

所以请注意上下文线索，了解如何措辞！在默认情况下，给你一些概念：

- "evaluation" 和 "benchmark" 是边界情况，但可以使用
- 对于 "JSON" 和 "assertion"，你需要看到用户确实知道这些术语是什么的明显线索，然后才能不加解释地使用它们

如果你有疑问，可以简要解释术语，如果你不确定用户是否能理解，可以随时用简短的定义来澄清。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这种情况，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的修正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户措辞/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但由用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在完成这部分之前，先不要写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用的话通过子代理并行研究，否则内联研究。做好准备以减少用户的负担。

### 撰写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发、做什么。这是主要的触发机制——包括技能做什么以及何时使用的具体上下文。所有"何时使用"的信息都放在这里，而不是正文。注意：目前 Claude 有"触发不足"的倾向——在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"激进"一点。例如，不要写"How to build a simple fast dashboard to display internal Anthropic data."，你可以写"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能写作指南

#### 技能解剖学

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

1. **元数据**（名称 + 描述）- 始终加载（约 100 词）
2. **SKILL.md 正文** - 技能触发时加载（理想情况下 < 500 行）
3. **打包资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

这些字数是近似值，必要时可以超出。

**关键模式：**
- 将 SKILL.md 保持在 500 行以内；如果接近此限制，请添加额外的层级结构，并清晰指示使用该技能的语言模型接下来应该去哪里继续查阅。
- 在 SKILL.md 中清晰引用文件，并提供何时读取的指导
- 对于大型参考文件（> 300 行），应包含目录

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

#### 零惊喜原则

这不用说，但技能绝对不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果对技能进行了描述，其内容在意图上不应该让用户感到意外。不要配合创建误导性技能或旨在促进未授权访问、数据外泄或其他恶意活动的技能。不过，像"扮演 XYZ"这样的角色扮演是可以的。

#### 编写模式

在指令中优先使用祈使句形式。

**定义输出格式** - 可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有用。你可以这样格式化它们（但如果示例中有"Input"和"Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为什么重要，而不是使用生硬死板的"MUST"。运用心智理论，力求让技能具有通用性，而不是局限于特定例子。先写一份草稿，然后用全新的视角审视并改进它。

### 测试用例

在撰写技能初稿后，设计 2-3 个现实可行的测试提示——即真实用户可能会说的话。与用户分享：[不必使用完全相同的措辞] "我想尝试这几个测试用例。您觉得合适吗，或者需要补充更多？"然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时先不写断言——只保存提示内容。下一阶段在运行过程中再起草断言。

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

请参阅 `references/schemas.md` 获取完整模式（包括 `assertions` 字段，你稍后会添加该字段）。

## 运行和评估测试用例

本节是一个连续的过程——不要中途停止。不要使用 `/skill-test` 或任何其他测试技能。

将结果放在 `<skill-name>-workspace/` 中，作为技能目录的同级目录。在工作区内，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在其中每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——在执行过程中逐步创建。

### 步骤 1：在同一轮中启动所有运行（带技能版本和基线版本）

对于每个测试用例，在同一轮中启动两个子代理——一个带技能，一个不带技能。这很重要：不要先启动带技能的运行，然后再回来启动基线版本。一次启动所有内容，这样它们大致同时完成。

**带技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基线运行**（相同的提示词，但基线取决于上下文）：

- **创建新 skill**：完全不使用 skill。使用相同的提示词，不使用 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。编辑前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将基线子代理指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言暂时可以为空）。为每个评估使用描述性名称——不要只使用 "eval-0"。目录名称也使用这个名称。如果本次迭代使用了新的或修改过的评估提示词，需要为每个新的评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行中起草断言

不要只是等待运行完成——你可以高效利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们的检查内容。

好的断言应具备客观可验证性，并拥有描述性名称——它们应该在基准测试查看器中清晰可读，让浏览结果的人能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加给需要人工判断的内容。

起草好断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中将看到的内容——包括定性输出和定量基准。

### 步骤 3：运行完成后，捕获计时数据

当每个子代理任务完成时，你将收到一条包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知传递，不会保存在其他地方。在通知到达时立即处理，而不是尝试批量处理。

### 第四步：评分、聚合并启动查看器

所有运行完成后：

1. **为每次运行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）—— 查看器依赖这些精确的字段名。对于可以通过编程检查的断言，编写并运行脚本，而不是凭肉眼判断——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准测试** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器所需的确切 schema。将每个 with_skill 版本放在其 baseline 对应版本之前。

**进行分析师审查** — 阅读基准数据，揭示聚合统计数据可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容 — 例如无论技能水平如何都始终通过的断言（非区分性）、高方差评估（可能不稳定）以及时间/token 权衡。

**启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代 2+，还需要传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协同工作/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，使用 `--static <output_path>` 来生成独立的 HTML 文件而不是启动服务器。当用户点击"Submit All Reviews"时，反馈将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下次迭代可以拾取。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告诉用户**类似这样的话："我已在浏览器中打开了结果。有两个标签页——'Outputs' 允许你浏览每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能内联渲染
- **Previous Output**（迭代 2+）：折叠区域，显示上次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：文本框，输入时自动保存
- **Previous Feedback**（迭代 2+）：上次迭代的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置下的通过率、耗时和 token 使用量，以及每个评估的细分和分析师观察。

通过 prev/next 按钮或方向键导航。完成后，点击"Submit All Reviews"，所有反馈会保存到 `feedback.json`。

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

空反馈表示用户认为没问题。将你的改进集中在用户有具体反馈的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 提升技能

这是循环的核心。你已经运行了测试用例，用户也审阅了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中提炼普遍性。** 这里的大局观是，我们正在尝试创建可以重复使用数百万次（可能真的如此，甚至更多）的技能，跨越许多不同的提示。你和用户之所以在少数例子上反复迭代，是因为这样可以更快地推进。用户对这些例子了如指掌，能够快速评估新输出。但如果你和用户共同开发的技能只对这些例子有效，那就毫无用处。与其进行繁琐的过拟合修改，或者设置过度限制性的 MUST，不如尝试分支出去，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，说不定你就能找到绝佳的方案。

2. **保持提示简洁。** 移除那些没有实际作用的元素。务必阅读转录稿，而不仅仅是最终输出——如果看起来技能让模型浪费大量时间做无意义的事情，你可以尝试删除导致这种行为的技能部分，然后观察结果。

3. **解释背后的原因。** 努力解释你要求模型做每一件事的**原因**。当今的大语言模型很聪明。它们有很好的心智理论，只要给它们一个好的框架，就能超越机械指令，真正发挥作用。即使用户的反馈很简短或带有情绪，也要尝试真正理解任务，理解用户为什么写他们写的内容，以及他们实际写了什么，然后将这些理解传递到指令中。如果你发现自己在写全大写的 ALWAYS 或 NEVER，或者使用超级僵化的结构，那就是一个黄色警告信号——如果可能的话，重新措辞并解释推理，这样模型就能理解你为什么要求它做这件事。这是一种更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录稿，注意子代理是否独立编写了类似的辅助脚本或对某事采取了相同的多步骤方法。如果所有 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是技能应该捆绑该脚本的强烈信号。写一次，放到 `scripts/` 中，然后告诉技能使用它。这可以节省未来每一次调用的重复劳动。

这个任务相当重要（我们正在尝试创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，仔细思考。我建议先写一份修订草案，然后重新审视并做出改进。真正努力进入用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你创建的是一个新技能，基线始终是 `without_skill`（无技能）——这在所有迭代中保持不变。如果你是在改进现有技能，使用你的判断来决定什么作为基线合理：用户最初带来的版本，还是之前的迭代版本。
3. 使用 `--previous-workspace` 指向前一个迭代来启动审查器
4. 等待用户审阅并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈都是空的（一切都看起来很好）
- 你没有取得有意义的进展

---

## 高级：盲测比较

在某些情况下，你想要对技能的两个版本进行更严格的比较（例如，用户问"新版本真的更好吗？"），有一个盲测比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的代理，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md 前言中的描述字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，提供优化描述以提高触发准确性的服务。

### 第一步：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

# 技术文档翻译

我注意到您分享了关于创建AI技能评估集的工作流程文档。这是一份关于如何为技能编写评估查询的指南。

为了完成翻译任务，请您提供：

1. **技能名称** - 例如 "PDF Extraction" 或其他技能名称
2. **技能描述** - 技能的简要说明
3. **HTML模板** - 从 `assets/eval_review.html` 读取的模板内容（如果需要翻译）
4. **评估数据** - 您希望插入的JSON评估项目

---

**关于您分享的内容：**

这是一份关于**评估查询编写规范**的技术文档，涵盖了：

- **should-trigger 查询**（应触发技能的查询）：要求具体、真实、包含上下文细节
- **should-not-trigger 查询**（不应触发技能的查询）：近失误案例，需要技巧性设计
- **HTML审查流程**：使用 `eval_review.html` 模板进行用户审查
- **优化循环**：在后台运行评估优化

---

如果您希望我翻译整个文档（英文→中文），或者您有其他具体的技术文档需要翻译，请提供具体内容。

如果您需要我协助执行工作流程中的某个步骤（如读取HTML模板、生成评估集等），请明确说明您需要的具体操作。

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（也就是驱动当前会话的模型），这样触发测试才能匹配用户的实际体验。

在运行过程中，定期跟踪输出，向用户更新当前是第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 留置测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它重新评估每个新描述在训练集和测试集上的表现，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——按测试分数而非训练分数选择，以避免过拟合。

### 技能触发的工作原理

了解触发机制有助于设计更好的评估查询。技能会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 会根据该描述决定是否咨询技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务上咨询技能——简单的一步式查询（如"读取此 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以直接用基础工具处理。复杂、多步或专门的查询在描述匹配时能可靠地触发技能。

这意味着你的评估查询应该足够实质，让 Claude 确实能从咨询技能中受益。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md 前置元数据。向用户展示前后的对比并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查你是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，打包技能并将 .skill 文件展示给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，引导用户前往生成的 `.skill` 文件路径，以便安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审核 → 改进 → 重复），但由于 Claude.ai 没有子代理，一些机制有所变化。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。阅读技能的 SKILL.md 文件，然后按照其说明自行完成测试提示。逐个执行。这不如独立子代理那样严格（因为技能是你写的，测试也是你在跑，所以拥有完整的上下文），但这是一个有用的合理性检查——人工审核步骤可以弥补这一点。跳过基线运行——直接使用技能按要求完成任务。

**审核结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），则完全跳过浏览器审核。直接在对话中呈现结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告诉用户位置，以便他们可以下载和检查。inline 询问反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，询问反馈——只是中间没有浏览器审核。你仍然可以在文件系统上组织结果到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，跳过此部分。

**盲比**：需要子代理。跳过此部分。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称**。注意技能的目录名称和 `name` 前端字段——保持不变。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置**。已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——由于权限问题，直接写入可能会失败。

---

## Cowork 专用说明

在 Cowork 中，需要了解的主要事项是：

- 你有子代理，所以主要工作流（并行生成测试用例、运行基线、评分等）都能正常运作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，所以在生成评估查看器时，使用 `--static <output_path>` 编写独立的 HTML 文件，而不是启动服务器。然后提供一个用户可以在浏览器中打开的 HTML 链接。
- 由于某种原因，Cowork 配置似乎不鼓励 Claude 在运行测试后生成评估查看器，所以再次重申：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器，让人工在你自己审核输入之前先查看示例，使用 `generate_review.py`（不要自己写定制代码）。先说声抱歉，但我必须大写：*在审核输入之前*生成评估查看器。你希望尽快让人工看到这些例子！
- 反馈工作方式不同：由于没有运行的服务器，查看器的"提交所有审核"按钮将下载 `feedback.json` 文件。你可以从那里读取它（可能需要先请求访问权限）。
- 打包可以正常工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 subprocess 调用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意其状态良好后再保存。
- **更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。请遵循上面 Claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专业子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何对两个输出进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本为何胜出

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

为强调起见，这里再重复一次核心循环：

- 弄清楚技能是关于什么的
- 起草或编辑技能
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审核它们
  - 运行定量评估
- 重复直到你和孩子都满意
- 打包最终技能并返回给用户

如果你有 TodoList，请添加步骤以确保不会忘记。如果你在 Cowork 中，请特别在 TodoList 中放入"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审核测试用例"，以确保它发生。

祝你好运！