---
name: skill-creator
description: 创建新 Skill，修改并改进现有 Skill，以及衡量 Skill 性能。当用户希望从头开始创建 Skill、更新或优化现有 Skill、运行 evals 以测试 Skill、通过方差分析对 Skill 性能进行基准测试，或优化 Skill 描述以提高触发准确性时使用。
---

# Skill Creator

一个用于创建新 Skill 并对其进行迭代改进的 Skill。

宏观来看，创建 Skill 的过程如下：

- 确定 Skill 的功能及其大致实现方式
- 编写 Skill 草稿
- 创建一些测试 prompt，并在其上运行 claude-with-access-to-the-skill
- 帮助用户从定性和定量两方面评估结果
  - 当运行在后台进行时，如果没有现成的定量 evals，起草一些（如果有，可以按原样使用，或者如果你觉得需要改变，也可以修改）。然后向用户解释（或者如果已经存在，解释现有的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，并让他们查看定量指标
- 根据用户对结果的评估反馈重写 Skill（如果定量基准测试中显示出明显的缺陷，也要进行修改）
- 重复直到满意为止
- 扩大测试集并尝试更大规模的测试

使用此 Skill 时，你的任务是确定用户处于此过程的哪个阶段，然后介入并帮助他们推进这些阶段。例如，他们可能会说“我想做一个用于 X 的 Skill”。你可以帮助缩小他们的需求范围，编写草稿，编写测试用例，确定他们想要如何评估，运行所有 prompt，并重复此过程。

另一方面，也许他们已经有了 Skill 的草稿。在这种情况下，你可以直接进入循环的评估/迭代部分。

当然，你应该始终保持灵活，如果用户说“我不需要运行一堆评估，随便聊聊就行”，你可以照做。

然后，在 Skill 完成后（同样，顺序是灵活的），你还可以运行 Skill 描述改进器，我们为此有一个单独的脚本，用于优化 Skill 的触发。

明白了吗？很好。

## 与用户沟通

Skill Creator 可能会被对编码术语熟悉程度各异的人群使用。如果你还没听说过（你怎么可能听说过呢，这只是最近才开始的事），现在的趋势是 Claude 的能力正在激励水管工打开终端，父母和祖父母去谷歌搜索“how to install npm”。另一方面，大多数用户可能具备相当的计算机素养。

因此，请注意语境线索，以了解如何措辞沟通！在默认情况下，为了给你一些概念：

- “evaluation”和“benchmark”处于临界状态，但可以使用
- 对于“JSON”和“assertion”，你需要在没有解释的情况下使用它们之前，看到用户明确知道这些是什么的迹象

如果你不确定，简要解释术语是可以的，如果你不确定用户是否能理解，可以随意用简短的定义来澄清术语。

---

## 创建 Skill

### 捕捉意图

首先理解用户的意图。当前的对话可能已经包含了用户想要捕捉的工作流（例如，他们说“把这个变成一个 Skill”）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前进行确认。

1. 这个 Skill 应该让 Claude 能够做什么？
2. 这个 Skill 何时应该触发？（什么用户短语/语境）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证 Skill 是否有效？具有客观可验证输出的 Skill（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的 Skill（写作风格、艺术）通常不需要。根据 Skill 类型建议适当的默认设置，但让用户决定。

### 访谈与研究

主动询问关于边缘情况、输入/输出格式、示例文件、成功标准和依赖关系的问题。在解决这部分问题之前，不要编写测试 prompt。

检查可用的 MCP——如果对研究有用（搜索文档、寻找类似的 Skill、查阅最佳实践），如果可用，通过 subagents 并行研究，否则在线进行。准备好上下文以减轻用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**: Skill 标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包括 Skill 的功能以及何时使用的具体语境。所有“何时使用”的信息都在这里，而不是在正文中。注意：目前 Claude 倾向于“触发不足”——在 Skill 有用时却不使用。为了解决这个问题，请让 Skill 描述稍微“强势”一点。例如，与其写“如何构建一个简单的快速 Dashboard 来显示内部 Anthropic 数据。”，不如写“如何构建一个简单的快速 Dashboard 来显示内部 Anthropic 数据。每当用户提到 Dashboard、数据可视化、内部指标，或想要显示任何类型的公司数据时，即使他们没有明确要求‘Dashboard’，也要确保使用此 Skill。”
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
- **Skill 的其余部分 :)**

### Skill 编写指南

#### Skill 剖析

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

#### Progressive Disclosure

Skills 采用三级加载系统：
1. **Metadata**（名称 + 描述）- 始终位于上下文中（约 100 词）
2. **SKILL.md body** - 当 skill 触发时位于上下文中（理想情况少于 500 行）
3. **Bundled resources** - 按需调用（无限制，脚本无需加载即可执行）

