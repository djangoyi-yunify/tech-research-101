---
name: skill-creator
description: 创建新技能、修改和改进现有技能，并衡量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析对技能性能进行基准测试，或优化技能的描述以提高触发准确性时使用。
---

# 技能创建器

用于创建新技能并迭代改进它们的技能。

从高层来看，创建技能的流程如下：

- 决定你想要技能做什么以及大致如何实现
- 撰写技能的初稿
- 创建几个测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 当后台运行发生时，如果没有定量评估，则起草一些（如果已有一些，你可以直接使用或修改，如果你觉得有什么需要改变的话）。然后向用户解释它们（或者如果已经存在，解释那些已存在的）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果让他们查看，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（如果定量基准测试中出现任何明显的缺陷，也要考虑）
- 重复直到满意为止
- 扩大测试集并尝试更大规模地运行

你使用这个技能时的任务是弄清楚用户处于流程的哪个阶段，然后加入并帮助他们完成这些阶段。例如，可能他们说"我想为 X 创建一个技能"。你可以帮助缩小他们的需求范围，撰写初稿，编写测试用例，弄清楚他们想要如何评估，运行所有提示词，并重复。

另一方面，可能他们已经有一个技能的初稿。在这种情况下，你可以直接进入评估/迭代循环。

当然，你应该始终保持灵活性，如果用户说"我不需要运行大量评估，只是随便聊聊"，你也可以那样做。

然后，在技能完成后（但同样，顺序是灵活的），你还可以运行技能描述优化器，我们有单独的脚本来做这件事，以优化技能的触发。

好的？好的。

## 与用户沟通

技能创建器可能被各种熟悉程度的用户使用——从对编码术语不太熟悉到非常熟悉。如果你没有听说（你怎么能呢，这是最近才开始流行的），现在有一种趋势：Claude 的强大能力激励着水管工打开终端，父母和祖父母去谷歌"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索来理解如何措辞！在默认情况下，只是给你一些概念：

- "evaluation"和"benchmark"是边界情况，但可以用
- 对于"JSON"和"assertion"，在你使用它们而不解释之前，需要看到用户确实知道这些术语的严重线索

如果你有疑问，可以简要解释术语，如果你不确定用户是否能理解，可以随意用简短的定义来澄清。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如他们说"把这个变成一个技能"）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在继续下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户措辞/上下文）
3. 预期的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但让用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖关系。在你把这部分弄清楚之前，先不要写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用的话，通过子代理并行研究，否则内联研究。准备好背景知识来减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发，做什么。这是主要的触发机制——包括技能做什么以及具体的使用上下文。所有"何时使用"的信息都放在这里，而不是在正文里。注：目前 Claude 有一种"触发不足"的倾向——在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"激进"一点。例如，不要写"How to build a simple fast dashboard to display internal Anthropic data."，你可以写"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

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

1. **元数据**（名称 + 描述）- 始终加载（~100 字）
2. **SKILL.md 正文** - 技能触发时加载（建议 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可执行无需加载）

以上字数为近似值，必要时可超出限制。

**关键模式：**

- 将 SKILL.md 控制在 500 行以内；如接近此限制，请添加额外的层级结构，并清晰指示使用该技能的模型应前往何处继续跟进。
- 在 SKILL.md 中清晰引用文件，并提供何时读取的指导。
- 对于大型参考文件（>300 行），请包含目录。

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

#### 不惊喜原则

毋庸置疑，技能不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果对技能进行了描述，其内容不应在意图上让用户感到惊讶。不要配合创建误导性技能或旨在促进未授权访问、数据外泄或其他恶意活动的请求。不过，"扮演 XYZ"这类角色扮演是可以的。

#### 编写模式

在指令中优先使用祈使句。

**定义输出格式** - 可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例会很有帮助。你可以这样格式化它们（但如果示例中有 "Input" 和 "Output"，你可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为什么重要，而不是使用生硬刻板的 MUST 指令。运用心理理论（theory of mind），努力让技能具有通用性，而不是局限于具体示例。先写一份草稿，然后用全新的视角审视并改进它。

### 测试用例

在撰写技能草稿后，设计 2-3 个切合实际的测试提示——真实用户可能会说的话。与用户分享：[不一定要使用完全相同的措辞]“我想尝试这几个测试用例。看起来是否合适，或者你想补充更多？”然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时先不写断言——只保存提示内容。在运行进行期间，你可以在下一步撰写断言。

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

## 运行和评估测试用例

本节是一个连续的完整流程——请勿中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

请将结果放在 `<skill-name>-workspace/` 中，作为skill目录的同级目录。在workspace中，按迭代（`iteration-1/`、`iteration-2/`等）组织结果，在每个迭代内，每个测试用例对应一个目录（`eval-0/`、`eval-1/`等）。无需预先创建所有目录——在执行过程中逐步创建即可。

