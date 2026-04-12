---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及衡量技能绩效。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、执行技能性能基准测试并进行差异分析，或优化技能描述以提高触发准确性时使用此技能。
---

# 技能创建器

一个用于创建新技能并迭代改进它们的技能。

从高层次来看，创建技能的过程如下：

- 决定你希望技能做什么，以及它应该大致如何实现
- 撰写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行进行时，如果还没有定量评估，则起草一些定量评估（如果已存在一些评估，你可以直接使用或修改，如果你觉得某些内容需要更改）。然后向用户解释这些评估（或者如果已经存在，向用户解释已存在的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，让他们查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈（以及定量基准测试中发现的任何明显缺陷）重写技能
- 重复此过程直到你满意为止
- 扩展测试集并在更大规模上再试

你使用此技能时的任务是弄清楚用户处于这个过程的哪个阶段，然后帮助他们推进这些阶段。例如，可能他们会说"我想为 X 创建一个技能"。你可以帮助缩小他们的含义范围，撰写初稿，编写测试用例，找出他们想要如何评估，运行所有提示词，并重复。

另一方面，可能他们已经有一个技能初稿了。在这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活，如果用户说"我不需要运行大量评估，只管和我一起感受就行"，你也可以这样做。

然后，在技能完成后（但同样，顺序是灵活的），你也可以运行技能描述优化器，我们有一个单独的脚本来做这件事，以优化技能的触发。

清楚了吗？清楚了。

## 与用户沟通

技能创建器可能被对编码术语熟悉程度差异很大的用户使用。如果你没有听说（你怎么能听说呢，这是最近才开始流行的），现在有一种趋势，Claude 的强大激励着水管工打开终端，父母和祖父母开始搜索"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索来理解如何措辞你的沟通！在默认情况下，给你一些概念：

- "evaluation"和"benchmark"是边界情况，但可以使用
- 对于"JSON"和"assertion"，你需要在使用这些术语之前看到用户确实知道它们是什么的明显线索，否则需要解释

如果你有疑问，可以简要解释术语，如果你不确定用户是否能理解，可以随时用简短的定义来澄清术语。

---

## 创建一个技能

### 捕捉意图

首先了解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在继续下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户措辞/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但由用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项等问题。等待这部分确定后再编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用则通过子代理并行研究，否则内联研究。准备好背景知识以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发以及做什么。这是主要的触发机制——包括技能做什么以及具体的使用上下文。所有"何时使用"信息都放在这里，而不是放在正文部分。注意：目前 Claude 有一种"触发不足"的倾向——在技能有用时不使用它们。为了解决这个问题，请让技能描述稍微"积极主动"一些。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标或想要显示任何类型的企业数据时，务必使用此技能，即使他们没有明确要求'仪表板'。"
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

技能采用三级加载机制：

1. **元数据**（名称 + 描述）- 始终在上下文中呈现（约 100 字）
2. **SKILL.md 正文** - 技能触发时在上下文中加载（理想情况下 < 500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可执行而不加载）

以上字数为近似值，如需要可适当增加。

**关键模式：**

- 将 SKILL.md 保持在 500 行以内；如接近此限制，需增加一层层级结构，并清晰指示使用该技能的模型接下来应前往何处继续。
- 在 SKILL.md 中清晰引用文件，并提供读取时机指南
- 对于大型参考文件（> 300 行），需包含目录

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

#### 不意外原则

不言而喻，技能不能包含恶意软件、利用代码或任何可能危及系统安全的内容。如果技能描述了其意图，其内容不应让用户感到意外。不要配合创建误导性技能或设计用于未经授权访问、数据窃取或其他恶意活动的技能。不过，类似“扮演 XYZ 的角色扮演”这类是可以的。

#### 编写模式

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

**示例模式** - 包含示例很有用。你可以这样格式化它们（但如果示例中包含"Input"和"Output"，你可能需要稍微调整一下）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为何重要，而不是使用生硬刻板的 MUST 指令。运用心智理论，努力使技能具有普适性，而非局限于特定例子。先写一份草稿，然后用全新的视角审视并改进它。

### 测试用例

在撰写技能草案后，设计 2-3 个真实的测试提示——即真实用户可能会说的话。与用户分享：[不必使用完全相同的措辞]“我想尝试这几个测试用例。您觉得可以吗，或者还想添加更多？”

将测试用例保存到 `evals/evals.json`。暂时先不写断言——在运行进行期间的下一步再起草断言。

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

请参阅 `references/schemas.md` 获取完整模式定义（包括稍后需要添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的整体流程——中途不要停止。请勿使用 `/skill-test` 或任何其他测试技能。

请将结果放在 `<skill-name>-workspace/` 中，作为技能目录的同级目录。在工作区内，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），并在每个迭代内为每个测试用例创建目录（`eval-0/`、`eval-1/` 等）。不需要预先创建所有目录，随着进行创建即可。

### 第一步：同一轮次中启动所有运行（包括有技能和无技能的基线）

对于每个测试用例，在同一轮次中生成两个子代理——一个有技能，一个没有。这一点很重要：不要先启动有技能的运行，然后再回来启动基线。一次性启动所有运行，这样它们能差不多同时完成。

