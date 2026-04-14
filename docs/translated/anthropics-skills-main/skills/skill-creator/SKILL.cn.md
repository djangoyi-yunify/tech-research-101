---
name: skill-creator
description: 创建新技能、修改和改进现有技能，并测量技能性能。当用户想要从零创建技能、编辑或优化现有技能、运行评估测试技能、使用方差分析对技能性能进行基准测试，或优化技能描述以提高触发准确性时使用此技能。
---

# 技能创建器

一个用于创建新技能并迭代改进它们的技能。

从高层次来看，创建技能的过程如下：

- 决定你想要技能做什么，以及大致如何实现
- 编写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户从定性和定量两个角度评估结果
  - 在运行进行的同时，如果没有定量评估，可以起草一些定量评估（如果已经有一些，你可以直接使用或根据需要进行修改）。然后向用户解释它们（或者如果已经存在，向用户解释现有的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（如果定量基准测试中有任何明显的缺陷，也要据此改进）
- 重复直到你满意为止
- 扩大测试集并尝试更大规模的测试

你使用此技能时的任务是弄清楚用户处于这个过程中的哪个阶段，然后帮助他们推进这些阶段。例如，也许他们说"我想制作一个用于 X 的技能"。你可以帮助他们明确需求，撰写初稿，编写测试用例，确定他们想要如何评估，运行所有提示词，并重复这个过程。

另一方面，也许他们已经有一个技能初稿了。在这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估，就这么来"，你也可以照做。

然后在技能完成后（但同样，顺序是灵活的），你也可以运行技能描述优化器，我们有一个单独的脚本用于此，以优化技能的触发。

好的？好的。

## 与用户沟通

技能创建器可能被各种熟悉程度不同的用户使用。如果你没听说过（你怎么可能听说过呢，毕竟这只是最近才开始的趋势），现在有一股潮流：Claude 的强大能力激励着水管工打开终端、父母和祖父母去搜索"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索来理解如何措辞！在默认情况下，给你一些参考：

- "evaluation"（评估）和"benchmark"（基准测试）处于边界，但可以使用
- 对于"JSON"和"assertion"（断言），你需要看到用户确实了解这些术语的明确线索，然后才能不解释就使用它们

如果有疑问，可以简要解释术语；如果不确定用户是否理解，可以简短地澄清定义。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前的对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是的话，首先从对话历史中提取答案——使用的工具、步骤序列、用户的纠正、观察到的输入/输出格式。用户可能需要填补空白，并且在继续下一步之前应该确认。

1. 这个技能应该让 Claude 能够做什么？
2. 这个技能应该在什么时候触发？（什么样的用户用语/上下文）
3. 期望的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。输出主观的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认值，但由用户决定。

### 访谈和研究

主动询问有关边缘情况、输入/输出格式、示例文件、成功标准和依赖项的问题。等到这部分完善后再编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有子代理可用则通过子代理并行研究，否则内联进行。做好准备以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发、做什么。这是主要的触发机制——包括技能的功能以及使用它的具体上下文。所有"何时使用"的信息都放在这里，不放在正文里。注：目前 Claude 有"触发不足"的倾向——在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"强硬"一点。例如，不要写"How to build a simple fast dashboard to display internal Anthropic data."（如何构建一个简单的快速仪表板来显示内部 Anthropic 数据），你可以写"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"（如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标，或想要显示任何类型的公司数据时，请确保使用此技能，即使他们没有明确要求"仪表板"。）
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能编写指南

#### 技能的组成结构

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

1. **元数据**（名称 + 描述）— 始终在上下文中加载（约 100 词）
2. **SKILL.md 主体**— 技能触发时在上下文中加载（理想情况不超过 500 行）
3. **捆绑资源**— 按需加载（无限制，脚本可在不加载的情况下执行）

这些字数统计仅为近似值，如有需要可以超出。

**关键模式：**

