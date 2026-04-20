---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及衡量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、使用方差分析进行技能性能基准测试，或优化技能描述以获得更好的触发准确性时使用此技能。
---

# Skill Creator

用于创建新技能并进行迭代改进的技能。

从高层来看，创建技能的过程如下：

- 确定你想要技能做什么以及大致如何实现
- 编写技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户对结果进行定性和定量评估
  - 在运行在后台进行的同时，如果没有定量评估的话，起草一些定量评估（如果有的话，你可以直接使用或根据需要对评估进行修改）。然后向用户解释这些评估（或者如果评估已经存在，向用户解释已有的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈（以及从定量基准测试中发现的任何明显缺陷）重写技能
- 重复直到你满意为止
- 扩展测试集并尝试更大规模的运行

使用此技能时，你的工作是弄清楚用户处于这个过程中的哪个阶段，然后介入并帮助他们完成这些阶段。例如，也许他们会说"我想为 X 创建一个技能"。你可以帮助他们明确需求，编写初稿，编写测试用例，确定他们想要的评估方式，运行所有提示词，并重复这个过程。

另一方面，也许他们已经有一个技能初稿。在这种情况下，你可以直接进入评估/迭代循环部分。

当然，你应该始终保持灵活，如果用户说"我不需要运行一堆评估，只是随便聊聊"，你也可以那样做。

然后在技能完成后（但同样，顺序是灵活的），你也可以运行技能描述优化器，我们有单独的脚本来做这件事，以优化技能的触发机制。

明白了吗？明白了。

## 与用户沟通

Skill Creator 可能会被各种熟悉代码术语程度不同的用户使用。如果你没有听说过（你怎么能听说过呢，因为它才刚刚开始流行），现在有一种趋势：Claude 的强大能力激励着管道工们打开终端、父母和祖父母们去搜索"如何安装 npm"。另一方面，大多数用户可能具有相当的计算机素养。

所以请注意上下文线索来理解如何措辞你的沟通！在默认情况下，给你一些参考：

- "evaluation"和"benchmark"是边界情况，但可以接受
- 对于"JSON"和"assertion"，你需要看到用户明确表示他们知道这些东西是什么，才可以在不解释的情况下使用它们

如果你有疑问，可以简要解释术语；如果不确定用户是否能理解，可以用一个简短的定义来澄清术语。

---

## 创建技能

### 明确意图

首先理解用户的意图。当前对话可能已经包含了用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是的话，首先从对话历史中提取答案——使用的工具、步骤序列、用户做出的更正、观察到的输入/输出格式。用户可能需要填补空白，并且应该在进入下一步之前进行确认。

1. 这个技能应该让 Claude 能够做什么？
2. 这个技能应该在什么时候触发？（什么用户短语/上下文）
3. 期望的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在完善这部分之前先不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有的话通过 subagents 并行研究，否则进行内联研究。带着上下文做好准备，以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发、做什么。这是主要的触发机制——包括技能做什么以及具体的使用上下文。所有"何时使用"的信息都放在这里，而不是在正文中。注意：目前 Claude 有"触发不足"的倾向——即在技能有用时不使用它们。为了解决这个问题，请让技能描述稍微"激进"一点。例如，不要写"如何构建一个简单的快速仪表盘来显示 Anthropic 内部数据"，你可以写"如何构建一个简单的快速仪表盘来显示 Anthropic 内部数据。每当用户提到仪表盘、数据可视化、内部指标，或想要显示任何类型的公司数据时，一定要使用此技能，即使他们没有明确要求'仪表盘'。"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)

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

Skills 使用三级加载系统：

1. **元数据**（名称 + 描述）- 始终在上下文中显示（约 100 词）
2. **SKILL.md 主体** - 当 skill 触发时在上下文中显示（理想 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

这些字数统计是近似值，如有需要可以超出限制。

**关键模式：**

- SKILL.md 保持在 500 行以内；如果接近此限制，请添加额外的层级结构，并明确指示使用该 skill 的模型接下来应去哪里继续跟进。
- 从 SKILL.md 中清晰引用文件，并提供何时应读取它们的指导
- 对于大型参考文件（>300 行），请包含目录

**领域组织**：当 skill 支持多个领域/框架时，按变体进行组织：

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

这不用说，但 Skill 不得包含恶意软件、利用代码或任何可能危害系统安全的内容。Skill 的内容如果被描述，不应在意图上让用户感到意外。不要响应创建误导性 Skill 或旨在促进未授权访问、数据泄露或其他恶意活动的请求。不过，像“扮演 XYZ 角色”这样的内容是可以的。

#### 写作模式

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