这些字数仅为预估值，如有必要可适当超出。

**关键模式：**
- 将 SKILL.md 控制在 500 行以内；若接近该限制，请增加一级层级结构，并附上明确指引，告知使用该 skill 的模型下一步该去往何处继续执行。
- 在 SKILL.md 中明确引用文件，并提供何时读取的指导
- 对于大型参考文件（>300 行），需包含目录

**领域组织**：当 skill 支持多个领域/框架时，按变体进行组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅读取相关的参考文件。

#### 无惊吓原则

这一点不言而喻，但技能绝不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果对技能的内容进行描述，其意图不应让用户感到意外。不要配合创建误导性技能或旨在协助未经授权访问、数据泄露及其他恶意活动的技能请求。不过，诸如“扮演 XYZ”之类的情况是可以的。

#### 编写模式

在指令中优先使用祈使语气。

**定义输出格式** - 你可以这样做：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有帮助。你可以按如下格式编排（但如果示例中包含 "Input" 和 "Output"，你可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释为何事情重要，而非使用生硬陈旧的“必须（MUST）”指令。运用心智理论，尽量使 Skill 具备通用性，而非局限于特定示例。先起草初稿，随后以全新的视角审视并进行改进。

### 测试用例

编写完 Skill 草稿后，设计 2-3 个真实的测试 Prompt —— 即真实用户实际会提出的内容。与用户分享这些 Prompt：“我想尝试这几个测试用例。它们看起来合适吗？您是否需要添加更多？”（无需严格使用此措辞）。随后运行测试。

将测试用例保存至 `evals/evals.json`。暂不要编写断言 —— 仅保留 Prompt。您将在下一步运行过程中起草断言。

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

请参阅 `references/schemas.md` 获取完整的 schema（包含 `assertions` 字段，你稍后会添加该字段）。

## 运行和评估测试用例

本节是一个连续的过程——请勿中途停止。请勿使用 `/skill-test` 或任何其他测试 skill。

将结果放入 `<skill-name>-workspace/`，作为 skill 目录的兄弟目录。在 workspace 内，按迭代组织结果（`iteration-1/`、`iteration-2/` 等），并在该目录下为每个测试用例创建一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有这些目录——只需在过程中按需创建。

### 步骤 1：在同一轮次中启动所有运行（带 skill 和 baseline）

对于每个测试用例，在同一轮次中启动两个 subagent——一个带 skill，一个不带。这一点很重要：不要先启动带 skill 的运行，稍后再回来处理 baseline。同时启动所有内容，以便它们大约在同一时间完成。

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
- **Creating a new skill**：完全没有 skill。使用相同的 prompt，无 skill path，保存到 `without_skill/outputs/`。
- **Improving an existing skill**：旧版本。编辑前，对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline subagent 指向该 snapshot。保存到 `old_skill/outputs/`。

为每个测试用例编写一个 `eval_metadata.json`（assertions 目前可以为空）。根据测试内容为每个 eval 命名一个描述性的名称——而不仅仅是 "eval-0"。目录名也使用此名称。如果此次迭代使用了新的或修改过的 eval prompts，请为每个新的 eval 目录创建这些文件——不要假设它们是从之前的迭代中继承的。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：当 runs 正在进行时，起草 assertions

不要只是等待 runs 结束 —— 你可以高效利用这段时间。为每个 test case 起草定量 assertions 并向用户解释。如果 `evals/evals.json` 中已存在 assertions，请审查它们并解释它们检查的内容。

好的 assertions 应是客观可验证的且具有描述性名称 —— 它们应在 benchmark viewer 中清晰易读，以便浏览结果的人能立即理解每个 assertion 检查的内容。主观技能（写作风格、设计质量）更适合定性评估 —— 不要强行将 assertions 用于需要人工判断的事物。

一旦起草完成，请用 assertions 更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们将在 viewer 中看到什么 —— 包括定性输出和定量 benchmark。

### 步骤 3：当 runs 完成时，捕获计时数据

当每个 subagent task 完成时，你会收到一个包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到 run 目录下的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会 —— 它通过 task notification 传递，且未持久化到其他地方。请在每个 notification 到达时立即处理，而非尝试批量处理。

### 步骤 4：评分、聚合并启动 viewer

当所有运行完成后：

