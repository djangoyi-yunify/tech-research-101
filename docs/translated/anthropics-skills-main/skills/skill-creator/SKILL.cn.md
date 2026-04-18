---
name: skill-creator
description: 创建新技能、修改和完善现有技能，以及衡量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、对技能性能进行方差分析基准测试，或优化技能描述以提高触发准确性时使用。
---

# 技能创建器

用于创建新技能并对其进行迭代改进的技能。

从高层次来看，创建技能的过程如下：

- 确定技能的目标功能以及大致的实现方式
- 编写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行
- 帮助用户对结果进行定性和定量评估
  - 在运行于后台进行时，如果没有定量评估则起草一些（如果已有，可以直接使用或根据需要进行修改）。然后向用户解释它们（或者如果已存在，解释现有的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（以及如果定量基准测试中发现任何明显的缺陷）
- 重复直到满意为止
- 扩大测试集并在更大规模上再次尝试

使用此技能时，你的工作是弄清楚用户处于哪个阶段，然后帮助他们推进这些阶段。例如，用户可能说"我想制作一个用于 X 的技能"。你可以帮助他们明确需求、编写草稿、编写测试用例、确定评估方式、运行所有提示词，并重复这个过程。

另一方面，如果用户已经有技能草稿，你可以直接进入评估/迭代环节。

当然，你应该始终保持灵活，如果用户说"我不需要运行一堆评估，陪我就行"，那就照做。

然后在技能完成后（同样，顺序可以灵活调整），你还可以运行技能描述优化器，我们有专门的脚本来做这个，以优化技能的触发。

明白了吗？明白。

## 与用户沟通

技能创建器可能被各种熟悉程度不同的用户使用。如果你不了解这个趋势（你怎么能知道呢，它才刚刚开始流行），现在有一种趋势：克劳德的力量正在激励管道工打开终端，家长和祖父母们开始搜索"如何安装 npm"。另一方面，大多数用户可能具备相当的计算机素养。

所以请注意上下文线索，了解如何措辞！在默认情况下，给你一些参考：

- "evaluation"（评估）和 "benchmark"（基准测试）处于边界，但可以使用
- 对于 "JSON" 和 "assertion"（断言），你需要看到用户明显表现出了解这些术语的迹象后才能不加解释地使用

如果你有疑问，可以简要解释术语；如果不确定用户是否理解，也可以用简短的定义来澄清。

---

## 创建技能

### 捕捉意图

首先理解用户的意图。当前对话中可能已经包含了用户想要捕捉的工作流程（例如，他们说"把这个变成技能"）。如果是这样，先从对话历史中提取答案——使用的工具、步骤顺序、用户的修正、观察到的输入/输出格式。用户可能需要填补空白，并且在进入下一步之前应确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（用户的哪些表述/上下文）
3. 期望的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在这些问题解决之前，先不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果有子代理则通过子代理并行研究，否则内联研究。带着上下文做好准备，以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发，做什么。这是主要的触发机制——包括技能的功能和具体的使用场景。所有"何时使用"的信息都放在这里，不要放在正文里。注：目前 Claude 有"触发不足"的倾向——在技能有用时不使用它们。为了解决这个问题，请把技能描述写得稍微"强势"一些。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。每当用户提到仪表板、数据可视化、内部指标，或想要显示任何类型的公司数据时，一定要使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其他部分 :)**

### 技能编写指南

#### 技能的结构

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

1. **元数据**（名称 + 描述）- 始终在上下文中（~100 字）
2. **SKILL.md 主体** - 技能触发时在上下文中（理想情况 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可以在不加载的情况下执行）

这些字数统计是近似值，如果需要可以自由增加。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如果接近此限制，请添加额外的层级结构，并明确指示使用该技能的模型下一步应该去哪里跟进。
- 从 SKILL.md 中清晰地引用文件，并提供关于何时阅读它们的指导
- 对于大型引用文件（>300 行），包含目录

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

#### 避免意外原则

这不言而喻，但技能必须不包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。技能的内容如果被描述，不应该在用户意图上令人意外。不要配合创建误导性技能或旨在促进未授权访问、数据泄露或其他恶意活动的技能。不过，像“扮演 XYZ”这类的事情是可以的。

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

