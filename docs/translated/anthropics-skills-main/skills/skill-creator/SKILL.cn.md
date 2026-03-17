---
name: skill-creator
description: 创建新 skill，修改和改进现有 skill，并衡量 skill 性能。当用户想要从头创建 skill、编辑或优化现有 skill、运行 evals 测试 skill、通过方差分析进行 skill 性能基准测试，或优化 skill 的描述以提高触发准确性时使用。
---

# Skill Creator

一个用于创建新 skill 并迭代改进它们的 skill。

从宏观层面来看，创建 skill 的过程如下：

- 确定 skill 的功能及其大致实现方式
- 编写 skill 草稿
- 创建一些测试 prompt，并运行 claude-with-access-to-the-skill 进行测试
- 帮助用户从定性和定量两个角度评估结果
  - 在后台运行期间，如果没有现成的定量 evals，则起草一些（如果已有，可以照常使用，或者如果觉得需要更改则进行修改）。然后向用户解释它们（或者如果它们已经存在，解释现有的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，并让他们查看定量指标
- 根据用户对结果的评估反馈（以及定量基准测试中显现的任何明显缺陷）重写 skill
- 重复此过程直到满意为止
- 扩大测试集并在更大规模上再次尝试

使用此 skill 时，你的任务是确定用户处于流程的哪个阶段，然后介入并帮助他们推进这些阶段。例如，他们可能会说“我想为 X 制作一个 skill”。你可以帮助缩小他们的意图范围，编写草稿，编写测试用例，确定他们想要如何评估，运行所有 prompt，并重复此过程。

另一方面，也许他们已经有了 skill 草稿。在这种情况下，你可以直接进入循环的评估/迭代部分。

当然，你应该始终保持灵活，如果用户说“我不需要运行一堆评估，只想随意聊聊”，你可以照做。

然后，在 skill 完成后（同样，顺序是灵活的），你还可以运行 skill description improver，我们要为此单独准备了一个脚本，用来优化 skill 的触发。

明白了吗？很好。

## 与用户沟通

Skill creator 可能会被对代码术语熟悉程度各异的人群使用。如果你还没听说过（你怎么可能听说过，毕竟这 trend 才刚刚开始），现在有一种趋势，Claude 的强大能力正在激励水管工打开他们的终端，父母和祖父母去谷歌搜索“how to install npm”。另一方面，大多数用户可能具备相当的计算机素养。

因此，请注意语境线索，以了解如何措辞沟通！在默认情况下，为了给你一些参考：

- “evaluation”和“benchmark”处于临界点，但可以使用
- 对于“JSON”和“assertion”，在用户未了解这些概念的情况下使用前，你需要看到用户确实知道这些是什么的确切迹象，否则需要解释

如果你不确定，可以简要解释术语，如果你不确定用户是否能理解，可以随意用简短的定义进行澄清。

---

## 创建 skill

### 捕捉意图

首先理解用户的意图。当前的对话可能已经包含了用户想要捕捉的 workflow（例如，他们说“把这个变成一个 skill”）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前进行确认。

1. 这个 skill 应该让 Claude 能够做什么？
2. 这个 skill 何时应该触发？（用户说的什么话/什么语境）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证 skill 是否正常工作？具有客观可验证输出的 skill（文件转换、数据提取、代码生成、固定的 workflow 步骤）受益于测试用例。具有主观输出的 skill（写作风格、艺术）通常不需要。根据 skill 类型建议适当的默认值，但让用户决定。

### 访谈与研究

主动询问关于边缘情况、输入/输出格式、示例文件、成功标准和依赖关系的问题。在搞定这部分之前，先不要编写测试 prompt。

检查可用的 MCPs - 如果对研究有用（搜索文档、查找类似的 skill、查找最佳实践），如果可用，通过 subagents 并行研究，否则内联进行。准备好上下文以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：Skill 标识符
- **description**：何时触发，做什么。这是主要的触发机制 - 包括 skill 的功能以及何时使用它的具体语境。所有“何时使用”的信息都在这里，而不是在正文中。注意：目前 Claude 有一种“触发不足”的倾向——即在该使用 skill 时不使用它们。为了解决这个问题，请让 skill 的描述稍微“强势”一点。例如，与其写“如何构建一个简单快速的 dashboard 来显示 Anthropic 内部数据”，不如写“如何构建一个简单快速的 dashboard 来显示 Anthropic 内部数据。每当用户提到 dashboard、数据可视化、内部指标，或想要显示任何类型的公司数据时，请务必使用此 skill，即使他们没有明确要求‘dashboard’。”
- **compatibility**：所需工具、依赖项（可选，很少需要）
- **skill 的其余部分 :)**