**有技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline 运行**（相同的 prompt，但 baseline 取决于上下文）：

- **创建新 skill**：完全没有 skill。使用相同的 prompt，不指定 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：旧版本。在编辑之前，对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline 子代理指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言暂时可以留空）。为每个评估使用描述性名称——不要只是写 "eval-0"。目录名也使用这个名称。如果本次迭代使用了新的或修改后的评估 prompt，需要为每个新的评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行中起草断言

不要仅仅等待运行完成——你可以充分利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们在基准测试查看器中应该清晰可读，以便有人浏览结果时能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要把需要人工判断的内容强行加上断言。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### 步骤 3：运行完成时，采集计时数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是唯一捕获这些数据的机会——它通过任务通知传递，并未在其他地方持久化。请在每个通知到达时立即处理，而不是尝试批量处理。

### 第4步：评分、聚合并启动查看器

所有运行完成后：

1. **为每次运行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每个运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用字段 `text`、`passed` 和 `evidence`（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以编程检查的断言，请编写并运行脚本而不是凭肉眼判断——脚本更快、更可靠，并且可以在多次迭代中重复使用。

2. **聚合到基准** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器所需的具体模式。

将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **进行分析师审查** — 阅读基准数据，揭示聚合统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论技能如何都始终通过的断言（无区分度）、高方差评估（可能不稳定）以及 time/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代 2+，还需要传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作环境/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，供下次迭代使用。

注意：请使用 generate_review.py 来创建查看器，无需编写自定义 HTML。

5. **告诉用户**类似这样的话："我已在浏览器中打开了结果。有两个标签页——'Outputs'可以让你逐个查看测试用例并留下反馈，'Benchmark'显示定量比较。完成后请回到这里告诉我。"

### 查看器中用户看到的内容

"Outputs"标签页一次显示一个测试用例：

- **Prompt**：下发的任务
- **Output**：技能生成的文件，尽可能内联渲染
- **Previous Output**（迭代 2+）：显示上次迭代输出的折叠部分
- **Formal Grades**（如果运行了分级）：显示断言通过/失败的折叠部分
- **Feedback**：输入时自动保存的文本框
- **Previous Feedback**（迭代 2+）：用户上次的评论，显示在文本框下方

"Benchmark"标签页显示统计摘要：每个配置的行通过率、耗时和 token 使用量，包含每次评估的细分和分析师观察结果。

通过 prev/next 按钮或方向键进行导航。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 第 5 步：读取反馈

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

空反馈意味着用户认为没问题。将改进重点放在用户有具体投诉的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是循环的核心。你已经运行了测试用例，用户已经审核了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中总结普遍规律。** 这里的大局观是，我们正在尝试创建可以百万次（也许真的可能更多，谁知道呢）跨不同提示词使用的技能。你和用户之所以在一遍又一遍地迭代少数几个示例，是因为这样可以帮助加快速度。用户对这些示例了如指掌，能够快速评估新输出。但如果你和用户共同开发的技能只能用于这些示例，那它就毫无用处。与其进行繁琐的过拟合修改，或者使用强制性的MUST规则，不如尝试分支出去，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你就能找到很棒的方案。

2. **保持提示词精简。** 移除没有起到作用的元素。一定要阅读转录文本，而不仅仅是最终输出——如果看起来技能让模型浪费了大量时间做没有成效的事情，你可以尝试删除导致这种结果的技能部分，然后观察会发生什么。

3. **解释背后的原因。** 努力解释你要求模型做每一件事的**原因**。当今的LLM非常聪明。它们具有良好的心理理论能力，如果给予良好的引导，就能超越死板的指令，真正发挥作用。即使用户的反馈简短或带有挫败感，也要努力真正理解任务，理解用户为什么写他们所写的内容，以及他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己在全大写使用ALWAYS或NEEVER，或者使用超级僵化的结构，这是一个黄色信号——如果可能的话，重新措辞并解释推理，这样模型就能理解你要求它做的事情为什么重要。这是一个更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意是否所有子代理都独立编写了类似的辅助脚本或采用了相同的多步骤方法。如果3个测试用例都导致子代理编写了`create_docx.py`或`build_chart.py`，这就是一个强烈的信号，表明技能应该捆绑该脚本。写一次，放到`scripts/`中，然后告诉技能使用它。这可以为未来的每次调用节省重复开发的时间。

这个任务相当重要（我们正尝试在这里创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，仔细思考。我建议先写一份修改草案，然后重新审视并加以改进。真的尽最大努力进入用户的头脑，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的`iteration-<N+1>/`目录中，包括基线运行。如果你正在创建一个新技能，基线始终是`without_skill`（无技能）——这在迭代中保持不变。如果你正在改进现有技能，使用你的判断力判断什么作为基线合理：用户最初带来的原始版本，或者上一次迭代。
3. 使用`--previous-workspace`指向上一次迭代来启动审核者
4. 等待用户审核并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲测比较

