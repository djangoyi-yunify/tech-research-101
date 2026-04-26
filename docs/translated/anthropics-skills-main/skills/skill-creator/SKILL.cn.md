# Skill Creator

一个用于创建新技能并迭代改进它们的技能。

从高层次来看，创建技能的过程如下：

- 决定你想要技能做什么，以及大致如何实现
- 撰写技能的初稿
- 创建一些测试提示词，并在它们上运行 `claude-with-access-to-the-skill`
- 帮助用户定性和定量地评估结果
  - 当后台运行时，如果没有定量评估基准（evals），则起草一些（如果已存在，可以直接使用或根据需要修改）。然后向用户解释它们（如果已存在，则解释已存在的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（也包括从定量基准中发现的明显缺陷）
- 重复直到满意为止
- 扩大测试集并尝试更大规模的测试

使用此技能时，你的工作是弄清楚用户处于哪个阶段，然后帮助他们推进这些阶段。例如，也许他们说"我想为 X 创建一个技能"。你可以帮助细化他们的需求，撰写初稿，编写测试用例，确定他们想要的评估方式，运行所有提示词，并重复这个过程。

另一方面，也许他们已经有一个技能初稿。在这种情况下，你可以直接进入评估/迭代环节。

当然，你应该始终保持灵活，如果用户说"我不需要运行一堆评估，只是想随便聊聊"，你也可以那样做。

然后，在技能完成后（同样，顺序是灵活的），你还可以运行技能描述优化器，我们有专门的脚本用于优化技能的触发机制。

明白了吗？很好。

## 与用户沟通

Skill Creator 很可能会被各类熟悉程度不同的用户使用——从对编程术语一窍不通到精通都有。如果你还没听说（你怎么可能听说呢，这只是最近才开始的趋势），现在有一股潮流，Claude 的能力激发了管道工打开终端、父母和祖父母去搜索"如何安装 npm"。另一方面，大部分用户可能都具有较高的计算机素养。

所以请注意上下文线索来理解如何措辞！在默认情况下，给你一些参考：

- "evaluation"和"benchmark"属于边缘情况，但可以用
- 对于"JSON"和"assertion"，你需要看到用户明显表现出知道这些术语的含义后，才能不加解释地使用

如果有疑问，简要解释术语是可以的，如果不确定用户是否能理解某个术语，可以加上简短定义。

## 创建技能

### 捕捉意图

首先理解用户的意图。当前对话可能已经包含用户想要捕捉的工作流程（例如，他们说"把这个变成一个技能"）。如果是的话，首先从对话历史中提取答案——使用的工具、步骤顺序、用户的修改、观察到的输入/输出格式。用户可能需要填补空白，应该在进入下一步之前让他们确认。

1. 这个技能应该让 Claude 能够做什么？
2. 什么时候应该触发这个技能？（什么用户用语/上下文）
3. 期望的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。输出主观的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认值，但让用户决定。

### 访谈与研究

主动询问边界情况、输入/输出格式、示例文件、成功标准和依赖项。在这些问题弄清楚之前，先不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有子代理可用则并行研究，否则内联研究。带着上下文准备，减少用户的负担。

### 编写 SKILL.md

基于用户访谈，填写以下组件：

- **name**: 技能标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包括技能做什么以及特定的使用上下文。所有"何时使用"的信息都放在这里，不放在正文里。注意：目前 Claude 有"触发不足"的倾向——即在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"激进"一些。例如，不要写"如何构建一个简单的快速仪表盘来显示内部 Anthropic 数据"，你可以写"如何构建一个简单的快速仪表盘来显示内部 Anthropic 数据。当用户提到仪表盘、数据可视化、内部指标，或想要显示任何类型的公司数据时，务必使用此技能，即使他们没有明确要求'仪表盘'。"
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能编写指南

#### 技能的组成结构

...

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

1. **元数据**（名称 + 描述）- 始终在上下文中（~100 词）
2. **SKILL.md 主体** - 技能触发时在上下文中（理想情况下 <500 行）
3. **捆绑资源** - 按需加载（无限量，脚本可在不加载的情况下执行）

这些字数统计是近似值，如有需要可以超出。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如果接近此限制，请添加额外的层次结构，并提供清晰的指引，说明使用该技能的模型下一步应前往何处继续跟进。
- 从 SKILL.md 中明确引用文件，并提供关于何时读取它们的指引
- 对于大型参考文件（>300 行），请包含目录