- 将 SKILL.md 控制在 500 行以内；如果接近此限制，请添加额外的层级结构，并提供清晰的指引，说明使用该技能的模型接下来应前往何处继续跟进。
- 在 SKILL.md 中清晰引用文件，并提供关于何时读取它们的指导
- 对于大型参考文件（超过 300 行），请包含目录

**领域组织**：当一个技能支持多个领域/框架时，按变体进行组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅读取相关的参考文件。

#### 避免意外原则

不言而喻，技能不得包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果技能有描述，其内容不应在意图上使用户感到意外。不要配合创建误导性技能或设计用于促进未授权访问、数据窃取或其他恶意活动的技能。不过，像“扮演 XYZ 进行角色扮演”这类是可以的。

#### 编写模式

在指令中优先使用祈使句。

**定义输出格式** - 你可以这样做：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 在文档中包含示例非常有用。你可以用这样的方式格式化它们（但如果示例中有"Input"和"Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为什么重要，而不是使用老旧的、命令式的 MUST。用好心智理论，尽量让技能更具通用性，而不是局限于特定案例。先写一份草稿，然后用新鲜的眼光审视并改进它。

### 测试用例

写完技能草稿后，设计 2-3 个真实的测试提示语——即真实用户可能会说的话类型。与用户分享：[不必使用完全相同的措辞] "这是我想尝试的几个测试用例。您觉得这些合适吗，或者想添加更多？" 然后运行它们。

将测试用例保存到 `evals/evals.json`。先不要编写断言——只需要提示语。你会在下一步运行过程中起草断言。

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

请参阅 `references/schemas.md` 获取完整 schema（包含 `assertions` 字段，你稍后会添加该字段）。

## 运行和评估测试用例

本节是一个连续的序列——请不要中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放在技能目录的同级目录 `<skill-name>-workspace/` 中。在 workspace 内，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在每个迭代下，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要一开始就创建所有目录——随着进行逐步创建即可。

### 步骤 1：在同一轮次中启动所有运行（有技能版本和无技能 baseline）

对于每个测试用例，在同一轮次中启动两个子代理——一个有技能，一个没有技能。这很重要：不要先启动有技能的运行，然后再回过头来运行 baseline。请一次性全部启动，这样它们都能在同一时间段完成。

**有技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同 prompt，但 baseline 取决于上下文）：

- **创建新技能**：完全没有技能。使用相同 prompt，无技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：使用旧版本。在编辑之前，先对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline subagent 指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言部分暂时留空即可）。为每个 eval 起一个描述性名称，基于其测试内容——不要简单地命名为 "eval-0"。同时用这个名称来命名目录。如果本次迭代使用了新的或修改过的 eval prompts，需为每个新 eval 目录创建这些文件——不要假设它们会从上一次迭代继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行期间起草断言

不要只是等待运行完成——你可以利用这段时间来提高效率。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查并解释它们检查的内容。

好的断言应该是客观可验证的，并具有描述性的名称——它们在基准查看器中应该清晰易读，这样查看结果的人能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的事物。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中将看到的内容——包括定性输出和定量基准。

### 步骤 3：运行完成后，捕获计时数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知获取，不会持久化到其他地方。收到每个通知时立即处理，不要尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行评分** — 生成一个 grader 子代理（或内联评分），读取 `agents/grader.md`，并对每个断言进行评估。将结果保存到每次运行目录中的 `grading.json`。grading.json 的期望数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖于这些确切的字段名。对于可以编程检查的断言，编写并运行脚本，而不是肉眼检查——脚本更快、更可靠，并且可以在迭代中复用。

2. **聚合到基准测试** — 运行 skill-creator 目录中的聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解 viewer 期望的确切 schema。

将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **进行分析师审查** — 阅读 benchmark 数据，发现汇总统计数据可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论技能如何都总是通过的断言（非区分性）、高方差的评估（可能不稳定）以及时间/token 权衡。

