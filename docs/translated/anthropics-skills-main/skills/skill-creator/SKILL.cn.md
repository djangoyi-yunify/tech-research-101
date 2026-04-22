---
name: skill-creator
description: 创建新技能、修改和改进现有技能，并测量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、使用方差分析进行技能性能基准测试，或优化技能描述以提高触发准确性时使用。
---

# Skill Creator

用于创建新技能并迭代改进技能的能力。

从高层来看，创建技能的过程如下：

- 确定技能的功能和大致实现方式
- 编写技能初稿
- 创建一些测试提示词，并在支持技能访问的环境下运行
- 帮助用户定性和定量评估结果
  - 在后台运行期间，如果还没有定量评估，可以起草一些（如果已有，可以直接使用或根据需要修改）。然后向用户解释这些评估（或者如果已存在，解释现有的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（如果定量基准测试中发现明显缺陷，也应一并考虑）
- 重复直到满意
- 扩展测试集并在更大规模上再试

使用此技能时，你的工作是弄清楚用户处于哪个阶段，然后帮助他们推进这些阶段。比如用户可能说"我想创建一个用于 X 的技能"。你可以帮助明确他们的需求、编写初稿、编写测试用例、确定评估方式、运行所有提示词，并循环迭代。

另一方面，如果用户已经有一个技能初稿，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估，随便聊聊就行"，你也可以照做。

然后，在技能完成后（顺序同样可以灵活调整），你还可以运行技能描述优化器，我们有专门的脚本用于优化技能的触发机制。

明白了吗？明白了。

## 与用户沟通

技能创建者可能被对编码术语熟悉程度差异很大的用户使用。如果你没听说过（你怎么能听说呢，这是最近才兴起的趋势），现在有一种趋势是 Claude 的能力激励了管道工打开终端、父母和祖父母去搜索"如何安装 npm"。另一方面，大多数用户可能对计算机相当熟悉。

所以请注意上下文线索，了解如何措辞！举一个默认情况的例子：

- "evaluation"和"benchmark"边界模糊，但可以使用
- 对于"JSON"和"assertion"，你需要看到用户明显熟悉这些术语的迹象，才能不加解释地使用

如果有疑问，可以简要解释术语；如果不确定用户是否能理解，可以附带简短定义进行澄清。

---

## 创建技能

### 明确意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤序列、用户的纠正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前确认。

1. 这个技能应该让 Claude 能够做什么？
2. 技能应该在什么时候触发？（什么样的用户措辞/上下文）
3. 期望的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。输出主观的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项等问题。等到这部分完善后再编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有子代理可用则通过子代理并行研究，否则内联研究。准备好相关背景知识以减少用户负担。

### 编写 SKILL.md

基于用户访谈，填写以下组件：

- **name**: 技能标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包括技能的功能以及具体的使用场景。所有"何时使用"的信息都放在这里，而不是正文。注意：目前 Claude 有"触发不足"的倾向——即在有用的时候不使用技能。为了解决这个问题，请把技能描述写得稍微"强硬"一些。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，可以写成"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标，或想要显示任何类型的公司数据时，一定要使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能编写指南

#### 技能结构

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

#### 渐进式展示

Skill 使用三级加载系统：

1. **元数据**（名称 + 描述）- 始终在上下文中加载（~100 词）
2. **SKILL.md 主体** - 当 skill 触发时在上下文中加载（理想情况下 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可执行而不需要加载）

这些字数统计是近似值，必要时可以超出限制。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如果接近此限制，请添加额外的层级结构，并附带清晰的指引，说明使用该 skill 的模型接下来应去哪里跟进。
- 在 SKILL.md 中清晰地引用文件，并提供关于何时应读取它们的指导
- 对于大型参考文件（>300 行），请包含目录

**领域组织**：当一个 skill 支持多个领域/框架时，按变体组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅读取相关的参考文件。

### 不违背用户预期原则

这不用说，但技能不能包含恶意软件、利用代码或任何可能危害系统安全的内容。如果描述了技能的目的，其内容不应让用户感到意外。不要配合创建误导性技能或旨在促进未授权访问、数据泄露或其他恶意活动的技能。不过，像「扮演 XYZ」这类内容是可以的。

### 写作模式

在指令中优先使用祈使句。

**定义输出格式** - 你可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有帮助。你可以这样格式化它们（但如果示例中有"输入"和"输出"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物的重要性，而不是使用沉重的、刻板的 MUST。用心理理论（theory of mind）来思考，尝试让技能具有通用性，而不是过于局限于特定示例。先写一个初稿，然后用新的视角审视它并改进。

### 测试用例

写完技能初稿后，设计 2-3 个现实的测试提示——这些是真实用户可能会说的话。将它们分享给用户：[不一定使用完全相同的措辞] "这里有几个我想尝试的测试用例。看起来没问题吧，还是你想添加更多？" 然后运行它们。