**领域组织**：当技能支持多个领域/框架时，按变体组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 只读取相关的参考文件。

#### 不令人惊讶原则

这不用说，但技能必须不包含恶意软件、利用代码或任何可能危害系统安全的内容。如果描述了技能的用途，其内容不应该让用户感到意外。不要配合创建误导性技能或旨在促进未经授权访问、数据泄露或其他恶意活动的技能请求。不过，像“扮演 XYZ 角色”这样的内容是可以的。

#### 编写模式

指令中优先使用祈使句。

**定义输出格式** - 你可以这样做：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有帮助。你可以像这样格式化它们（不过如果示例中包含“Input”和“Output”，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为何重要，而不是使用老套生硬的"MUST"（必须）指令。运用心理理论（Theory of Mind），尽量让技能具有通用性，不要过于局限于特定示例。先写一份草稿，然后用新的视角审视并改进。

### 测试用例

撰写完技能草案后，创建2-3个真实的测试提示——即真实用户可能会说的话类型。将其分享给用户：[不必使用完全相同的措辞]"以下是我想尝试的几个测试用例。你觉得这些合适吗，或者想添加更多？"然后运行它们。

将测试用例保存到 `evals/evals.json`。暂不编写断言——只需提示。在运行进行期间，你将在下一步起草断言。

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

完整的 schema（包括`assertions`字段，你稍后会添加）请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续流程——不要中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放在技能目录的同级目录 `<skill-name>-workspace/` 中。在 workspace 内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代目录下，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——边进行边创建。

### 步骤 1：在同一轮次中启动所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮次中启动两个子代理——一个有技能，一个没有技能。这很重要：不要先启动 with-skill 运行，然后再回来启动 baseline。先同时启动所有任务，以便它们差不多同时完成。

**With-skill 运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline 运行**（相同 prompt，但 baseline 取决于上下文）：

- **创建新技能**：完全不使用技能。使用相同的 prompt，不指定技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：使用旧版本。在编辑之前，先对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline 子代理指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言部分可先留空）。根据测试内容为每个 eval 赋予描述性名称——不要简单地命名为 "eval-0"。目录名也使用该名称。如果本次迭代使用了新的或修改过的 eval prompt，需要为每个新 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第 2 步：在运行进行期间起草断言

不要只是等待运行完成——你可以利用这段时间。 为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查并解释它们检查的内容。

好的断言应该能够被客观验证，并且具有描述性名称——它们应该在基准测试查看器中清晰可读，这样人们在查看结果时能够立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的内容。

一旦起草完断言，请更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准测试。

### 第 3 步：当运行完成时，捕获计时数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。请立即将此数据保存到运行目录中的 `timing.json` 文件：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它通过任务通知传递，不会被持久化到其他位置。请在收到每个通知时立即处理，而不是尝试批量处理。

### 步骤 4：评分、汇总并启动查看器

所有运行完成后：

1. **对每次运行进行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（不要使用 `name`/`met`/`details` 或其他变体）— 查看器依赖于这些确切的字段名。对于可以编程检查的断言，请编写并运行脚本而非肉眼观察 — 脚本更快、更可靠，并且可以在迭代中复用。

2. **汇总到基准测试** — 运行 skill-creator 目录中的汇总脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参阅 `references/schemas.md` 以了解 viewer 所需的精确 schema。将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **进行分析师检查** — 读取 benchmark 数据，揭示汇总统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results" 部分）了解需要关注的内容 — 例如无论技能如何都总是通过的断言（非区分性）、高方差的评估（可能 flaky）以及 time/token 权衡。

4. **启动 viewer**，同时加载定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第2次及之后的迭代，还需要传入 `--previous-workspace <workspace>/iteration-<N-1>` 参数。

**Cowork / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 将独立的 HTML 文件写入指定路径，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈将作为 `feedback.json` 文件下载。下载后，将 `feedback.json` 复制到工作空间目录中，以便下一次迭代拾取。

注：请使用 generate_review.py 来生成查看器，无需编写自定义 HTML。

### 查看器中的用户界面

"Outputs"选项卡每次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（第2次及之后的迭代）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：自动保存的文本框
- **Previous Feedback**（第2次及之后的迭代）：上次迭代的评论，显示在文本框下方