**示例模式** - 包含示例很有用。你可以这样格式化它们（但如果示例中包含"Input"和"Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事情的重要性，而不是用生硬死板的"MUST"（必须）指令。运用心智理论，努力让技能具有通用性，而不是局限于特定示例。先写一份草稿，然后用新的视角审视它并加以改进。

### 测试用例

写完技能草案后，想出 2-3 个贴近实际的测试提示——真正用户可能会说的话。将它们分享给用户：[不一定要用完全相同的措辞] "这是我想尝试的几个测试用例。看起来没问题，还是你想添加更多？" 然后运行它们。

将测试用例保存到 `evals/evals.json`。先不要写断言——只有提示。你会在下一步（运行进行中时）起草断言。

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

完整的 schema（包括后续需要添加的 `assertions` 字段）请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续流程——不要中途停止。不要使用 `/skill-test` 或任何其他测试 skill。

将结果放在 `<skill-name>-workspace/` 中，作为 skill 目录的同级目录。在 workspace 内，按迭代组织结果（`iteration-1/`、`iteration-2/` 等），在每个迭代内，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要提前创建所有这些目录——随着进度逐步创建即可。

### 步骤 1：在同一轮中启动所有运行（包括 with-skill 和 baseline）

对于每个测试用例，在同一轮中启动两个 subagent——一个带有 skill，一个不带。这很重要：不要先启动 with-skill 运行，稍后再回来做 baseline。要同时启动所有任务，确保它们几乎同时完成。

**With-skill 运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基准测试运行**（相同 prompt，但基准依赖上下文）：

- **创建新技能**：完全没有技能。相同 prompt，无技能路径，保存至 `without_skill/outputs/`。
- **改进现有技能**：旧版本。编辑前，对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让基准子代理指向该快照。保存至 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言目前可以为空）。为每个评测使用描述性名称——不要只是"eval-0"。目录名也使用该名称。如果本次迭代使用了新的或修改过的评测 prompt，需要为每个新评测目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第2步：在运行进行时，起草断言

不要只是等待运行完成——你可以高效利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言是客观可验证的，并且具有描述性名称——它们应该在基准查看器中清晰易读，这样查看结果的人能够立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人类判断的事物上。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。还要向用户解释他们在查看器中会看到什么——包括定性输出和定量基准。

### 第3步：运行完成时，捕获时序数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它通过任务通知传递，不会被持久化到其他地方。请在每条通知到达时实时处理，而不是尝试批量处理。

### 步骤 4：评分、聚合和启动查看器

所有运行完成后：

1. **对每次运行进行评分** — 启动一个 grader 子代理（或内联评分），读取 `agents/grader.md` 并根据输出对每个断言进行评估。将结果保存到每个运行目录下的 `grading.json` 文件中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以编程检查的断言，编写并运行脚本而不是肉眼观察——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **聚合到基准测试中** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参阅 `references/schemas.md` 获取 viewer 期望的精确 schema。

将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **执行分析师审查** — 阅读 benchmark 数据，发现聚合统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results" 部分）了解需要关注的内容 — 例如无论是否使用 skill 都总是通过的断言（非区分性）、高方差的评估（可能是 flaky）以及 time/token 的权衡。

4. **启动 viewer**，同时加载定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第二次及以后的迭代，还需要传递 `--previous-workspace <workspace>/iteration-<N-1>` 参数。

**Cowork / headless 环境：** 如果 `webbrowser.open()` 不可用，或环境中没有显示器，请使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。用户点击 "Submit All Reviews" 后，反馈将下载为 `feedback.json` 文件。下载后，请将 `feedback.json` 复制到工作区目录，以便下次迭代使用。

注意：请使用 `generate_review.py` 来生成查看器，无需编写自定义 HTML。

5. **提示用户** 例如："已在浏览器中打开结果。页面有两个标签页——'Outputs' 可逐个查看测试用例并留下反馈，'Benchmark' 显示量化对比。完成后返回这里告知我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能内联渲染
- **Previous Output**（第二次及以后的迭代）：折叠部分，显示上次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠部分，显示断言通过/失败情况
- **Feedback**：文本框，输入时自动保存
- **Previous Feedback**（第二次及以后的迭代）：用户上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的正确率、耗时和 token 使用量，包括按评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键操作。完成后，点击 "Submit All Reviews" 将所有反馈保存到 `feedback.json`。

### Step 5: 读取反馈

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

空白的反馈意味着用户认为没问题。将改进重点放在用户有具体抱怨的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 提升技能

