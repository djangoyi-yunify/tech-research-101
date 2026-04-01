---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及测量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析进行技能性能基准测试，或优化技能的描述以提高触发准确性时使用。
---

# Skill Creator

一个用于创建新技能并迭代改进它们的技能。

从高层次来看，创建技能的过程如下：

- 决定你希望技能做什么以及大致如何实现
- 编写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行的同时，如果没有定量评估，则起草一些定量评估（如果已经存在一些，你可以直接使用或修改，如果你觉得需要改变它们）。然后向用户解释它们（或者如果已经存在，解释已经存在的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供他们查看，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（以及如果定量基准测试中出现任何明显缺陷）
- 重复直到满意为止
- 扩展测试集并尝试更大规模

你使用这个技能时的任务是找出用户处于这个过程的哪个阶段，然后帮助他们推进这些阶段。例如，可能他们会说"我想为 X 创建一个技能"。你可以帮助缩小他们想要的范围，编写初稿，编写测试用例，找出他们想要如何评估，运行所有提示词，并重复。

另一方面，也许他们已经有一个技能的初稿。在这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估，只管陪我一起看看"，你也可以这样做。

然后，在技能完成后（但同样，顺序是灵活的），你也可以运行技能描述优化器，我们有单独的脚本，用于优化技能的触发。

明白了吗？明白了。

## 与用户沟通

技能创建者可能被各种熟悉程度的用户使用。如果你没有听说（你怎么能呢，这是最近才开始的现象），现在有一种趋势：Claude 的强大能力激励水管工打开终端，父母和祖父母去搜索"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索来理解如何措辞你的沟通！在默认情况下，给你一些概念：

- "evaluation" 和 "benchmark" 是边界情况，但可以使用
- 对于 "JSON" 和 "assertion"，你需要看到用户确实知道这些术语是什么的严重线索，然后才能使用它们而不解释

如果你有疑问，可以简要解释术语，如果你不确定用户是否能理解，可以随意用简短的定义来澄清。

---

## 创建一个技能

### 捕获意图

首先理解用户的意图。当前对话可能已经包含了用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在继续下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该何时触发？（什么用户短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但由用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在敲定这部分之前不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用，通过子代理并行研究，否则内联研究。准备好上下文以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**: 技能标识符
- **description**: 何时触发、做什么。这是主要的触发机制——包括技能做什么以及具体的使用场景。所有"何时使用"信息放在这里，而不是正文部分。注意：目前 Claude 有"触发不足"的倾向——在技能有用时不使用它。为了解决这个问题，请把技能描述写得稍微"强势"一点。例如，不要写成"How to build a simple fast dashboard to display internal Anthropic data."，你可以写成"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
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

技能使用三级加载系统：

1. **元数据**（名称+描述）- 始终在上下文环境中（~100词）
2. **SKILL.md 正文** - 技能触发时加载（理想情况下<500行）
3. **捆绑资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

这些词数为近似值，如有需要可以超出。

**关键模式：**

- 将 SKILL.md 保持在500行以下；如接近此限制，应添加额外的层级结构，并明确指示使用该技能的模型接下来应该去哪里跟进。
- 在 SKILL.md 中清晰引用文件，并提供何时读取的指导
- 对于大型参考文件（>300行），应包含目录

**领域组织**：当一个技能支持多个领域/框架时，按变体组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 只读取相关的参考文件。

#### 避免惊喜原则

这不用说，但 skills 不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果描述了某个 skill，其内容不应该在意图上让用户感到意外。不要配合创建误导性 skills 或设计用于促进未授权访问、数据泄露或其他恶意活动的 skills。不过，像"roleplay as an XYZ"这样的内容是可以的。

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

**示例模式** - 包含示例是很有用的。你可以这样格式化它们（但如果示例中包含 "Input" 和 "Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释为什么某些东西很重要，而不是使用刻板生硬的"MUST"指令。运用心理 Theory of Mind，尽量让技能具有通用性，而不是局限于特定示例。先写一份草稿，然后用全新的视角审视并改进它。

### 测试用例

撰写完技能草案后，设计 2-3 个真实的测试提示——也就是真实用户可能会说的话。与用户分享：「我想尝试这几个测试用例。您觉得这样可以吗，或者需要补充更多？」然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时先不写断言——运行过程中再起草断言。

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

请参阅 `references/schemas.md` 获取完整模式（包括 `assertions` 字段，你稍后会添加）。

## 运行和评估测试用例

本节是一个连续的完整流程——不要中途停止。不要使用 `/skill-test` 或任何其他测试技能。

将结果放在技能目录的同级目录 `<skill-name>-workspace/` 中。在工作空间内，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在其中每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——随着进行再创建。

### 步骤 1：在同一回合中启动所有运行（带技能版本和基线版本）