### Step 1: 在同一轮次中启动所有运行（包括有skill和无skill的baseline）

对于每个测试用例，在同一轮次中启动两个子代理——一个有skill，一个没有。这很重要：请勿先启动有skill的运行，稍后再回来启动baseline。一次全部启动，这样它们会在大致相同的时间完成。

**有skill的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基准运行**（相同的提示词，但基准取决于上下文）：

- **创建新技能**：完全没有技能。相同的提示词，无技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：旧版本。在编辑之前，快照技能（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将基准子代理指向快照。保存到 `old_skill/outputs/`。

为每个测试用例编写一个 `eval_metadata.json`（断言暂时可以为空）。给每个评估一个描述性名称——不仅仅是 "eval-0"。目录也使用这个名称。如果此迭代使用新的或修改的评估提示词，为每个新的评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：当运行进行时，起草断言

不要只是被动等待运行完成——你可以有效地利用这段时间。为每个测试用例起草定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应当是客观可验证的，并且具有描述性名称——它们应该在基准测试查看器中清晰呈现，以便查看结果的人能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的事物上。

一旦起草好断言，请更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### 步骤 3：运行完成时，捕获计时数据

当每个子代理任务完成时，你将收到包含 `total_tokens` 和 `duration_ms` 的通知。请立即将此数据保存到运行目录中的 `timing.json` 文件：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知传递，不会被持久化到其他位置。在通知到达时立即处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **为每次运行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录中的 `grading.json`。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）—— 查看器依赖于这些确切的字段名称。对于可以编程检查的断言，编写并运行脚本而不是人工检查 — 脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及平均值 ± 标准差和差值。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器期望的确切模式。

将每个 with_skill 版本放在其对应的 baseline 之前。

**进行分析师审查** — 阅读基准测试数据，揭示聚合统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论技能如何都总是通过的断言（无区分度）、高方差评估（可能不稳定）以及 time/token 权衡。

**启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代 2+，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作环境/无头环境：** 如果 `webbrowser.open()` 不可用或环境无显示器，请使用 `--static <output_path>` 写入独立的 HTML 文件而非启动服务器。当用户点击"Submit All Reviews"时，反馈将下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下一次迭代拾取。

注意：请使用 generate_review.py 创建查看器，无需编写自定义 HTML。

5. **告知用户** 例如："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个查看测试用例并留下反馈，'Benchmark' 显示定量对比。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给定的任务
- **Output**：技能产生的文件，在可能的情况下内联渲染
- **Previous Output**（迭代 2+）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：一个文本框，输入时自动保存
- **Previous Feedback**（迭代 2+）：他们上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的总通过率、耗时和 token 使用量，以及每次评估的细分和分析师观察结果。

导航通过 prev/next 按钮或方向键完成。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

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

空反馈意味着用户认为没问题。将你的改进重点放在用户有具体投诉的测试用例上：

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是循环的核心。你已经运行了测试用例，用户也已经审查了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中总结规律。** 这里的大局是：我们正在尝试创建可以被使用上百万次（也许是字面意思，甚至更多，谁知道呢）的技能，应用于各种不同的提示词。你和用户之所以一遍又一遍地迭代少数几个例子，是因为这样可以帮助加快速度。用户对这些例子了如指掌，可以快速评估新的输出。但是如果你和用户共同开发的技能只能用于这些例子，那就没有任何意义。与其做一些繁琐的过度拟合修改，或者使用强制性的 MUST 指令，如果有顽固的问题，你可以尝试分支出去，使用不同的隐喻，或者推荐不同的工作模式。尝试的成本相对较低，也许你就能找到很棒的方案。

2. **保持提示词简洁。** 移除那些没有发挥作用的内容。一定要阅读转录文本，而不仅仅是最终输出——如果看起来技能正在让模型浪费时间做些没有成效的事情，你可以尝试删除导致这种结果的技能部分，然后看看会发生什么。

3. **解释背后的原因。** 努力解释你要求模型做每一件事的**原因**。当今的 LLM 很聪明，它们有很好的心智理论，如果给它们一个好的框架，就能超越死板的指令，真正让事情发生。即使用户的反馈很简洁或带有挫败感，也要试着真正理解任务，理解用户为什么写他们写的内容，以及他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己写了全大写的 ALWAYS 或 NEVER，或者使用了超级僵化的结构，那就是一个黄色警示信号——如果可能的话，重新组织语言并解释推理过程，这样模型就能理解你要求的事情为什么重要。这是一个更人性化、更有力量、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意 subagent 是否都独立编写了类似的辅助脚本，或者对某件事情采取了相同的多步骤方法。如果三个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，那就是一个强烈的信号，表明技能应该捆绑这个脚本。只写一次，放到 `scripts/` 目录下，然后告诉技能去使用它。这可以为以后每一次调用省去重复造轮子的麻烦。