1. **对每次运行进行评分** —— 生成一个 grader subagent（或进行内联评分），读取 `agents/grader.md` 并依据输出评估每个 assertion。将结果保存到每个运行目录下的 `grading.json` 中。`grading.json` 中的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而非 `name`/`met`/`details` 或其他变体）—— viewer 依赖于这些精确的字段名称。对于可通过编程方式检查的 assertion，请编写并运行脚本而非人工目测 —— 脚本更快、更可靠，且可跨迭代复用。

2. **聚合为 benchmark** —— 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每种配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 以了解 viewer 所需的确切 schema。
将每个 with_skill 版本放在其 baseline 对应项之前。

3. **进行一轮分析师审查** —— 阅读 benchmark data，揭示 aggregate stats 可能隐藏的模式。请参阅 `agents/analyzer.md`（“Analyzing Benchmark Results”章节）以了解需要关注的内容——例如无论 skill 如何总是通过的 assertions（non-discriminating）、high-variance evals（possibly flaky），以及 time/token tradeoffs。

4. **启动 viewer**，同时包含 qualitative outputs 和 quantitative data：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及之后的迭代，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作 / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。当用户点击 "Submit All Reviews" 时，反馈将作为 `feedback.json` 文件下载。下载后，将 `feedback.json` 复制到工作空间目录，以便下一次迭代读取。

注意：请使用 generate_review.py 创建查看器；无需编写自定义 HTML。

5. **告知用户**类似以下内容：“我已在您的浏览器中打开结果。有两个标签页——'Outputs' 允许您点击查看每个测试用例并留下反馈，'Benchmark' 展示定量对比。完成后，请回到此处告诉我。”

### 用户在查看器中看到的内容

“Outputs”标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，在可能的情况下内嵌渲染
- **Previous Output**（第 2 次及后续迭代）：显示上一次迭代输出的折叠部分
- **Formal Grades**（如果运行了评分）：显示断言通过/失败的折叠部分
- **Feedback**：一个输入时自动保存的文本框
- **Previous Feedback**（第 2 次及后续迭代）：他们上次的评论，显示在文本框下方

“Benchmark”标签页显示统计摘要：每个配置的通过率、耗时和 Token 使用量，以及每次评估的细分和分析师观察结果。

通过上一个/下一个按钮或方向键进行导航。完成后，他们点击 "Submit All Reviews"，将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

当用户告诉你他们已完成时，读取 `feedback.json`：

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

空反馈意味着用户认为结果没问题。请重点改进用户提出具体意见的测试用例。

使用完毕后，请关闭 viewer server：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 Skill

这是循环的核心。你已经运行了 test case，用户也审查了结果，现在你需要根据他们的反馈让 skill 变得更好。

### 如何思考改进

1. **从反馈中进行归纳。** 这里的宏观图景是，我们正试图创建可以在许多不同的 prompt 中使用数百万次（也许字面上就是如此，甚至更多）的 skill。在这里，你和用户只在几个示例上反复迭代，因为这有助于加快速度。用户对这些示例了如指掌，评估新 output 对他们来说很快。但是，如果你和用户共同开发的 skill 仅适用于那些示例，那它就毫无用处。与其进行繁琐的过拟合修改，或添加压迫性限制的 MUST，如果遇到顽固问题，你可以尝试扩展思路，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会发现很棒的东西。

2. **保持 prompt 精简。** 移除那些没有发挥作用的内容。务必阅读 transcript，而不仅仅是最终的 output —— 如果看起来 skill 正在让 model 浪费大量时间做无益的事情，你可以尝试删除 skill 中导致这种情况的部分，看看会发生什么。

3. **解释原因。** 尽力解释你要求 model 做每件事背后的**原因**。如今的 LLM *很聪明*。它们拥有良好的心智理论，当被赋予良好的引导机制时，能够超越死板的指令，真正把事情做成。即使来自用户的反馈简短或令人沮丧，也要尝试真正理解任务，为什么用户要写他们所写的内容，以及他们实际写了什么，然后将这种理解传达指令中。如果你发现自己正在用全大写字母书写 ALWAYS 或 NEVER，或者使用超级僵化的结构，那就是一个警示信号——如果可能的话，重构并解释推理过程，以便 model 理解你要求的事情为何重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找 test case 中的重复工作。** 阅读 test run 的 transcript，注意 subagent 是否都独立编写了类似的 helper script 或对某事采取了相同的多步骤方法。如果所有 3 个 test case 都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这是一个强烈的信号，表明 skill 应该捆绑该 script。编写一次，将其放入 `scripts/`，并告诉 skill 使用它。这可以节省未来的每次调用，避免重复造轮子。

这项任务相当重要（我们正试图在这里创造每年数十亿的经济价值！），你的思考时间并不是瓶颈；花点时间，认真反复琢磨。我建议先写一份修订草案，然后重新审视并加以改进。尽最大努力进入用户的脑海，理解他们想要和需要什么。