### Skill 编写指南

#### Skill 的剖析

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
1. **Metadata** (名称 + 描述) - 始终位于上下文中（约 100 词）
2. **SKILL.md body** - 每当 Skill 触发时即位于上下文中（理想情况下 <500 行）
3. **Bundled resources** - 按需加载（无限制，脚本可在不加载的情况下执行）

这些字数仅为概略估算，如有需要，篇幅可适当延长。

**关键模式：**
- 将 SKILL.md 保持在 500 行以内；若接近此限制，请增加额外的层级结构，并明确指出使用该 Skill 的模型接下来应去往何处进行后续操作。
- 在 SKILL.md 中清晰引用文件，并提供关于何时读取这些文件的指引
- 对于大型参考文件（>300 行），需包含目录

**领域组织**：当一个 Skill 支持多个领域/框架时，请按变体进行组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 只读取相关的参考文件。

#### 无惊吓原则

这一点不言而喻，但 skill 绝不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果对 skill 的意图进行描述，其内容不应让用户感到意外。不要配合创建具有误导性的 skill，或旨在促成未授权访问、数据窃取或其他恶意活动的 skill。不过，像“扮演 XYZ”之类的情况是可以接受的。

#### 写作模式

在指令中优先使用祈使句形式。

**定义输出格式** - 方法如下：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例非常有用。您可以像这样格式化它们（但如果示例中包含 "Input" 和 "Output"，您可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事情的重要性，而不是使用生硬陈旧的强制性措辞。运用 theory of mind（心智理论），尽量使技能具有通用性，而不局限于特定示例。先撰写初稿，然后用全新的视角审视并加以改进。

### 测试用例

编写完技能初稿后，构思 2-3 个真实的测试提示——即真实用户实际会提出的问题。与用户分享这些提示：[不必完全使用这段原话] “这里有几个我想尝试的测试用例。这些看起来合适吗，还是您想再添加一些？” 然后运行它们。

将测试用例保存到 `evals/evals.json`。暂时不要编写断言——只写提示即可。您将在下一步运行期间编写断言。

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

请参阅 `references/schemas.md` 获取完整 schema（包括你稍后会添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的流程 —— 请勿中途停止。请勿使用 `/skill-test` 或任何其他测试 skill。

将结果放入 `<skill-name>-workspace/`，作为 skill 目录的兄弟目录。在 workspace 内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，并在其中为每个测试用例创建一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有这些 —— 只需在过程中按需创建目录。

### 步骤 1：在同一轮次中生成所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮次中生成两个 subagent —— 一个使用 skill，一个不使用。这一点很重要：不要先生成 with-skill 运行，稍后再回来处理 baseline。请一次性启动所有任务，以便它们能在大致相同的时间完成。

**With-skill 运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同的 prompt，但 baseline 依赖于上下文）：
- **Creating a new skill**：完全没有 skill。相同的 prompt，无 skill path，保存到 `without_skill/outputs/`。
- **Improving an existing skill**：旧版本。编辑前，对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline subagent 指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写一个 `eval_metadata.json`（assertions 目前可以为空）。根据测试内容为每个 eval 指定一个描述性名称——不要仅命名为 "eval-0"。此名称也用于目录名。如果本次迭代使用新的或修改过的 eval prompt，请为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代中沿用。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行期间，草拟断言

不要只是等待运行结束——你可以充分利用这段时间。为每个测试用例草拟定量断言，并向用户解释这些断言。如果 `evals/evals.json` 中已存在断言，请审查它们并解释其检查内容。

好的断言是客观可验证的，且具有描述性的名称——它们应在基准测试查看器中清晰可读，以便浏览结果的人能立即理解每项断言的检查内容。主观技能（如写作风格、设计质量）更适合进行定性评估——不要对需要人工判断的内容强行使用断言。

断言草拟完成后，请更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户说明他们将在查看器中看到的内容——包括定性输出和定量基准测试结果。

### 步骤 3：随着运行完成，记录计时数据

当每个 subagent 任务完成时，你会收到一个包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录下的 `timing.json` 文件中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——数据通过任务通知传递，且未在其他地方持久化。请在通知到达时立即处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分** —— 生成一个 grader subagent（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每个运行目录下的 `grading.json` 中。`grading.json` 中的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而非 `name`/`met`/`details` 或其他变体）—— 查看器依赖于这些确切的字段名称。对于可以通过编程方式检查的断言，请编写并运行脚本，而非人工查看 —— 脚本速度更快、更可靠，并且可以在迭代中重复使用。