将测试用例保存到 `evals/evals.json`。先不要写断言——只需要提示。在下一步中，当运行进行时，你再起草断言。

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

有关完整 schema（包括稍后需要添加的 `assertions` 字段），请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续的流程——不要中途停止。不要使用 `/skill-test` 或任何其他测试 skill。

将结果放在 `<skill-name>-workspace/` 中，作为 skill 目录的同级目录。在 workspace 中，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在该目录下，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要提前创建所有目录——边走边创建即可。

### 步骤 1：在同一轮中启动所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮中启动两个子 agent——一个有 skill，一个没有。这很重要：不要先启动 with-skill 运行，然后再回头做 baseline。将所有任务同时启动，以便它们大约同时完成。

**With-skill 运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同 prompt，但 baseline 取决于上下文）：

- **创建新 skill**：完全没有 skill。使用相同 prompt，不指定 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：旧版本。在编辑之前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline subagent 指向快照目录。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言可以暂时留空）。为每个 eval 提供一个描述性名称，基于其测试内容——不要简单地命名为 "eval-0"。同时用这个名称作为目录名。如果本次迭代使用了新的或修改过的 eval prompt，需要在每个新的 eval 目录中创建这些文件——不要假设它们会从之前的迭代继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### Step 2: 当运行进行时，起草断言

不要只是等待运行完成——你可以有效地利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言是客观可验证的，并且有描述性的名称——它们应该在 benchmark 查看器中清晰易读，这样人们只需扫一眼结果就能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的事物。

起草完成后，用断言更新 `eval_metadata.json` 文件和 `evals/evals.json`。还要向用户解释他们在查看器中会看到什么——包括定性输出和定量 benchmark。

### Step 3: 当运行完成时，捕获计时数据

当每个子代理任务完成时，你将收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它通过任务通知传递，不会持久化存储在其他地方。每个通知到达时立即处理，不要尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录下的 `grading.json`。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以编程检查的断言，编写并运行脚本而不是人工查看——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准中** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参阅 `references/schemas.md` 了解 viewer 期望的确切 schema。将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **进行分析师审查** — 阅读 benchmark 数据，发现汇总统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论是否有 skill 都总是通过的断言（非区分性）、高方差的评估（可能是 flaky）以及 time/token 的权衡。

4. **启动 viewer**，同时加载定性输出和定量数据。

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及以上迭代，还需要传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**无头环境（headless）或无图形界面的环境：** 如果 `webbrowser.open()` 不可用，或环境中没有显示器，可以使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈将以 `feedback.json` 文件的形式下载。下载后，将 `feedback.json` 复制到工作区目录，以便下次迭代时读取。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告诉用户**类似这样的内容："我已在浏览器中打开了结果。有两个标签页——'Outputs' 可让你逐个查看测试用例并留下反馈，'Benchmark' 显示定量比较结果。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（第 2 次及以上迭代）：折叠部分，显示上次迭代的输出
- **Formal Grades**（如果运行了 grading）：折叠部分，显示断言的通过/失败情况
- **Feedback**：用户输入时自动保存的文本框
- **Previous Feedback**（第 2 次及以上迭代）：用户在上一轮的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的通过率、耗时和 token 使用量，包含每次评估的细分数据和分析师观察。

导航可通过 prev/next 按钮或方向键。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

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

空白的反馈意味着用户认为没有问题。请将您的改进重点放在用户有具体投诉的测试用例上。

完成操作后，请关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 skill

这是循环的核心。你已经运行了测试用例，用户也审查了结果，现在你需要根据他们的反馈来改进这个 skill。

### 如何思考改进

1. **从反馈中进行泛化。** 这里发生的大局是：我们正在尝试创建可以被使用数百万次（也许是字面意义上的，甚至更多）的 skills，应用于各种不同的 prompt。你和用户在这里只是在少数几个例子上反复迭代，因为这样速度更快。用户对这些例子了如指掌，评估新输出也很快捷。但如果你和用户共同开发的 skill 只适用于这些例子，那它就毫无用处。与其进行过于精细的过拟合修改，或者设置限制性过强的 MUST 指令，如果有顽固的问题，你可以尝试拓展思路，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你能找到真正出色的方案。

2. **保持 prompt 精简。** 删除没有发挥作用的内容。确保阅读了 transcript，而不仅仅是最终输出——如果看起来 skill 让模型浪费了大量时间做没有产出价值的事情，你可以尝试删除导致这种行为的 skill 部分，看看会发生什么。

3. **解释为什么。** 尽力解释你要求模型执行每个操作的**原因**。当今的 LLMs 非常**智能**。它们有很好的心智理论，在得到良好引导的情况下，可以超越机械指令，真正把事情做成。即使用户的反馈简短或带有挫败感，也要努力真正理解任务、理解用户为什么写他们写的内容、理解他们实际写了什么，然后将这种理解传达给指令。如果你发现自己用大写字母写 ALWAYS 或 NEVER，或者使用过于僵化的结构，这是一个黄色警告信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求做的事情为什么重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的 transcript，注意子代理是否都独立编写了类似的辅助脚本或采用相同的多步骤方法处理某件事。如果 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈信号，表明 skill 应该捆绑该脚本。写一次，放到 `scripts/` 目录中，然后告诉 skill 使用它。这可以节省每次未来调用都要重新发明轮子的时间。