### 迭代循环

改进 skill 后：

1. 将你的改进应用到 skill 中
2. 将所有 test case 重新运行到一个新的 `iteration-<N+1>/` 目录中，包括 baseline run。如果你正在创建一个新的 skill，baseline 始终是 `without_skill`（没有 skill）——这在迭代过程中保持不变。如果你正在改进现有的 skill，请自行判断什么作为 baseline 更合理：用户带来的原始版本，还是上一次迭代。
3. 启动 reviewer，并让 `--previous-workspace` 指向上一次迭代
4. 等待用户审查并告诉你他们已完成
5. 阅读新的反馈，再次改进，重复此过程

继续进行直到：
- 用户表示满意
- 反馈全部为空（一切看起来都不错）
- 你没有取得实质性进展

---

## 高级：盲测对比

对于想要在两个版本的 skill 之间进行更严格比较的情况（例如，用户问“新版本真的更好吗？”），有一个盲测对比系统。请阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思想是：将两个 output 提供给一个独立的 agent，而不告诉它哪个是哪个，让它判断质量。然后分析获胜者获胜的原因。

这是可选的，需要 subagent，大多数用户不需要它。人工审查循环通常就足够了。

---

## Description 优化

`SKILL.md` frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。在创建或改进 skill 后，提议优化 description 以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个 eval query —— 混合应触发和不应触发的情况。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

查询必须真实，是 Claude Code 或 Claude.ai 用户实际会输入的内容。不要抽象的请求，而是具体、详细、有实际内容的请求。例如，文件路径、关于用户工作或情况的个人背景、列名和值、公司名称、URL。还要有一些背景故事。有些可能使用小写、包含缩写、拼写错误或口语化表达。使用不同长度的混合，重点关注边界情况而非清晰明确的情况（用户后续有机会确认这些查询）。

差: `"Format this data"`, `"Extract text from PDF"`, `"Create a chart"`

好: `"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

对于 **should-trigger** 查询（8-10条），要考虑覆盖面。你需要用不同的措辞表达同一意图——有些正式，有些随意。包括用户没有明确说出技能或文件类型但显然需要它的情况。加入一些不常见的用例，以及该技能与其他技能竞争但应该胜出的情况。

对于 **should-not-trigger** 查询（8-10条），最有价值的是那些"擦边球"——与技能共享关键词或概念但实际上需要其他功能的查询。考虑相邻领域、朴素关键词匹配会触发但不应该触发的歧义表达，以及查询涉及该技能功能但在上下文中另一个工具更合适的情况。

要避免的关键点：不要让 should-not-trigger 查询明显无关。以 "Write a fibonacci function" 作为 PDF 技能的负面测试太简单了——它测试不了任何东西。负面案例应该真正具有迷惑性。

### 第2步：与用户审核

使用 HTML 模板向用户展示评估集进行审核：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → 评估项的 JSON 数组（不要加引号——这是一个 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → 技能名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → 技能的当前描述
3. 写入临时文件（如 `/tmp/eval_review_<skill-name>.html`）并打开：`open /tmp/eval_review_<skill-name>.html`
4. 用户可以编辑查询、切换 should-trigger、添加/删除条目，然后点击 "Export Eval Set"
5. 文件下载到 `~/Downloads/eval_set.json` ——检查 Downloads 文件夹获取最新版本，以防有多个文件（如 `eval_set (1).json`）

这一步很重要——糟糕的评估查询会导致糟糕的描述。

### 第3步：运行优化循环

告诉用户："这需要一些时间——我会在后台运行优化循环并定期检查。"

将评估集保存到工作区，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的 model ID（即驱动当前会话的那个），以便触发测试与用户的实际体验相匹配。

运行期间，定期查看输出，向用户更新当前迭代次数和分数情况。

这会自动处理完整的优化循环。它将 eval set 拆分为 60% 的训练集和 40% 的留出测试集，评估当前的 description（对每个 query 运行 3 次以获取可靠的触发率），然后调用具有 extended thinking 功能的 Claude，根据失败的情况提出改进建议。它会在训练集和测试集上重新评估每个新的 description，最多迭代 5 次。完成后，它会在浏览器中打开一份 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON —— 该结果根据测试集分数而非训练集分数选出，以避免过拟合。

### Skill 触发机制

理解触发机制有助于设计更好的 eval queries。Skills 会显示在 Claude 的 `available_skills` 列表中，包含其名称和 description，Claude 根据该 description 决定是否使用该 skill。重要的是要知道，Claude 只会在遇到它无法轻松独立处理的任务时才会使用 skills —— 像 "read this PDF" 这样简单的单步查询，即使 description 完美匹配，也可能不会触发 skill，因为 Claude 可以使用基础工具直接处理它们。当 description 匹配时，复杂、多步骤或专业化的查询会可靠地触发 skills。

这意味着你的 eval queries 应该足够有实质内容，让 Claude 确实能从使用 skill 中受益。像 "read file X" 这样的简单查询是糟糕的测试用例 —— 无论 description 质量如何，它们都不会触发 skills。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description` 并更新 skill 的 SKILL.md frontmatter。向用户展示修改前后的对比并报告分数。