4. **启动 viewer**，同时加载定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于 iteration 2+，还要传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**Cowork / headless 环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会被下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录中，以便下一次迭代拾取。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告诉用户**类似这样的话："我已在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后，回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs"标签页一次显示一个测试用例：
- **Prompt**：给定的任务
- **Output**：技能产生的文件，尽可能以内联方式渲染
- **Previous Output**（iteration 2+）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：自动保存的文本框
- **Previous Feedback**（iteration 2+）：上次评论，显示在文本框下方

"Benchmark"标签页显示统计摘要：每个配置的正确率、时间和 token 使用量，以及每个评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

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

空反馈意味着用户认为可以接受。将改进重点放在用户有具体意见的测试用例上。完成操作后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是循环的核心部分。你已经运行了测试用例，用户也审查了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中归纳总结。** 这里的大局是：我们正在尝试创建可以重复使用数百万次（也许真的是数百万次，甚至更多）的技能，这些技能适用于各种不同的 prompt。你和用户之所以在少数几个例子上反复迭代，是因为这样可以加快速度。用户对这些例子了如指掌，能够快速评估新的输出。但如果你和用户共同开发的技能只能用于这些例子，那它就毫无用处。与其做出容易过拟合的繁琐改动，或者过于严格的 MUST 约束，不如尝试分支出去，使用不同的隐喻，或者推荐不同的工作模式。尝试的成本相对较低，也许你会发现真正出色的方案。

2. **保持 prompt 精简。** 删除那些没有发挥作用的内容。确保阅读了转录文本，而不仅仅是最终输出——如果看起来技能让模型浪费了大量时间去做一些没有成效的事情，你可以尝试删除导致这一问题的技能部分，然后观察结果。

3. **解释原因。** 努力解释你要求模型执行的每项操作背后的**原因**。当前的 LLM 非常智能，它们具有良好的心智理论，当给它们一个良好的框架时，能够超越死板的指令，真正把事情做好。即使用户的反馈简短或带有挫败感，也要努力真正理解任务，理解用户为什么写他们写的内容，理解他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己用全大写写 ALWAYS 或 NEVER，或者使用过于僵化的结构，这是一个黄色警告信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求做的事情为什么重要。这是一种更人性化、更有力、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意子代理是否都独立编写了类似的辅助脚本或对某些内容采用了相同的多步骤方法。如果 3 个测试用例的结果都是子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明技能应该打包这个脚本。写一次，放在 `scripts/` 目录中，然后告诉技能使用它。这样可以节省未来每次调用的重复开发工作。

这项任务非常重要（我们正在尝试在这里创造每年数十亿美元的经济价值！）你的思考时间不是瓶颈，慢慢来，真正深入思考。我建议先写一份修订草案，然后重新审视并改进。真正尽你所能进入用户的头脑，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将改进应用到技能中
2. 重新运行所有测试用例到新的 `iteration-<N+1>/` 目录，包括基线运行。如果你创建的是新技能，基线始终是 `without_skill`（不使用技能）——这在各次迭代中保持不变。如果你是在改进现有技能，使用你的判断来选择什么作为合理的基线：用户最初带来的原始版本，或者前一个迭代。
3. 使用 `--previous-workspace` 指向之前的迭代来启动审查器
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续下去直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲测比较

对于需要更严格比较技能两个版本的情况（例如，用户问"新版本真的更好吗？"），有一个盲测比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的代理，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md 头部信息中的 description 字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，提供优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I need more information to proceed. The process you've outlined requires:

1. **Skill name** — e.g., "PDF Text Extraction" or "CSV Data Analysis"
2. **Skill description** — the current description that would appear in the skill registry
3. **Evaluation data** — the JSON array of query examples (should-trigger and should-not-trigger)

Additionally, the HTML template at `assets/eval_review.html` needs to exist. Does that file currently exist in your workspace?