这个任务相当重要（我们正在尝试在这里创造每年数十亿美元的经济价值！）你的思考时间不是瓶颈；慢慢来，真正深入思考。我建议先写一份修订草案，然后重新审视它并进行改进。真正尽力代入用户的角色，理解他们想要什么、需要什么。

### 迭代循环

改进 skill 后：

1. 将你的改进应用到 skill
2. 将所有测试用例重新运行到新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建一个新 skill，基线始终是 `without_skill`（不使用 skill）——这在迭代过程中保持不变。如果你在改进现有 skill，使用你的判断来决定什么作为基线有意义：用户最初带来的原始版本，还是之前的迭代。
3. 使用 `--previous-workspace` 指向之前的迭代来启动 reviewer
4. 等待用户审查并告知你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户说他们满意了
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲测比较

在某些情况下，你需要对两个版本的 skill 进行更严格的比较（例如，用户问"新版本真的更好吗？"），可以使用盲测比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的代理，不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人类审查循环通常就足够了。

---

## 描述优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。在创建或改进 skill 后，建议优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I need some clarification before proceeding:

**Which skill should I create the eval set for?**

Please provide:
1. **Skill name** (e.g., "PDF Text Extraction", "CSV Data Analysis", etc.)
2. **Skill description** (what it does — the current draft if you have one)
3. **Use cases** (optional, but helpful context about what users typically ask for)

Once you share that, I'll:
- Generate realistic should-trigger/should-not-trigger queries
- Create the HTML review template
- Open it for your review and iteration

What's the skill?

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

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，将生成的 `.skill` 文件路径告知用户，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（草稿 → 测试 → 审查 → 改进 → 循环），但由于 Claude.ai 没有 subagent，部分机制会有所不同。以下是需要适配的内容：

**运行测试用例**：没有 subagent 意味着无法并行执行。对于每个测试用例，读取 skill 的 SKILL.md，然后按照其指示自己完成测试提示。逐一进行。这不如独立 subagent 那样严格（因为你既编写了 skill 又在运行它，完全了解上下文），但这是一个有用的检查——人工审查步骤可以弥补这一不足。跳过基线运行——直接使用 skill 完成请求的任务。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示，或者你在远程服务器上），则完全跳过浏览器审查。直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），则保存到文件系统并告知他们文件位置，以便下载检查。行内请求反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖于没有 subagent 就无法进行的基线比较。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill、重新运行测试用例、请求反馈——只是中间没有浏览器审查。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果你在 Claude.ai 上，跳过此步骤。

**盲测比较**：需要 subagent。跳过。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的环境中都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能是在要求你更新现有 skill，而不是创建新的。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名和 `name` frontmatter 字段——保持不变。例如，如果已安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前先复制到可写位置。** 已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 暂存**，然后复制到输出目录——直接写入可能因权限问题失败。

---

## Cowork 专用说明

在 Cowork 中，主要需要了解的是：

- 你有 subagent，所以主要工作流程（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果遇到严重的超时问题，串行运行测试提示是可以接受的。）
- 你没有浏览器或显示器，所以在生成 eval viewer 时，使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 由于某些原因，Cowork 配置似乎不鼓励 Claude 在运行测试后生成 eval viewer，所以再次重申：无论你在 Cowork 还是 Claude Code 中，运行测试后都应该始终生成 eval viewer，让人工在你自己评估输入之前查看示例。提前致歉，但我还是要大写：**在评估输入之前先生成 EVAL VIEWER***。你想尽快让人工看到它们！
- 反馈机制有所不同：由于没有运行中的服务器，viewer 的"Submit All Reviews"按钮会下载 `feedback.json` 作为文件。然后你可以从那里读取它（可能需要先请求访问权限）。
- 打包功能正常——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 subprocess 调用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 并且用户确认状态良好后再进行。
- **更新现有 skill**：用户可能是在要求你更新现有 skill，而不是创建新的。请按照上面 claude.ai 部分中的更新指南操作。

---

## 参考文件

agents/ 目录包含专门 subagent 的说明。在需要生成相关 subagent 时阅读它们。

- `agents/grader.md` — 如何对输出进行断言评估
- `agents/comparator.md` — 如何对两个输出进行盲测 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本为何胜出

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次强调核心循环：

- 明确 skill 的主题
- 起草或编辑 skill
- 运行能够访问 skill 的 claude 处理测试提示
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终 skill 并返回给用户

如果，你有 TodoList，请将步骤添加到其中以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"添加到 TodoList 中以确保它被执行。

祝你好运！