**示例模式** - 加入示例很有帮助。你可以这样格式化它们（但如果示例中包含"Input"和"Output"，可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### Writing Style

尝试向模型解释为什么要这样做，而不是使用过于强硬的、命令式的 Must 语句。运用换位思考，让技能具有通用性，而不是局限于特定示例。先写一份草稿，然后用新的视角审视并改进。

### Test Cases

写完技能草稿后，设计 2-3 个贴近实际的测试提示词 — 就像真实用户会说的话那样。将这些与用户分享：[无需使用完全相同的措辞] "这是我想尝试的几个测试用例。这些看起来合适吗，或者你想补充更多？" 将测试用例保存到 `evals/evals.json`。暂时不要写断言 — 只需要提示词即可。在下一步测试运行期间起草断言。

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

See `references/schemas.md` for the full schema (including the `assertions` field, which you'll add later).

完整的 schema 定义请参见 `references/schemas.md`（包括 `assertions` 字段，该字段稍后添加）。

## Running and evaluating test cases

## 运行和评估测试用例

This section is one continuous sequence — don't stop partway through. Do NOT use `/skill-test` or any other testing skill.

本节是一个连续流程——不要半途而止。请勿使用 `/skill-test` 或任何其他测试 skill。

Put results in `<skill-name>-workspace/` as a sibling to the skill directory. Within the workspace, organize results by iteration (`iteration-1/`, `iteration-2/`, etc.) and within that, each test case gets a directory (`eval-0/`, `eval-1/`, etc.). Don't create all of this upfront — just create directories as you go.

将结果放在 skill 目录的同级目录 `<skill-name>-workspace/` 中。在 workspace 内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代目录内，每个测试用例各有一个目录（`eval-0/`、`eval-1/` 等）。不要一开始就创建所有目录——随着进度逐步创建即可。

### Step 1: Spawn all runs (with-skill AND baseline) in the same turn

### 步骤 1：在同一轮中启动所有运行（带 skill 和基线）

For each test case, spawn two subagents in the same turn — one with the skill, one without. This is important: don't spawn the with-skill runs first and then come back for baselines later. Launch everything at once so it all finishes around the same time.

对于每个测试用例，在同一轮中启动两个 subagent——一个带 skill，一个不带。这很重要：不要先启动带 skill 的运行，稍后再回头做基线。一开始就同时启动所有运行，使其大致在同一时间完成。

**With-skill run:**

**带 skill 的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同的 prompt，但 baseline 取决于上下文）：

- **创建新 skill**：完全没有 skill。使用相同 prompt，不指定 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：旧版本。在编辑之前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline subagent 指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言可以暂时留空）。根据测试内容为每个 eval 赋予描述性名称——不要只用 "eval-0"。目录名也使用这个名称。如果本次迭代使用了新的或修改过的 eval prompt，需要为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第 2 步：在运行进行时起草断言

不要只是等待运行完成——你可以有效地利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且有描述性的名称——它们应该在 benchmark 查看器中清晰可读，这样查看结果的人能立即理解每个断言检查的内容。主观技能（如写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的事物。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量 benchmark。

### 第 3 步：当运行完成时，捕获计时数据

当每个 subagent 任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录的 `timing.json` 文件中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知传递，不会持久化存储在其他地方。每当收到通知时就立即处理，不要尝试批量处理。

### 第 4 步：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分** — 生成一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名称。对于可以编程检查的断言，编写并运行脚本而不是肉眼观察——脚本更快、更可靠，并且可以在迭代中复用。

2. **聚合到基准测试** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

This produces `benchmark.json` and `benchmark.md` with pass_rate, time, and tokens for each configuration, with mean ± stddev and the delta. If generating benchmark.json manually, see `references/schemas.md` for the exact schema the viewer expects.
Put each with_skill version before its baseline counterpart.

3. **Do an analyst pass** — read the benchmark data and surface patterns the aggregate stats might hide. See `agents/analyzer.md` (the "Analyzing Benchmark Results" section) for what to look for — things like assertions that always pass regardless of skill (non-discriminating), high-variance evals (possibly flaky), and time/token tradeoffs.

4. **Launch the viewer** with both qualitative outputs and quantitative data:

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置项的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参阅 `references/schemas.md` 以了解 viewer 所需的确切 schema。
请将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **执行分析员审查** — 阅读基准测试数据，揭示汇总统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results" 部分）了解需要关注的内容——例如无论是否使用 skill 都总是通过的断言（非区分性）、高方差的评估（可能是 flaky），以及 time/token 权衡。

4. **启动 viewer**，同时传入定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及以上的迭代，还需要传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。当用户点击 "Submit All Reviews" 时，反馈会下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下一次迭代使用。

注意：请使用 `generate_review.py` 来创建查看器，无需编写自定义 HTML。

5. **告知用户** 类似："已在浏览器中打开结果。有两个标签页——'Outputs' 允许你逐个查看测试用例并留下反馈，'Benchmark' 显示定量比较。完成后回到这里告诉我。"

### 查看器中用户看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（第 2 次及以上迭代）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：自动保存的文本框
- **Previous Feedback**（第 2 次及以上迭代）：上次迭代的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置文件的通过率、耗时和 token 使用情况，包含每个评估的详细分析和分析师观察。

导航通过上一个/下一个按钮或方向键完成。完成后，用户点击 "Submit All Reviews"，所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

当用户告知完成时，读取 `feedback.json`：

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

空反馈表示用户认为没问题。将改进重点放在用户有具体反馈意见的测试用例上。

完成之后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 提升技能

这是循环的核心部分。你已经运行了测试用例，用户也审查了结果，现在需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中总结规律。** 这里的宏观目标是创建能够被使用数百万次（可能是字面意思，甚至更多）的技能，应用于各种不同的 prompt。你和用户在这里反复迭代少数几个例子是因为这样速度更快。用户对这些例子了如指掌，能够快速评估新的输出。但如果你和用户共同开发的技能只适用于这些例子，那它就毫无用处。与其进行繁琐的过拟合修改，或者施加过于严格的 MUST 限制，如果某个问题很棘手，你可以尝试拓展思路，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，说不定就能找到绝妙的解决方案。

2. **保持 prompt 精简。** 删除没有发挥作用的内容。不要只看最终输出，要仔细阅读 transcript——如果看起来技能在让模型浪费大量时间做低效的事情，你可以尝试删除导致这种行为的部分，然后观察结果。

3. **解释原因。** 努力解释你要求模型做的每件事背后的 **原因**。现在的 LLM 非常聪明，它们有很好的心智理论，给定一个好的框架后，能够超越死板的指令，真正发挥作用。即使用户的反馈很简短或带有情绪，也要真正理解任务本身，理解用户为什么这样写，理解他们实际写了什么，然后将这种理解传达给指令。如果你发现自己用全大写写 ALWAYS 或 NEVER，或者使用过于死板的结构，这是一个黄色警示信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求的事情为什么重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的 transcript，注意子代理是否独立编写了相似的辅助脚本或采用相同的多步骤方法。如果 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈信号，表明技能应该打包这个脚本。写一次，放在 `scripts/` 中，并告诉技能使用它。这可以节省每次调用时重新发明轮子的成本。

这个任务相当重要（我们正在尝试创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，真正深入思考。我建议先写一份修订草案，然后重新审视并改进。真正努力进入用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能
2. 将所有测试用例重新运行到新的 `iteration-<N+1>/` 目录，包括 baseline 运行。如果你创建的是新技能，baseline 始终是 `without_skill`（无技能）——它在迭代过程中保持不变。如果你是在改进现有技能，使用你的判断力来确定什么作为 baseline 最有意义：用户最初带来的原始版本，还是之前的迭代版本。
3. 使用 `--previous-workspace` 指向之前的迭代目录来启动 reviewer
4. 等待用户审查并告知你完成
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 所有反馈都是空的（一切都看起来很好）
- 你没有取得有意义的进展

---

## 高级：盲测对比

当你想要对技能的两个版本进行更严格的比较时（例如用户问"新版本真的更好吗？"），可以使用盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的代理，不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，建议优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

# Clarification Needed

I need to know which skill you're building evaluation queries for before I can proceed.

Please provide:

1. **Skill name** — What is this skill called?
2. **Skill description** — What does the skill do? (Current or draft version)
3. **File types / tools involved** — What inputs does it work with? (e.g., XLSX, JSON, PDFs, images)
4. **Primary use case** — What task is the user trying to accomplish?

Once I have that context, I can generate the 16-20 evaluation queries (8-10 should-trigger, 8-10 should-not-trigger) following the guidelines you outlined — realistic queries with personal context, specific details, edge cases, and tricky near-misses.

**Example context I'd need:**
> *"A skill that extracts structured data from PDFs — the user uploads a PDF (invoices, contracts, reports) and gets extracted data in JSON/CSV format."*

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（驱动当前会话的模型），使触发测试与用户实际体验相匹配。

运行时，定期监控输出，向用户更新当前迭代次数和分数情况。

这自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它在训练集和测试集上重新评估每个新描述，迭代最多 5 次。完成后，在浏览器中打开 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——选择依据是测试分数而非训练分数，以避免过拟合。

### 技能触发机制

理解触发机制有助于设计更好的评估查询。技能以其名称 + 描述出现在 Claude 的 `available_skills` 列表中，Claude 根据描述决定是否调用某个技能。重要的是，Claude 只会在自己难以独立处理的任务中调用技能——简单、单步的查询（如"读取这个 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理它们。复杂的、多步骤的或专门的查询，在描述匹配时会可靠地触发技能。

这意味着评估查询应该足够实质，让 Claude 真正受益于调用技能。像"读取文件 X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，更新技能的 SKILL.md frontmatter。向用户展示前后的对比，并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查是否可以使用 `present_files` 工具。如果不能，跳过此步骤。如果可以，打包技能并将 .skill 文件展示给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，某些机制会有所不同。以下是需要适应的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，阅读技能的 SKILL.md，然后按照其说明自己完成测试提示。逐个进行。这不如独立子代理那样严格（你编写了技能同时也在运行它，因此你有完整的上下文），但这是一个有用的健全性检查——而人工审查步骤可以弥补这一点。跳过基线运行——只需使用技能完成任务即可。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示，或你在远程服务器上），则完全跳过浏览器审查器。直接在对话中呈现结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），请保存到文件系统并告知他们位置以便下载和检查。inline 询问反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖于没有子代理就没有意义的基线比较。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，征求反馈——只是中间没有浏览器审查器。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：本节需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果你在 Claude.ai 上，请跳过。

**盲比较**：需要子代理。跳过。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的地方都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 .skill 文件。

**更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。在这种情况下：

- **保留原始名称。** 记下技能的目录名称和 `name` frontmatter 字段——原样使用。例如，如果安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑之前复制到可写位置。** 安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先暂存到 `/tmp/`**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 特定说明

如果你在 Cowork 中，主要需要了解的是：

- 你有子代理，因此主要工作流程（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示是可以的。）
- 你没有浏览器或显示器，因此在生成 eval 查看器时，使用 `--static <output_path>` 编写独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 出于某种原因，Cowork 设置似乎不倾向于让 Claude 在运行测试后生成 eval 查看器，所以再次强调：无论你在 Cowork 还是 Claude Code 中，运行测试后都应该始终生成 eval 查看器供人类查看示例，然后再自己修改技能并尝试进行更正，使用 `generate_review.py`（而不是编写自己的定制 html 代码）。提前道歉，但我会用大写字母：**在评估输入之前生成 EVAL 查看器！** 你希望尽快将它们呈现在人类面前！
- 反馈的工作方式不同：由于没有运行中的服务器，查看器的"提交所有审查"按钮会将 `feedback.json` 下载为文件。然后你可以从那里读取它（你可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过子进程使用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并获得用户同意后再进行。
- **更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。按照上面 claude.ai 部分中的更新指南操作。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本击败了另一个

references/ 目录有额外的文档：

- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次重复核心循环以强调：

- 确定技能是关于什么的
- 起草或编辑技能
- 运行能够访问技能的 claude 进行测试提示
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终技能并返回给用户。

请将步骤添加到你的待办事项列表中（如果你有的话），以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"放入你的待办事项列表以确保它会发生。

祝你好运！