Once you provide these, I can:

- Generate realistic eval queries (8-10 should-trigger, 8-10 should-not-trigger)
- Populate the HTML review template with those queries
- Save it to `/tmp/eval_review_<skill-name>.html` and open it

What skill are we creating an eval set for?

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（驱动当前会话的模型），使触发测试与用户实际体验相匹配。

运行时，定期 tail 输出，向用户更新当前迭代次数和分数情况。

这将自动处理完整的优化循环。它将 eval 集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它在训练集和测试集上重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回带有 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### 技能触发机制的工作原理

了解触发机制有助于设计更好的 eval 查询。技能以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 根据该描述决定是否咨询某个技能。需要知道的重要一点是，Claude 只会在自己无法轻松处理的任务上咨询技能——简单的一步式查询（如"读取此 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理它们。复杂的、多步骤的或专门的查询在描述匹配时可靠地触发技能。

这意味着你的 eval 查询应该足够实质，以便 Claude 实际能从咨询技能中受益。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md frontmatter。向用户展示前后的对比，并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查你是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，打包技能并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，将生成的 `.skill` 文件路径告知用户，以便他们安装。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流程保持不变（起草 → 测试 → 评审 → 改进 → 循环），但由于 Claude.ai 没有子代理，某些机制会有所变化。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后按照其说明自己完成测试提示。逐个执行。这不如独立子代理那样严格（你编写了 skill，同时也在运行它，因此拥有完整的上下文），但这是一个有用的合理性检查——人类评审步骤可以弥补这一点。跳过基线运行——直接使用 skill 完成请求的任务。

**评审结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），则完全跳过浏览器评审。直接在对话中呈现结果。对于每个测试用例，显示提示词和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告知用户位置，以便他们下载和检查。inline 请求反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill、重新运行测试用例、请求反馈——只是中间没有浏览器评审。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，跳过此部分。

**盲测比较**：需要子代理。跳过。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的地方都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而非创建新 skill。这种情况下：
- **保留原始名称。** 注意 skill 的目录名称和 `name` frontmatter 字段——保持不变。例如，如果安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而非 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 暂存**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 特定说明

如果你在 Cowork 中，主要需要了解的是：

- 你有子代理，因此主要工作流程（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果遇到严重的超时问题，按顺序运行测试提示词也是可以的。）
- 你没有浏览器或显示器，因此在生成 eval viewer 时，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。然后提供一个用户可以在浏览器中打开的 HTML 链接。
- 由于某种原因，Cowork 设置似乎会让 Claude 在运行测试后不愿意生成 eval viewer，所以我要重申：无论你在 Cowork 还是 Claude Code 中，运行测试后都应该始终生成 eval viewer 让人类查看示例，然后才由自己修订 skill 并尝试修正，使用 `generate_review.py`（而不是自己写花哨的 HTML 代码）。提前道歉，但我会用全大写：**在评估输入之前先生成 EVAL VIEWER！** 你想尽快让人看到它们！
- 反馈的工作方式不同：由于没有运行中的服务器，viewer's "Submit All Reviews" 按钮会将 `feedback.json` 下载为文件。然后你可以从这里读取它（可能需要先请求访问权限）。
- 打包应该可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 subprocess 调用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 并且用户同意它状态良好后再进行。
- **更新现有 skill**：用户可能要求你更新现有 skill，而非创建新 skill。按照上面 claude.ai 部分的更新指南操作。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何对两个输出进行盲测 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本为何胜出

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再强调一次核心循环：

- 了解 skill 是关于什么的
- 起草或编辑 skill
- 运行可以访问 skill 的 claude 来处理测试提示词
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户评审它们
  - 运行定量评估
- 重复直到你和你满意
- 打包最终 skill 并返回给用户

请将这些步骤添加到你的 TodoList 中，以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类评审测试用例"添加到 TodoList 中以确保它被执行。

祝你好运！