2. **聚合为基准测试** —— 运行 skill-creator 目录下的聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每种配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 以获取查看器所需的精确 schema。
将每个 with_skill 版本置于其对应的 baseline 版本之前。

3. **执行分析师审查** —— 阅读基准测试数据，揭示聚合统计数据可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results" 章节）了解需要关注的内容——例如无论 skill 如何总是通过的断言（缺乏区分度）、高方差 evals（可能不稳定）以及 time/token tradeoffs。

4. **启动 viewer**，加载定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及后续的迭代，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

   **协作/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 来写入一个独立的 HTML 文件，而不是启动服务器。当用户点击 "Submit All Reviews" 时，反馈将作为 `feedback.json` 文件下载。下载后，将 `feedback.json` 复制到工作空间目录，以便下一次迭代读取。

注意：请使用 generate_review.py 创建查看器；无需编写自定义 HTML。

5. **告诉用户** 类似这样的话：“我已在浏览器中打开了结果。有两个标签页 — 'Outputs' 允许您点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后，请回到这里告诉我。”

### 用户在查看器中看到的内容

“Outputs”标签页一次显示一个测试用例：
- **Prompt**：给定的任务
- **Output**：技能生成的文件，在可能的情况下进行内联渲染
- **Previous Output**（第 2 次及后续迭代）：显示上一次迭代输出的折叠区域
- **Formal Grades**（如果运行了评分）：显示断言通过/失败情况的折叠区域
- **Feedback**：一个随输入自动保存的文本框
- **Previous Feedback**（第 2 次及后续迭代）：上次的评论，显示在文本框下方

“Benchmark”标签页显示统计摘要：每种配置的通过率、耗时和 Token 使用情况，以及各次评估的细分详情和分析观察。

导航通过上一步/下一步按钮或方向键完成。完成后，他们点击 "Submit All Reviews"，这将保存所有反馈到 `feedback.json`。

### 步骤 5：读取反馈

当用户告诉你他们完成了，读取 `feedback.json`：

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

空反馈意味着用户认为结果没问题。重点改进用户有具体投诉的测试用例。

使用完毕后终止查看器服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 Skill

这是循环的核心。你已经运行了测试用例，用户也已经审查了结果，现在你需要根据他们的反馈来改进这个 skill。

### 如何思考改进

1. **从反馈中归纳总结。** 这里的宏观图景是，我们正试图创建能够跨许多不同 prompt 使用数百万次（可能是字面意思，甚至更多谁知道呢）的 skill。在这里，你和用户仅针对少数示例反复迭代，因为这有助于加快进度。用户对这些示例了如指掌，评估新输出对他们来说很快。但如果你们共同开发的 skill 仅适用于这些示例，那它就毫无用处。与其进行琐碎且过拟合（overfitty）的修改，或者制定压抑性的强制性“必须”（MUST），如果遇到顽固问题，你可以尝试另辟蹊径，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会发现很棒的东西。

2. **保持 Prompt 精简。** 移除那些没有发挥作用的内容。务必阅读记录，而不仅仅是最终输出——如果看起来 skill 导致模型浪费大量时间做无效的事情，你可以尝试删除 skill 中导致这种情况的部分，看看会发生什么。

3. **解释原因。** 尽力解释你要求模型做每件事背后的**原因**。如今的 LLM 非常*聪明*。它们拥有良好的心智理论，当被赋予良好的引导时，它们能超越死板的指令，真正把事情做成。即使用户的反馈简短或充满挫败感，也要尝试真正理解任务，理解用户为什么写下这些内容，以及他们实际写了什么，然后将这种理解传达指令中。如果你发现自己用全大写字母写 ALWAYS 或 NEVER，或者使用超级僵硬的结构，那就是一个警示信号——如果可能的话，重新构建框架并解释推理过程，以便模型理解你要求的事情为何重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找跨测试用例的重复工作。** 阅读测试运行的记录，注意 subagent 是否都独立编写了类似的辅助脚本，或者对某事采取了相同的多步骤方法。如果所有 3 个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明 skill 应该打包该脚本。编写一次，放入 `scripts/`，并告诉 skill 使用它。这能让未来的每次调用免于重复造轮子。