这是整个循环的核心环节。你已经运行了测试用例，用户也审阅了结果，现在需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中提炼普适规律。** 这里最重要的大局观是：我们正在尝试创建能够被使用数百万次（也许字面意思上甚至更多）的技能，跨越各种不同的 prompt。你和用户之所以在少数几个例子上反复迭代，是因为这样速度更快。用户对这些例子了如指掌，能够快速评估新输出。但如果你和用户协作开发的技能只能在这些例子上奏效，那它就毫无用处。与其做出容易过拟合的微小调整，或设置过于严苛的 MUST 约束，不如尝试拓展思路——使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许就能找到出色的解决方案。

2. **保持 prompt 精简。** 删除那些没有发挥作用的内容。除了查看最终输出，还要仔细阅读转录文本——如果看起来技能让模型浪费大量时间在做无意义的事情上，可以尝试删除导致这种行为的技能部分，然后观察效果。

3. **解释原因。** 尽量为每一条要求解释**为什么**要这样做。当今的 LLM 非常聪明，它们有良好的心智理论，如果给它们一个好的框架，就能超越机械执行指令，真正把事情做好。即使用户反馈很简短或带有情绪，也要努力真正理解任务本身，理解用户为什么这样写，理解他们实际写的内容，然后将这种理解融入到指令中。如果你发现自己写了很多全大写的 ALWAYS 或 NEVER，或使用过于僵化的结构，这是一个黄色警示信号——如果可能的话，重新措辞，解释背后的推理，让模型理解你要求做的事情为什么重要。这是一种更人性化、更有力、更有效的方法。

4. **寻找测试用例间的重复工作。** 阅读测试运行的转录文本，注意是否所有 subagent 都独立编写了相似的辅助脚本，或采用相同的多步骤方法来处理某些事情。如果 3 个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，说明技能应该打包这个脚本。只写一次，放到 `scripts/` 目录，然后告诉技能使用它。这样可以节省未来每一次调用的重复工作。

这个任务相当重要（我们正在尝试创造每年数十亿美元的经济价值！）你的思考时间不是瓶颈；慢慢来，真正深入思考。我的建议是写一份修订草稿，然后重新审视它并做出改进。真正尽最大努力走进用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建一个新技能，基线始终是 `without_skill`（无技能）——这在迭代过程中保持不变。如果你在改进现有技能，使用你的判断力来确定什么作为基线有意义：用户最初带来的原始版本，还是上一个迭代版本。
3. 使用 `--previous-workspace` 指向上一个迭代目录来启动 reviewer
4. 等待用户审阅并告知完成
5. 阅读新的反馈，再次改进，重复

持续进行直到：

- 用户表示满意
- 所有反馈都是空的（一切都看起来很好）
- 你没有取得有意义的进展

---

## 高级：盲测对比

在某些情况下，你需要更严格地比较两个版本的技能（例如用户问"新版本真的更好吗？"），可以使用盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的 agent，但不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要 subagent，大多数用户不需要。人工审阅循环通常就足够了。

---

## 描述优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，建议优化描述以获得更好的触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的类型。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

# 评估集创建指南

---

查询必须是真实的、Claude Code 或 Claude.ai 用户实际会输入的内容。不是抽象请求，而是具体且细节丰富的请求。例如：文件路径、用户工作或情况的个人背景、列名和值、公司名称、URL。提供一些背景故事。部分查询可能使用小写、包含缩写、拼写错误或口语表达。使用不同长度的混合，专注于边缘情况而非简单明了的案例（用户将有机会审核确认）。

**反面示例：** `"Format this data"`、`"Extract text from PDF"`、`"Create a chart"`