"Benchmark"选项卡显示统计摘要：每个配置的通过率、耗时和 token 使用量，以及每个评估的细化和分析观察。

导航通过上一个/下一个按钮或方向键实现。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤5：读取反馈

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

空反馈表示用户认为没有问题。请将你的改进重点放在用户有具体抱怨的测试用例上。

完成后请关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

## 改进技能

这是循环的核心部分。你已经运行了测试用例，用户也审查了结果，现在需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中提炼通用规律。** 这里的大局是：我们正在尝试创建能够被使用上百万次（可能字面意义上，甚至更多）的技能，这些技能会应用于各种不同的 prompt。你和用户在这里反复迭代只有几个示例，是为了加快速度。用户对这些示例了如指掌，能够快速评估新的输出。但如果你和用户合作开发的技能只适用于这些示例，那它就毫无用处。与其进行一些零碎的过拟合修改，或者设置过于严苛的 MUST 约束，不如尝试分支出去，使用不同的比喻，或推荐不同的工作模式。尝试的成本相对较低，说不定你会发现真正很棒的方案。

2. **保持 prompt 精简。** 删除那些没有发挥作用的内容。不仅要看最终输出，还要阅读转录文本——如果看起来技能让模型浪费了大量时间做低效的事情，你可以尝试删除导致模型这样做的技能部分，然后观察结果。

3. **解释原因。** 努力解释你要求模型做的每件事背后的**原因**。现在的 LLM 很聪明。它们有很好的心智理论，当给它们一个好的框架时，能够超越死板的指令，真正发挥作用。即使用户的反馈简短或令人沮丧，也要努力理解任务，理解用户为什么写他们写的东西，理解他们实际写的内容，然后将这种理解传达给指令。如果你发现自己用全大写写 ALWAYS 或 NEVER，或者使用过于僵化的结构，那就是一个黄色警告——如果可能的话，重新组织并解释推理过程，让模型理解你要求它做的事情为什么重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意子代理是否都独立编写了类似的辅助脚本或对某事物采取了相同的多步骤方法。如果 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，那就是强烈信号，表明技能应该捆绑那个脚本。写一次，放在 `scripts/` 中，并告诉技能使用它。这可以节省未来每次调用重新发明轮子。

这个任务相当重要（我们正在尝试每年创造数十亿美元的经济价值！）你的思考时间不是瓶颈；慢慢来，真正仔细思考。我建议写一份修订草案，然后重新审视它并做出改进。真正尽你所能进入用户的头脑，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建新技能，基线始终是 `without_skill`（无技能）——它在迭代过程中保持不变。如果你正在改进现有技能，用你的判断来决定什么作为基线才有意义：用户最初使用的原始版本，还是之前的迭代。
3. 启动 reviewer，指定 `--previous-workspace` 指向之前的迭代
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈都是空的（一切都看起来很好）
- 你没有取得有意义的进展

---

## 高级：盲测对比

对于需要更严格比较技能两个版本的情况（例如用户问"新版本实际上更好吗？"），有一个盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：给一个独立代理两个输出，但不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md frontmatter 中的描述字段是决定 Claude 是否调用技能的主要机制。在创建或改进技能后，主动提出优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——包括应该触发和不应该触发的混合。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I need clarification before proceeding. What skill should I create the eval set for?

Please provide:
1. **Skill name** (e.g., "pdf-extract", "image-resize", "code-review")
2. **Skill description** (what the skill does)

Once you share those, I'll draft the should-trigger and should-not-trigger queries for your review.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

Use the model ID from your system prompt (the one powering the current session) so the triggering test matches what the user actually experiences.

While it runs, periodically tail the output to give the user updates on which iteration it's on and what the scores look like.

This handles the full optimization loop automatically. It splits the eval set into 60% train and 40% held-out test, evaluates the current description (running each query 3 times to get a reliable trigger rate), then calls Claude to propose improvements based on what failed. It re-evaluates each new description on both train and test, iterating up to 5 times. When it's done, it opens an HTML report in the browser showing the results per iteration and returns JSON with `best_description` — selected by test score rather than train score to avoid overfitting.

### How skill triggering works

Understanding the triggering mechanism helps design better eval queries. Skills appear in Claude's `available_skills` list with their name + description, and Claude decides whether to consult a skill based on that description. The important thing to know is that Claude only consults skills for tasks it can't easily handle on its own — simple, one-step queries like "read this PDF" may not trigger a skill even if the description matches perfectly, because Claude can handle them directly with basic tools. Complex, multi-step, or specialized queries reliably trigger skills when the description matches.