这个任务相当重要（我们正试图在这里创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，仔细思考。我建议先写一份修订草稿，然后再重新审视并做出改进。真正尽力去了解用户想要什么、需要什么。

### 迭代循环

改进技能之后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建一个新技能，基线始终是 `without_skill`（无技能）——它在各次迭代中保持不变。如果你在改进一个现有技能，使用你的判断力来决定什么作为基线有意义：用户最初带来的原始版本，还是上一次迭代。
3. 使用 `--previous-workspace` 指向上一次迭代来启动 reviewer
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲比对比

在某些情况下，如果你想更严格地比较技能的两个版本（例如，用户问"新版本实际上更好吗？"），有一个盲比对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的 agent，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要 subagents，大多数用户不需要。人类审查循环通常就足够了。

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

# 需要更多信息

您提到了"this skill"和技能描述，但我在消息中没有看到具体是哪个技能。请问：

1. **技能名称**是什么？
2. **技能描述**是什么？

提供这些信息后，我可以帮您：

- 创建 8-10 个 **should-trigger** 查询（不同措辞，覆盖边缘情况）
- 创建 8-10 个 **should-not-trigger** 查询（near-miss 案例，看起来相关但实际需要不同工具）
- 按照 Step 2 的流程生成 HTML 模板供您审核

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（为当前会话提供支持的那个），以便触发测试与用户的实际体验相匹配。

在运行过程中，定期查看输出，向用户更新当前是第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每次查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它对训练集和测试集重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开一份 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### 技能触发原理

了解触发机制有助于设计更好的评估查询。技能会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 会根据该描述决定是否咨询技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务上咨询技能——简单、一步式的查询（如"读取这个 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理它们。复杂、多步骤或专门的查询在描述匹配时能可靠地触发技能。

这意味着你的评估查询应该足够实质，让 Claude 确实能从咨询技能中受益。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md 前置元数据。向用户展示修改前后的对比并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时执行）

检查是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，请打包技能并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，将生成的 `.skill` 文件路径告知用户，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，部分机制有所变化。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，请阅读技能的 SKILL.md，然后按照其说明自行完成测试提示词。逐个执行。这不如独立子代理那样严格（你编写了技能，同时也在运行它，所以你拥有完整的上下文），但这是一个有用的健全性检查——而人工审查步骤可以弥补这一点。跳过基线运行——直接使用技能完成任务。

**审查结果**：如果你无法打开浏览器（例如 Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），请完全跳过浏览器审查器。相反，直接在对话中展示结果。对于每个测试用例，显示提示词和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），请将其保存到文件系统并告诉用户所在位置，以便他们可以下载和检查。直接询问反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，获取反馈——只是中间没有浏览器审查器。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此部分。

**盲比**：需要子代理。跳过此部分。

**打包**：`package_skill.py` 脚本可以在任何有 Python 和文件系统的地方运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称**。注意技能的目录名和 `name` 前置元数据字段——保持不变。例如，如果安装的技能是 `research-helper`，请输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑之前复制到可写位置**。安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里进行编辑，然后从副本打包。
- **如果手动打包，请先暂存到 `/tmp/`**，然后复制到输出目录——由于权限问题，直接写入可能会失败。

---

## Cowork 专用说明

如果你在 Cowork，主要需要了解以下几点：

- 你有子代理，所以主要工作流（并行生成测试用例、运行基线、评分等）都可以正常运作。（但是，如果遇到严重的超时问题，按顺序运行测试提示词也是可以的。）
- 你没有浏览器或显示器，因此在生成评估查看器时，请使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 无论如何，Cowork 配置似乎不利于 Claude 在运行测试后生成评估查看器，所以再次重申：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器供人工查看改进前的示例，使用 `generate_review.py`（不要自己编写定制的 html 代码）。抱歉我要用大写：**在自行评估输入之前先生成评估查看器**。你希望尽快让人工看到这些示例！
- 反馈工作方式不同：由于没有运行中的服务器，查看器的"提交所有审查"按钮会将 `feedback.json` 下载为文件。然后你可以从中读取它（你可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用子进程通过 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意它已经完善后再进行。
- **更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含用于专门子代理的说明。你需要在生成相关子代理时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何对两个输出进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本击败另一个版本的原因

references/ 目录包含其他文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

为强调起见，这里再次重复核心循环：

- 明确技能的作用
- 起草或编辑技能
- 使用 claude-with-access-to-the-skill 对测试提示词进行测试
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户查看它们
  - 运行定量评估
- 重复直到你满意并且用户满意
- 打包最终技能并将其返回给用户

请将这些步骤添加到你的待办事项列表中，以确保你不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"放入你的待办事项列表中，以确保它会发生。

祝你好运！