在你想对技能的两个版本进行更严格的比较（例如，用户问"新版本真的更好吗？"）的情况下，有一个盲测比较系统。阅读`agents/comparator.md`和`agents/analyzer.md`了解详情。基本思路是：给一个独立的代理两个输出，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审核循环通常就足够了。

---

## 描述优化

SKILL.md前言中的描述字段是决定Claude是否会调用技能的主要机制。在创建或改进技能后，主动提出优化描述以提高触发准确性。

### 第一步：生成触发评估查询

创建20个评估查询——混合应该触发和不应该触发的类型。保存为JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I don't see the skill name or description in your message. Could you provide:

1. **The skill name** (e.g., "PDF Processing", "Excel Formulas", etc.)
2. **The skill description** (what the skill does)

Once I have that information, I'll create the eval set following your guidelines:

- **Should-trigger (8-10)**: Realistic queries with file paths, context, column names, etc. — different phrasings, edge cases, cases where the skill competes with another
- **Should-not-trigger (8-10)**: Near-misses with shared keywords but different actual intent — genuinely tricky cases, not obvious non-matches

Then I'll use the HTML template to present it for review.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型ID（为当前会话提供支持的那个），以便触发测试与用户实际体验相符。

运行期间，定期查看输出，向用户更新当前是第几次迭代以及分数情况。

它自动处理整个优化循环。它将评估集拆分为60%训练集和40%保留测试集，评估当前描述（每个查询运行3次以获得可靠的触发率），然后调用Claude根据失败情况提出改进建议。它重新评估每个新描述在训练集和测试集上的表现，最多迭代5次。完成后，它会在浏览器中打开一份HTML报告，显示每次迭代的结果，并返回包含`best_description`的JSON——根据测试分数而非训练分数选择，以避免过拟合。

### 技能触发的工作原理

理解触发机制有助于设计更好的评估查询。技能会出现在Claude的`available_skills`列表中，包含其名称和描述，Claude根据该描述决定是否咨询技能。需要知道的重要一点是，Claude只会在无法自行轻松处理的任务上咨询技能——简单的一次性查询如"读取这个PDF"可能不会触发技能，即使描述完全匹配，因为Claude可以直接使用基础工具处理。复杂、多步骤或专业化的查询在描述匹配时能可靠地触发技能。

这意味着你的评估查询应该足够实质化，以便Claude确实能从咨询技能中受益。像"读取文件X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 步骤4：应用结果

从JSON输出中获取`best_description`，并更新技能的SKILL.md frontmatter。向用户展示修改前后的对比，并报告分数。

---

### 打包和展示（仅在`present_files`工具可用时）

检查是否可以使用`present_files`工具。如果没有，跳过此步骤。如果有，打包技能并将.skill文件展示给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，引导用户到生成的 `.skill` 文件路径，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，某些机制有所变化。以下是需要调整的地方：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，请阅读技能的 SKILL.md，然后按照其说明自己完成测试提示。逐个进行。这不如独立子代理严格（你编写了技能并正在运行它，所以你拥有完整的上下文），但这是一个有用的健全性检查——人工审查步骤可以弥补。跳过基线运行——只需使用技能按要求完成任务即可。

**审查结果**：如果无法打开浏览器（例如，Claude.ai 的虚拟机没有显示，或者你在远程服务器上），请完全跳过浏览器审查。相反，直接在对话中呈现结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），请将其保存到文件系统并告知他们位置，以便他们可以下载和检查。直接请求反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖于没有子代理就无意义的基线比较。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，请求反馈——只是中间没有浏览器审查者。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此部分。

**盲比较**：需要子代理。跳过此部分。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能是在要求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称。** 注意技能的目录名称和 `name` 前端字段——保持不变。例如，如果安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑之前复制到可写位置。** 安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，请先暂存到 `/tmp/`**，然后复制到输出目录——由于权限问题，直接写入可能会失败。

---

## Cowork 专用说明

在 Cowork 中，需要了解的主要事项是：

- 你有子代理，所以主要工作流（并行生成测试用例、运行基线、评分等）都能正常运作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也是可以接受的。）
- 你没有浏览器或显示，因此在生成评估查看器时，使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击该链接在浏览器中打开 HTML。
- 无论出于什么原因，Cowork 设置似乎不鼓励 Claude 在运行测试后生成评估查看器，所以再重申一遍：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器供人工查看示例，然后再自己评估输入，使用 `generate_review.py`（不要编写自己的定制 HTML 代码）。提前道歉，但我会大写：**在评估输入之前先生成评估查看器**。你想尽快让人工看到这些结果！
- 反馈工作方式不同：由于没有运行的服务器，查看器的"提交所有评论"按钮会将 `feedback.json` 下载为文件。然后你可以从那里读取它（你可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并用户同意其状态良好后再进行。
- **更新现有技能**：用户可能是在要求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时请阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本胜出另一个

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

为了强调，这里再次重复核心循环：

- 确定技能是做什么的
- 起草或编辑技能
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查它们
  - 运行定量评估
- 重复直到你和你都满意
- 打包最终技能并将其返回给用户。

如果你有 TodoList，请添加步骤以确保你不会忘记。如果你在 Cowork 中，请特别在 TodoList 中放入"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"，以确保它发生。

祝你好运！