**正面示例：** `"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

对于 **should-trigger** 查询（8-10条），考虑覆盖范围。需要同一意图的不同表述——一些正式，一些口语化。包含用户未明确提及技能或文件类型但明显需要的情况。加入一些不常见的用例，以及该技能与其他技能竞争但应胜出的情况。

对于 **should-not-trigger** 查询（8-10条），最有价值的是近似命中——查询与技能共享关键词或概念，但实际需要不同的东西。考虑相邻领域、歧义措辞（朴素关键词匹配会触发但不应该），以及查询涉及技能功能但在另一种工具更合适的上下文中的情况。

关键避免点：不要让 should-not-trigger 查询明显无关。"Write a fibonacci function" 作为 PDF 技能的负面测试太简单——不测试任何东西。负面案例应该真正具有挑战性。

### 步骤 2：与用户审核

使用 HTML 模板向用户展示评估集进行审核：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → eval 项目的 JSON 数组（不添加引号——这是一个 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → 技能名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → 技能的当前描述
3. 写入临时文件（例如 `/tmp/eval_review_<skill-name>.html`）并打开：`open /tmp/eval_review_<skill-name>.html`
4. 用户可以编辑查询、切换 should-trigger、添加/删除条目，然后点击"Export Eval Set"
5. 文件下载到 `~/Downloads/eval_set.json`——检查下载文件夹以获取最新版本以防有多个（例如 `eval_set (1).json`）

这一步很重要——不好的 eval 查询会导致糟糕的描述。

### 步骤 3：运行优化循环

告诉用户："这需要一些时间——我将在后台运行优化循环并定期检查。"

将评估集保存到工作区，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型ID（驱动当前会话的模型），使触发测试与用户的实际体验相匹配。

运行期间，定期查看输出，向用户更新当前迭代次数和分数情况。

这会自动处理整个优化循环。它将评估集划分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会对每个新描述在训练集和测试集上重新评估，最多迭代 5 次。完成后会在浏览器中打开一个 HTML 报告，展示每次迭代的结果，并返回带有 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### Skill 触发机制的工作原理

理解触发机制有助于设计更好的评估查询。Skill 会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 根据描述决定是否查阅某个 Skill。需要注意的是，Claude 只会对那些它无法自行轻松处理的任务查阅 Skill——简单的一步式查询（如"读取这个 PDF"）可能不会触发 Skill，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理它们。复杂的、多步骤的或专业化的查询，当描述匹配时，会可靠地触发 Skill。

这意味着你的评估查询应该足够实质，让 Claude 真正受益于查阅 Skill。像"读取文件 X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发 Skill。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，并更新 skill 的 SKILL.md  frontmatter。向用户展示前后的对比，并报告分数。

---

### 打包和呈现（仅在 `present_files` 工具可用时执行）

检查是否可以使用 `present_files` 工具。如果不能，请跳过此步骤。如果可以，请打包 skill 并向用户呈现 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，引导用户到生成的 `.skill` 文件路径，以便他们可以安装。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流程是相同的（起草 → 测试 → 审查 → 改进 → 循环），但由于 Claude.ai 没有子代理，一些机制会发生变化。以下是需要适应的内容：

**运行测试用例**：没有子代理意味着不能并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后按照其说明自行完成任务。一次处理一个。这不如独立子代理那样严谨（你编写了 skill 也在运行它，所以你有完整的上下文），但这是一个有用的合理性检查——人类审查步骤可以弥补。跳过基准运行——直接使用 skill 完成请求的任务。

**审查结果**：如果你无法打开浏览器（例如，Claude.ai 的虚拟机没有显示，或者你在远程服务器上），完全跳过浏览器审查器。相反，直接在对话中呈现结果。对于每个测试用例，显示提示词和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告诉他们位置以便下载检查。在对话中请求反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖于基准比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，请求反馈——只是中间没有浏览器审查器。如果你有文件系统，仍然可以将结果组织成迭代目录。

**描述优化**：本节需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果你在 Claude.ai 上，跳过它。

**盲测对比**：需要子代理。跳过它。

**打包**：`package_skill.py` 脚本可以在任何有 Python 和文件系统的环境下工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名和 `name` frontmatter 字段——保持不变。例如，如果安装的 skill 是 `research-helper`，输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑之前复制到可写位置。** 安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——直接写入可能会因权限而失败。

---

## Cowork 特定说明

如果你在 Cowork 中，主要需要了解的事项有：

- 你有子代理，所以主工作流程（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示而不是并行运行也是可以的。）
- 你没有浏览器或显示器，所以在生成 eval viewer 时，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。然后提供一个用户可以点击在浏览器中打开 HTML 的链接。
- 由于某些原因，Cowork 配置似乎不鼓励 Claude 在运行测试后生成 eval viewer，所以再重复一遍：无论你在 Cowork 还是 Claude Code 中，在运行测试后，你应该始终生成 eval viewer，让人类在看例子之后自己尝试修改 skill 并进行更正，使用 `generate_review.py`（而不是自己编写花哨的 html 代码）。提前道歉，但我还是要大写：**在评估输入之前先生成 EVAL VIEWER**。你想尽快把它们展示给人类！
- 反馈的工作方式不同：由于没有运行中的服务器，viewer 的"提交所有评论"按钮会将 `feedback.json` 下载为文件。然后你可以从那里读取它（你可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 的制作并且用户确认它处于良好状态后再进行。
- **更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。遵循上面 claude.ai 部分的更新指导。

---

## 参考文件

agents/ 目录包含专业子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何对两个输出进行盲测 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本胜过另一个

references/ 目录有额外的文档：

- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

再次重复核心循环以强调：

- 弄清楚 skill 是关于什么的
- 起草或编辑 skill
- 在测试提示上运行可访问 skill 的 claude
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终的 skill 并将其返回给用户。

如果你有这样的东西，请将步骤添加到你的 TodoList 中，以确保你不会忘记。如果你在 Cowork 中，请特别添加"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类审查测试用例"到你的 TodoList 中以确保它发生。

祝你好运！