---

### 打包和展示（仅当 `present_files` tool 可用时）

检查你是否有权访问 `present_files` tool。如果没有，请跳过此步骤。如果有，打包 skill 并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，引导用户找到生成的 `.skill` 文件路径，以便他们进行安装。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心 workflow 是相同的（draft → test → review → improve → repeat），但由于 Claude.ai 没有 subagents，一些机制需要调整。以下是需要适配的内容：

**Running test cases**: 没有 subagents 意味着无法并行执行。对于每个 test case，读取 skill 的 SKILL.md，然后按照其指令自行完成测试 prompt。逐一执行。这种方式不如独立的 subagents 严谨（因为你编写了 skill 同时也在运行它，所以拥有完整的上下文），但这是一个有用的完整性检查——人工审查步骤可以弥补这一点。跳过 baseline runs——直接使用 skill 按要求完成任务。

**Reviewing results**: 如果无法打开浏览器（例如 Claude.ai 的 VM 没有显示器，或者你在远程服务器上），则完全跳过浏览器审查环节。改为直接在对话中展示结果。对于每个 test case，显示 prompt 和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告诉用户位置，以便他们下载和检查。在对话中询问反馈："这个看起来怎么样？有什么需要修改的吗？"

**Benchmarking**: 跳过定量 benchmarking——它依赖于 baseline comparisons，在没有 subagents 的情况下没有意义。专注于用户的定性反馈。

**The iteration loop**: 与之前相同——改进 skill，重新运行 test cases，请求反馈——只是中间没有浏览器审查环节。如果有文件系统，仍然可以将结果组织到 iteration 目录中。

**Description optimization**: 此部分需要 `claude` CLI 工具（具体为 `claude -p`），该工具仅在 Claude Code 中可用。如果你在 Claude.ai 上，请跳过此步骤。

**Blind comparison**: 需要 subagents。跳过。

**Packaging**: `package_skill.py` 脚本可在任何有 Python 和文件系统的环境中运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

---

## Cowork 特定说明

如果你在 Cowork 中，主要需要了解以下几点：

- 你有 subagents，所以主 workflow（spawn test cases in parallel、run baselines、grade 等）都能正常运行。（但是，如果遇到严重的超时问题，可以串行而非并行运行 test prompts。）
- 没有浏览器或显示器，所以在生成 eval viewer 时，使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开该 HTML。
- 无论什么原因，Cowork 环境似乎会让 Claude 在运行测试后不愿生成 eval viewer，所以再次强调：无论你在 Cowork 还是 Claude Code 中，运行测试后，你应该始终为人生成 eval viewer，让他们在你自己修改 skill 和尝试修正之前查看示例，使用 `generate_review.py`（而不是编写自己的定制 html 代码）。提前抱歉，但我要用全大写了：在你自己评估输入之前 *先* 生成 EVAL VIEWER。你要尽快把它们呈现给用户！
- Feedback 工作方式不同：由于没有运行中的服务器，viewer 的 "Submit All Reviews" 按钮会将 `feedback.json` 作为文件下载。然后你可以从中读取（可能需要先请求访问权限）。
- Packaging 可以正常工作——`package_skill.py` 只需要 Python 和文件系统。
- Description optimization（`run_loop.py` / `run_eval.py`）在 Cowork 中应该能正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 制作且用户认为它状态良好后再进行。

---

## 参考文件

agents/ 目录包含专门的 subagents 的指令。需要生成相关 subagent 时请阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本为何优于另一个版本

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次重复核心循环以示强调：

- 弄清楚 skill 是做什么的
- 起草或编辑 skill
- 在 test prompts 上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终的 skill 并返回给用户

请将这些步骤添加到你的 TodoList（如果有的话）中，以确保不会忘记。如果你在 Cowork 中，请特别将 "Create evals JSON and run `eval-viewer/generate_review.py` so human can review test cases" 放入你的 TodoList，以确保它会执行。

祝你好运！