这项任务非常重要（我们正试图在这里创造每年数十亿的经济价值！），你的思考时间不是瓶颈；花点时间真正仔细斟酌一下。我建议先写一个修改草案，然后重新审视并加以改进。尽力进入用户的思维，理解他们想要和需要什么。

### 迭代循环

改进 skill 后：

1. 将你的改进应用到 skill 中
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建一个新的 skill，基线始终是 `without_skill`（无 skill）——这在迭代过程中保持不变。如果你正在改进现有的 skill，请根据判断选择合理的基线：用户带来的原始版本，或上一次迭代。
3. 启动 reviewer，并使用 `--previous-workspace` 指向上一次迭代
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复此过程

持续进行直到：
- 用户表示满意
- 反馈全为空（一切看起来都很好）
- 你没有取得实质性进展

---

## 进阶：盲测对比

对于需要更严格地对比 skill 的两个版本的情况（例如，用户问“新版本真的更好吗？”），有一个盲测对比系统。请阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出提供给一个独立的 agent，但不告知它哪个是哪个，让它评判质量。然后分析获胜者获胜的原因。

这是可选的，需要 subagent，大多数用户不需要它。人工审查循环通常就足够了。

---

## Description 优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。在创建或改进 skill 后，提议优化 description 以获得更好的触发准确率。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合了应触发和不应触发的查询。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

查询内容必须真实，应是 Claude Code 或 Claude.ai 用户实际会输入的内容。不要抽象的请求，而应是具体、明确且包含丰富细节的请求。例如，文件路径、关于用户工作或处境的个人背景、列名和数值、公司名称、URL。包含一点背景故事。有些可能是小写的，或者包含缩写、拼写错误或口语化表达。请混合使用不同的长度，重点关注边缘情况，而不是让它们过于清晰明确（用户稍后会有机会确认）。

错误：`"Format this data"`, `"Extract text from PDF"`, `"Create a chart"`

正确：`"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

对于 **should-trigger** 查询（8-10 条），请考虑覆盖率。你需要针对同一意图采用不同的措辞——有些正式，有些随意。包括用户未明确指出技能或文件类型但显然需要它的情况。加入一些不常见的用例，以及该技能与另一技能存在竞争但应胜出的情况。

对于 **should-not-trigger** 查询（8-10 条），最有价值的是那些“险些误触”的查询——即与技能共享关键词或概念，但实际上需要不同功能的查询。考虑相邻领域、可能导致简单关键词匹配误触发的歧义措辞，以及查询涉及技能功能但在上下文中另有更合适工具的情况。

需要避免的关键点：不要让 **should-not-trigger** 查询显得明显无关。例如，将 "Write a fibonacci function" 作为 PDF 技能的负面测试过于简单——它测试不出任何东西。负面案例应具有真正的迷惑性。

### 步骤 2：与用户审查

使用 HTML 模板向用户展示评估集以供审查：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → eval 项的 JSON 数组（周围不加引号——这是一个 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → 技能名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → 技能的当前描述
3. 写入临时文件（例如 `/tmp/eval_review_<skill-name>.html`）并打开它：`open /tmp/eval_review_<skill-name>.html`
4. 用户可以编辑查询、切换 should-trigger 状态、添加/删除条目，然后点击 "Export Eval Set"
5. 文件将下载到 `~/Downloads/eval_set.json` —— 请检查 Downloads 文件夹以获取最新版本，以防存在多个文件（例如 `eval_set (1).json`）

这一步很重要——糟糕的 eval 查询会导致糟糕的描述。

### 步骤 3：运行优化循环

告诉用户：“这需要一些时间——我会在后台运行优化循环并定期检查。”

将评估集保存到工作区，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用你系统提示词中的 Model ID（即为当前会话提供支持的那个），以便触发测试与用户的实际体验相匹配。

运行期间，定期跟踪输出，向用户更新当前迭代次数和分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 的训练集和 40% 的留出测试集，评估当前的描述（每个查询运行 3 次以获得可靠的触发率），然后根据失败的情况调用 Claude 提出改进建议。它会在训练集和测试集上重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON —— 该描述是根据测试分数而非训练分数选择的，以避免过拟合。

### Skill 触发机制

理解触发机制有助于设计更好的评估查询。Skills 会出现在 Claude 的 `available_skills` 列表中，包含其名称和描述，Claude 根据该描述决定是否咨询 Skill。需要知道的重要一点是，Claude 只会为它难以独自处理的任务咨询 Skills —— 像“读取此 PDF”这样简单的单步查询，即使描述完美匹配，也可能不会触发 Skill，因为 Claude 可以使用基本工具直接处理它们。当描述匹配时，复杂、多步骤或专门的查询会可靠地触发 Skills。

这意味着你的评估查询应该足够有实质内容，让 Claude 确实能从咨询 Skill 中受益。像“读取文件 X”这样的简单查询是糟糕的测试用例 —— 无论描述质量如何，它们都不会触发 Skills。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description`，并更新 Skill 的 SKILL.md frontmatter。向用户展示修改前后的内容并报告分数。