对于每个测试用例，在同一回合中启动两个子代理——一个带技能，一个不带技能。这很重要：不要先启动带技能的运行，然后再来启动基线版本。一次全部启动，这样它们几乎同时完成。

**带技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基线运行**（相同 prompt，但基线取决于上下文）：

- **创建新 skill**：没有任何 skill。相同的 prompt，不使用 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。编辑前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让基线 subagent 指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言可以先留空）。为每个 eval 取一个描述性名称——不要只是叫 "eval-0"。目录名也使用这个名称。如果本次迭代使用了新的或修改过的 eval prompt，需要为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行期间起草断言

不要仅仅等待运行完成——你可以充分利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们在基准测试查看器中应该清晰可读，这样有人一眼看到结果就能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）最好通过定性评估——不要将断言强加于需要人工判断的事物上。

一旦起草完成，请更新 `eval_metadata.json` 文件和 `evals/evals.json` 中的断言。同时向用户解释他们将在查看器中看到的内容——包括定性输出和定量基准。

### 步骤 3：运行完成时捕获计时数据

当每个子代理任务完成时，你将收到一个包含 `total_tokens` 和 `duration_ms` 的通知。请立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——数据通过任务通知传递，不会保存到其他位置。收到通知后立即处理，不要尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **为每次运行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）—— 查看器依赖这些确切的字段名称。对于可以编程检查的断言，编写并运行脚本而不是人工检查 — 脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器期望的具体 schema。
将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **进行分析审查** — 阅读基准数据，揭示汇总统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论技能如何都会通过的断言（无区分度）、高方差评估（可能不稳定）以及时间/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代2+，还需要传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下次迭代使用。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告知用户** 类似以下内容："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你可以逐个查看测试用例并留下反馈，'Benchmark' 显示定量比较。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给定的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（迭代2+）：显示上次迭代输出的折叠部分
- **Formal Grades**（如果运行了分级）：显示断言通过/失败的折叠部分
- **Feedback**：自动保存的文本框，用户输入时实时保存
- **Previous Feedback**（迭代2+）：用户上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的正确率、耗时和 token 使用量，包含每个评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键完成。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

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

空反馈意味着用户认为没有问题。将改进重点放在用户有具体投诉的测试用例上。完成后关闭查看器服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是循环的核心。你已经运行了测试用例，用户也审阅了结果，现在你需要根据用户的反馈来改进技能。

### 如何思考改进

1. **从反馈中总结规律。** 这里的大局是：我们正在尝试创建可以被使用百万次（可能确实是百万次，甚至更多，谁知道呢）的技能，应用于许多不同的提示词。你和用户之所以反复迭代少数几个例子，是因为这样可以加快速度。用户对这些例子了如指掌，能够快速评估新的输出。但如果你和用户共同开发的技能只适用于这些例子，那就毫无用处。与其进行繁琐的过拟合修改，或者设置压迫性的强制性约束，不如尝试分支出去，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你就能找到很棒的方案。

2. **保持提示词精简。** 移除那些没有发挥作用的东西。不仅要阅读最终输出，还要阅读完整的记录稿——如果看起来技能正在让模型浪费大量时间做无意义的事情，你可以尝试删除导致这种行为的技能部分，然后观察结果。

3. **解释原因。** 努力解释你要求模型做每件事的**原因**。当今的大型语言模型非常聪明。它们有很好的心智理论，如果给予良好的引导，就能超越死板的指令，真正发挥作用。即使用户的反馈简短或带有挫败感，也要试着真正理解任务，理解用户为什么写他们写的内容，以及他们实际写了什么，然后将这种理解融入指令中。如果你发现自己正在用全大写写"总是"或"永远不要"，或者使用超级 rigid 的结构，那就是一个黄色警告信号——如果可能的话，重新措辞并解释推理，这样模型就能理解你要求的事情为什么重要。这是一个更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的记录稿，注意子代理是否都独立编写了类似的辅助脚本，或采取了相同的多步骤方法。如果3个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是技能应该捆绑该脚本的强烈信号。写一次，放到 `scripts/` 目录下，然后告诉技能使用它。这可以节省未来每次调用的重复开发时间。

这个任务相当重要（我们正试图在这里创造数十亿美元的年度经济价值！），你的思考时间不是瓶颈；慢慢来，仔细考虑。我建议你写一份修订草案，然后重新审视它并做出改进。真正努力代入用户的角色，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建一个新技能，基线始终是 `without_skill`（无技能）——这在所有迭代中保持不变。如果你在改进现有技能，使用你的判断力来决定什么作为基线合理：用户最初使用的原始版本，或者之前的迭代版本。
3. 使用 `--previous-workspace` 指向 previous iteration 来启动 reviewer
4. 等待用户审阅并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲比对比