This means your eval queries should be substantive enough that Claude would actually benefit from consulting a skill. Simple queries like "read file X" are poor test cases — they won't trigger skills regardless of description quality.

### Step 4: Apply the result

Take `best_description` from the JSON output and update the skill's SKILL.md frontmatter. Show the user before/after and report the scores.

---

### Package and Present (only if `present_files` tool is available)

Check whether you have access to the `present_files` tool. If you don't, skip this step. If you do, package the skill and present the .skill file to the user:

使用你的 system prompt 中的 model ID（驱动当前会话的那个），这样触发测试就能匹配用户实际体验。

运行时，定期 tail 输出，给用户更新当前是第几次迭代以及分数情况。

这会自动处理完整的优化循环。它将 eval 集划分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它在训练集和测试集上重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——选择依据是测试分数而非训练分数，以避免过拟合。

### Skill 触发机制的工作原理

理解触发机制有助于设计更好的 eval 查询。Skill 会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 根据该描述决定是否调用某个 skill。需要知道的重要一点是，Claude 只会在无法轻松独立处理的任务中调用 skill——简单的单步查询（如"读取这个 PDF"）可能不会触发 skill，即使描述完全匹配，因为 Claude 可以用基础工具直接处理。当描述匹配时，复杂的、多步骤的或专业化的查询能可靠地触发 skill。

这意味着你的 eval 查询应该足够实质化，让 Claude 真正受益于调用 skill。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触发 skill。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description`，并更新 skill 的 SKILL.md frontmatter。向用户展示前后的对比，并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查你是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，请打包 skill 并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，将用户引导到生成的 `.skill` 文件路径，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，一些机制会有所不同。以下是需要适应的部分：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后按照其说明自己完成测试提示。逐个进行。这不如独立子代理那样严格（你编写了 skill 也在运行它，所以你有完整的上下文），但这是一个有用的完整性检查——人类审查步骤可以弥补这部分。跳过基线运行——直接使用 skill 完成请求的任务。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示，或者你在远程服务器上），完全跳过浏览器审查。直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告诉用户位置，以便他们下载和检查。在行内请求反馈："看起来怎么样？有什么想改的吗？"

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，请求反馈——只是中间没有浏览器审查。如果有文件系统，你仍然可以在文件系统中将结果组织到迭代目录中。

**描述优化**：本节需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果你在 Claude.ai 上，跳过这部分。

**盲对比**：需要子代理。跳过。

**打包**：`package_skill.py` 脚本可以在任何有 Python 和文件系统的环境中运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而非创建新 skill。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名称和 `name` frontmatter 字段——保持不变。例如，如果已安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而非 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先暂存到 `/tmp/`**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 专用说明

如果你在 Cowork 中，需要了解的主要内容有：

- 你有子代理，所以主要工作流程（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果你遇到严重的超时问题，串行运行测试提示是可以接受的。）
- 你没有浏览器或显示器，所以在生成评估查看器时，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 无论出于什么原因，Cowork 配置似乎会让 Claude 在运行测试后不愿意生成评估查看器，所以我要再次重申：无论你在 Cowork 还是 Claude Code 中，运行测试后都应该始终生成评估查看器供人类查看示例，然后再自己评估输入！请使用 `generate_review.py`（不要自己写花哨的 html 代码）。抱歉先说这么多，但我还是要大写强调：在评估输入之前先生成评估查看器！你想尽快让人类看到它们！
- 反馈机制不同：由于没有运行中的服务器，查看器的"提交所有审查"按钮会下载 `feedback.json` 作为文件。然后你可以从这里读取它（可能需要先请求访问权限）。
- 打包功能正常——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 并且用户同意它已经做好之后再进行。
- **更新现有 skill**：用户可能要求你更新现有 skill，而非创建新 skill。请按照上面 Claude.ai 部分中的更新指南操作。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本胜出

references/ 目录有额外的文档：

- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次重复核心循环以强调：

- 弄清楚 skill 是关于什么的
- 起草或编辑 skill
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终 skill 并返回给用户

如果你是用 TodoList 的，请将步骤添加到其中以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类审查测试用例"放入 TodoList 以确保它被执行。

祝你好运！