---

### 打包与展示（仅当 `present_files` 工具可用时）

检查你是否有权使用 `present_files` 工具。如果没有，请跳过此步骤。如果有，请打包 Skill 并将 .skill 文件展示给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，引导用户找到生成的 `.skill` 文件路径，以便他们进行安装。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流程相同（草稿 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有 subagents，部分机制有所调整。以下是需要适应的地方：

**运行测试用例**：没有 subagents 意味着无法并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后按照其说明自行完成测试提示。请逐个执行。这不如独立的 subagents 严谨（因为编写 skill 的人也是运行它的人，所以你拥有完整的上下文），但这是一种有效的健全性检查——而且人工审查步骤可以弥补这一不足。跳过基线运行——只需使用 skill 按要求完成任务即可。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的 VM 没有显示器，或者你在远程服务器上），请完全跳过浏览器审查。取而代之的是，直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），请将其保存到文件系统并告知位置，以便用户下载和检查。在线询问反馈：“这个看起来怎么样？有什么需要修改的地方吗？”

**基准测试**：跳过定量基准测试——它依赖于基线比较，在没有 subagents 的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，请求反馈——只是中间没有浏览器审查环节。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（具体为 `claude -p`），该工具仅在 Claude Code 中可用。如果你在 Claude.ai 上，请跳过此步骤。

**盲测对比**：需要 subagents。跳过。

**打包**：`package_skill.py` 脚本可在任何有 Python 和文件系统的地方运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有的 skill，而不是创建新的。在这种情况下：
- **保留原始名称。** 注意 skill 的目录名和 `name` frontmatter 字段——保持不变。例如，如果安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **编辑前复制到可写位置。** 安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，并从副本打包。
- **如果手动打包，先暂存在 `/tmp/`**，然后复制到输出目录——由于权限问题，直接写入可能会失败。

---

## Cowork 特定说明

如果你处于 Cowork 环境中，主要需要了解的是：

- 你拥有 subagents，因此主工作流程（并行生成测试用例、运行基线、评分等）均可正常运行。（但是，如果遇到严重的超时问题，可以串行而非并行地运行测试提示。）
- 你没有浏览器或显示器，因此在生成评估查看器时，请使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击该链接在浏览器中打开 HTML。
- 无论出于何种原因，Cowork 的设置似乎会让 Claude 在运行测试后不愿生成评估查看器，所以重申一下：无论你是在 Cowork 还是 Claude Code 中，运行测试后，你应该始终使用 `generate_review.py`（而不是编写你自己的定制 html 代码）生成评估查看器，以便人类在你自己修订 skill 并尝试纠正之前查看示例。提前抱歉，但我这里要用大写了：在你自己评估输入之前，先生成评估查看器。你需要尽快让人类看到它们！
- 反馈机制不同：由于没有运行中的服务器，查看器的“Submit All Reviews”按钮会将 `feedback.json` 作为文件下载。你可以从那里读取它（可能需要先请求访问权限）。
- 打包功能正常——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过子进程使用 `claude -p`，而不是浏览器，但请务必等到你完全完成 skill 制作且用户认为其状态良好后再进行。
- **更新现有 skill**：用户可能要求你更新现有的 skill，而不是创建新的。请遵循上文 claude.ai 部分的更新指导。

---

## 参考文件

agents/ 目录包含专门 subagents 的说明。当你需要生成相关 subagent 时请阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲测 A/B 对比
- `agents/analyzer.md` — 如何分析一个版本为何胜过另一个版本

references/ 目录包含其他文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次重复核心循环以示强调：

- 弄清楚 skill 的用途
- 起草或编辑 skill
- 在测试提示上运行有权访问该 skill 的 claude
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终的 skill 并返回给用户。

如果你有 TodoList 之类的功能，请将这些步骤添加进去，以确保不会遗忘。如果你在 Cowork 中，请务必将“创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例”放入你的 TodoList，以确保执行。

祝好运！