对于需要更严格地比较技能两个版本的情况（例如，用户问"新版本真的更好吗？"），有一个盲比对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：让一个独立的代理在不知道哪个是哪个的情况下判断两个输出的质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人类审阅循环通常就足够了。

---

## 描述优化

SKILL.md 前言中的描述字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，建议优化描述以提高触发的准确性。

### 第一步：生成触发评估查询

创建20个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

# Creating Eval Set for Skill

I need more information to proceed. What skill are you creating eval sets for? Please provide:

1. **Skill name** (e.g., "PDF Text Extraction", "Excel Formulas", etc.)
2. **Skill description** (brief summary of what the skill does)

Once you provide these, I'll create:

- **8-10 should-trigger queries** — realistic, varied user requests that should activate this skill
- **8-10 should-not-trigger queries** — near-miss cases that share keywords but need different tools

Then I'll generate the HTML review file following the process you outlined.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（为当前会话提供支持的那个），以便触发的测试与用户实际体验相匹配。

在运行期间，定期尾随输出以向用户提供更新，告知当前是第几次迭代以及分数如何。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 留置测试集，评估当前描述（每次查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它对训练集和测试集重新评估每个新描述，迭代最多 5 次。完成后，它会在浏览器中打开一份 HTML 报告，显示每次迭代的结果，并返回带有 `best_description` 的 JSON——按测试分数而非训练分数选择，以避免过拟合。

### 技能触发的工作原理

了解触发机制有助于设计更好的评估查询。技能会以其名称 + 描述出现在 Claude 的 `important_to_know` 列表中，Claude 会根据该描述决定是否咨询技能。重要的一点是，Claude 只会在无法独立轻松处理的任务上咨询技能——简单的一步式查询（如“读取此 PDF”）可能不会触发技能，即使描述完全匹配，因为 Claude 可以使用基本工具直接处理它们。复杂、多步骤或专业化的查询在描述匹配时会可靠地触发技能。

这意味着您的评估查询应该足够实质化，以至于 Claude 实际上会从咨询技能中受益。像“读取文件 X”这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md 头信息。向用户显示之前和之后的情况，并报告分数。

---

### 打包和演示（仅在 `present_files` 工具可用时）

检查您是否有权访问 `present_files` 工具。如果没有，请跳过此步骤。如果有，请打包技能并向用户演示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，将用户指向生成的 `.skill` 文件路径，以便他们可以安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，某些机制会有所变化。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，先阅读技能的 SKILL.md，然后按照其说明自己完成测试提示。逐个进行。这不如独立子代理严格（你编写了技能，同时也在运行它，所以你完全了解上下文），但这是一个有用的健全性检查——人工审查步骤可以弥补这一点。跳过基线运行——直接使用技能完成请求的任务。

**审查结果**：如果无法打开浏览器（例如，Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），完全跳过浏览器审查器。相反，直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告知他们位置，以便他们可以下载和检查。直接询问反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，征求反馈——只是中间没有浏览器审查器。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此部分。

**盲对比**：需要子代理。跳过此部分。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的地方都可以运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能是在要求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称**。记下技能的目录名称和 `name` 前端字段——保持不变。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前复制到可写位置**。已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——由于权限问题，直接写入可能会失败。

---

## Cowork 专用说明

在 Cowork 中，需要了解的主要事项是：

- 你有子代理，所以主要工作流（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，因此在生成评估查看器时，使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击该链接在浏览器中打开 HTML。
- 由于某种原因，Cowork 配置似乎不利于 Claude 在运行测试后生成评估查看器，所以再次重申：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器供人工查看修正前的示例，使用 `generate_review.py`（不要自己编写定制的 HTML 代码）。预先道歉，但我会大写：*在*自己评估输入之前先生成评估查看器！你想尽快让人工看到它们！
- 反馈工作方式不同：由于没有运行的服务器，查看器的"提交所有审查"按钮将下载 `feedback.json` 作为文件。然后你可以从那里读取它（可能需要先请求访问）。
- 打包工作正常——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该正常工作，因为它使用 subprocess 调用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意其状态良好后再进行。
- **更新现有技能**：用户可能是在要求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专用子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` —— 如何针对输出评估断言
- `agents/comparator.md` —— 如何在两个输出之间进行盲 A/B 对比
- `agents/analyzer.md` —— 如何分析一个版本击败另一个版本的原因

references/ 目录有额外的文档：

- `references/schemas.md` —— evals.json、grading.json 等的 JSON 结构

---

为了强调，这里再重复一遍核心循环：

- 了解技能的作用
- 起草或编辑技能
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 来帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终技能并将其返回给用户

如果你的 TodoList 中有相关内容，请添加步骤以确保你不会忘记。如果你在 Cowork 中，请特别在 TodoList 中添加"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"，以确保它会发生。

祝你好运！