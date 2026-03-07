---
name: skill-creator
description: 创建新技能，修改和改进现有技能，并衡量技能性能。当用户希望从头创建技能、编辑或优化现有技能、运行 evals 测试技能、通过方差分析对技能性能进行基准测试，或优化技能描述以提高触发准确性时使用。
---

# Skill Creator

一个用于创建新技能并对其进行迭代改进的技能。

总体而言，创建技能的过程如下：

- 确定你希望技能做什么，以及大致如何做
- 编写技能草稿
- 创建一些测试 Prompt，并在其上运行 claude-with-access-to-the-skill
- 帮助用户定性和定量地评估结果
  - 当运行在后台进行时，如果没有定量的 evals，请起草一些（如果已经有一些，你可以按原样使用，或者如果你觉得需要更改，则进行修改）。然后向用户解释它们（或者如果它们已经存在，解释那些已经存在的）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，并让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（同时也针对定量 benchmark 中显现的明显缺陷进行修改）
- 重复此过程直到满意为止
- 扩充测试集并尝试更大规模的测试

使用此技能时，你的任务是确定用户处于流程的哪个阶段，然后介入并帮助他们推进这些阶段。例如，他们可能会说“我想为 X 制作一个技能”。你可以帮助缩小他们所指的范围，编写草稿，编写测试用例，确定他们想要如何评估，运行所有 Prompt，并重复此过程。

另一方面，也许他们已经有了技能草稿。在这种情况下，你可以直接进入循环中的评估/迭代部分。

当然，你应该始终保持灵活，如果用户说“我不需要运行一堆评估，随便弄弄就行”，你可以照做。

然后，在技能完成后（同样，顺序是灵活的），你还可以运行技能描述改进器，我们有一个单独的脚本来优化技能的触发。

明白了吗？很好。

## 与用户沟通

Skill creator 可能会被对编程术语熟悉程度各异的人群使用。如果你还没听说过（你怎么可能听说过呢，这只是最近才开始的事情），现在的趋势是 Claude 的能力正在激励水管工打开他们的终端，父母和祖父母去谷歌搜索 "how to install npm"。另一方面，大多数用户可能都具有相当的计算机素养。

所以请注意上下文线索，以了解如何措辞你的沟通！在默认情况下，为了给你一些概念：

- “evaluation”和“benchmark”处于边缘地带，但可以使用
- 对于“JSON”和“assertion”，在使用前你需要看到用户明确知道这些是什么的线索，否则需要解释

如果你不确定，简要解释术语是可以的，如果你不确定用户是否能理解，可以随意用简短的定义来澄清术语。

---

## 创建技能

### 捕捉意图

首先理解用户的意图。当前的对话可能已经包含了用户想要捕获的 workflow（例如，他们说“把这个变成一个技能”）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前进行确认。

1. 这个技能应该使 Claude 能够做什么？
2. 这个技能应该在何时触发？（什么用户短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出（文件转换、数据提取、代码生成、固定 workflow 步骤）的技能受益于测试用例。具有主观输出（写作风格、艺术）的技能通常不需要它们。根据技能类型建议适当的默认设置，但让用户决定。

### 访谈与研究

主动询问有关边缘情况、输入/输出格式、示例文件、成功标准和依赖关系的问题。在这部分解决之前，等待编写测试 Prompt。

检查可用的 MCPs —— 如果有助于研究（搜索文档、查找类似技能、查找最佳实践），如果可用，通过 subagents 并行研究，否则在线研究。准备好上下文以减轻用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**: Skill 标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包括技能做什么以及何时使用它的具体上下文。所有“何时使用”的信息都放在这里，而不是正文中。注意：目前 Claude 倾向于“欠触发”技能——即在它们有用时不使用它们。为了解决这个问题，请让技能描述稍微“激进”一点。例如，与其写“如何构建一个简单快速的仪表盘来显示内部 Anthropic 数据。”，你可以写“如何构建一个简单快速的仪表盘来显示内部 Anthropic 数据。确保每当用户提到仪表盘、数据可视化、内部指标或想要显示任何类型的公司数据时使用此技能，即使他们没有明确要求‘仪表盘’。”
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

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

技能采用三级加载系统：
1. **Metadata** (名称 + 描述) - 始终位于上下文中（约 100 词）
2. **SKILL.md body** - 当技能触发时位于上下文中（理想情况下 <500 行）
3. **Bundled resources** - 按需加载（无限制，脚本无需加载即可执行）

这些字数统计是大致的，如有需要，可以适当增加。

**关键模式：**
- 将 SKILL.md 保持在 500 行以内；如果接近此限制，请增加一个层级，并明确指出使用该技能的模型下一步应去往何处进行跟进。
- 在 SKILL.md 中清晰引用文件，并提供关于何时读取它们的指导。
- 对于大型参考文件（>300 行），需包含目录。

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

#### 无惊吓原则

这一点不言而喻，但技能绝不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果对技能的内容进行描述，其意图不应让用户感到意外。切勿配合创建具有误导性的技能，或旨在协助未授权访问、数据窃取或其他恶意活动的技能。不过，诸如“扮演 XYZ”之类的情况是可以接受的。

#### 编写模式

在指令中优先使用祈使句。

**定义输出格式** - 您可以这样做：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有用。你可以按如下方式格式化（但如果示例中包含 "Input" 和 "Output"，你可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事情的重要性，以此代替生硬陈旧的 MUST。运用心智理论，尽量使技能具有通用性，而非局限于特定示例而过于狭隘。先起草初稿，然后以全新的视角审视并进行改进。

### 测试用例

编写完技能草稿后，构思 2-3 个切合实际的测试提示词——即真实用户真正会说出的内容。与用户分享这些提示词：[不必完全照搬此措辞] “我想尝试这几个测试用例。这些看起来没问题吗，还是你想增加更多？” 然后运行它们。

将测试用例保存到 `evals/evals.json`。暂不编写断言——仅编写提示词。你将在下一步运行过程中起草断言。

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

请参阅 `references/schemas.md` 获取完整的 schema（包括你稍后将添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的过程——请勿中途停止。切勿使用 `/skill-test` 或任何其他测试 skill。

将结果放入 `<skill-name>-workspace/` 中，作为 skill 目录的同级目录。在 workspace 内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代目录内，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——只需随用随建。

### 步骤 1：在同一轮次中启动所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮次中生成两个子 agent——一个带有 skill，一个不带。这一点很重要：不要先生成 with-skill 运行，然后再回头处理 baseline。一次性启动所有内容，以便它们大致同时完成。

**With-skill 运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同的 prompt，但 baseline 取决于上下文）：
- **创建新技能**：完全没有技能。相同的 prompt，无 skill path，保存到 `without_skill/outputs/`。
- **改进现有技能**：旧版本。在编辑之前，对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline subagent 指向该快照。保存到 `old_skill/outputs`。

为每个测试用例编写一个 `eval_metadata.json`（assertions 目前可以为空）。根据测试内容为每个 eval 赋予一个描述性名称——不要只是 "eval-0"。目录名称也使用该名称。如果本次迭代使用了新的或修改过的 eval prompts，请为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行期间，起草断言

不要只是等待运行结束——你可以充分利用这段时间。为每个测试用例起草定量的断言，并向用户解释这些断言。如果 `evals/evals.json` 中已经存在断言，请检查它们并解释其检查内容。

好的断言应当客观可验证且具有描述性的名称——它们应该在基准测试查看器（benchmark viewer）中清晰易读，以便浏览结果的人能立即理解每个断言检查的内容。主观技能（如写作风格、设计质量）更适合定性评估——不要强行对需要人工判断的内容使用断言。

断言起草完成后，请更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户说明他们将在查看器中看到的内容——包括定性输出和定量基准测试结果。

### 步骤 3：随着运行完成，捕获计时数据

当每个 subagent 任务完成时，你会收到一个包含 `total_tokens` 和 `duration_ms` 的通知。请立即将此数据保存到运行目录下的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知传递，且未在其他地方持久化。请在通知到达时逐个处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

一旦所有运行完成：

1. **对每次运行进行评分** —— 生成一个 grader subagent（或进行内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录下的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）—— 查看器依赖于这些确切的字段名称。对于可以通过编程方式检查的断言，请编写并运行脚本，而不是依靠肉眼检查 —— 脚本更快、更可靠，并且可以在迭代中复用。

2. **聚合为基准测试** —— 在 skill-creator 目录下运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

此操作会生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 `benchmark.json`，请参阅 `references/schemas.md` 以获取 viewer 所需的确切 schema。
将每个 with_skill 版本置于其 baseline 对应版本之前。

3. **执行分析师审查** —— 阅读 benchmark 数据并挖掘聚合统计可能掩盖的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results" 章节）了解具体关注点 —— 例如无论 skill 如何总是通过的断言（无区分度）、高方差评估（可能不稳定）以及 time/token 权衡。

4. **启动 viewer**，包含定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及以后的迭代，还需传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**协同工作 / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。当用户点击 "Submit All Reviews" 时，反馈将作为 `feedback.json` 文件下载。下载后，将 `feedback.json` 复制到工作空间目录中，以便下一次迭代时读取。

注意：请使用 generate_review.py 创建查看器；无需编写自定义 HTML。

5. **告知用户** 类似这样的话：“我已在您的浏览器中打开了结果。有两个标签页 —— 'Outputs' 允许您点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后，请回到这里告诉我。”

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给定的任务
- **Output**：技能生成的文件，尽可能在页面内渲染
- **Previous Output**（第 2 次及以后的迭代）：显示上一次迭代输出的折叠区域
- **Formal Grades**（如果运行了评分）：显示断言通过/失败的折叠区域
- **Feedback**：一个随输入自动保存的文本框
- **Previous Feedback**（第 2 次及以后的迭代）：他们上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每种配置的通过率、耗时和 Token 使用情况，以及每次评估的细分和分析师观察结果。

导航通过上一步/下一步按钮或方向键完成。完成后，他们点击 "Submit All Reviews"，这会将所有反馈保存到 `feedback.json`。

### 第 5 步：读取反馈

当用户告诉你他们完成时，读取 `feedback.json`：

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

空反馈意味着用户认为没有问题。重点改进用户提出具体不满的测试用例。

使用完毕后请终止 viewer server：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 Skill

这是循环的核心环节。你已经运行了测试用例，用户也审查了结果，现在你需要根据他们的反馈让 skill 变得更好。

### 如何思考改进

1. **从反馈中进行归纳。** 这里的核心大局是，我们试图创建可以在数百万个不同的 prompts 中使用（可能是字面意思，甚至更多，谁知道呢）的 skills。在这里，你和用户只针对几个例子反复迭代，因为这有助于加快速度。用户对这些例子了如指掌，评估新输出很快。但如果你们共同开发的 skill 只适用于那些例子，它就没用了。与其进行琐碎的过拟合修改，或者制定苛刻的强制性指令（MUST），如果遇到顽固的问题，你可以尝试另辟蹊径，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会找到绝佳方案。

2. **保持 prompt 精简。** 移除那些没有起到应有作用的内容。务必阅读执行记录，而不仅仅是最终输出——如果看起来 skill 让模型浪费大量时间做无产出的事情，你可以尝试去掉 skill 中导致这种情况的部分，看看会发生什么。

3. **解释“为什么”。** 尽力解释你要求模型做每件事背后的**原因**。如今的 LLMs *很聪明*。它们拥有良好的心智理论，当在良好的引导下，它们能超越机械指令，真正把事情做成。即使用户的反馈简短或沮丧，也要尝试真正理解任务，理解为什么用户写这些内容，以及他们实际写了什么，然后将这种理解传达指令中。如果你发现自己用全大写书写 ALWAYS 或 NEVER，或者使用超级僵硬的结构，那就是一个警示信号——如果可能，重新构建并解释推理过程，以便模型理解你要求的事情为何重要。这是一种更人性化、更强大、更有效的方法。

4. **寻找跨测试用例的重复工作。** 阅读测试运行的记录，注意 subagents 是否都独立编写了类似的辅助脚本或对某事采取了相同的多步骤方法。如果所有 3 个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这是一个强烈的信号，表明 skill 应该捆绑该脚本。编写一次，放在 `scripts/` 中，并告诉 skill 使用它。这可以节省每次未来的调用去重复造轮子。

这项任务非常重要（我们要在这里每年创造数十亿的经济价值！），你的思考时间不是阻碍；慢慢来，仔细斟酌。我建议先写一个修改草案，然后重新审视并进行改进。尽最大努力设身处地为用户着想，理解他们想要和需要什么。

### 迭代循环

改进 skill 之后：

1. 将你的改进应用到 skill 中
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建一个新的 skill，基线始终是 `without_skill`（无 skill）——这在迭代中保持不变。如果你正在改进现有的 skill，请根据判断选择合理的基线：用户带来的原始版本，或上一次迭代。
3. 启动 reviewer，并使用 `--previous-workspace` 指向上一次迭代
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复此过程

继续进行，直到：
- 用户表示满意
- 反馈全为空（一切看起来都很好）
- 你没有取得实质性的进展

---

## 高级：盲测对比

对于需要在 skill 的两个版本之间进行更严格比较的情况（例如，用户问“新版本真的更好吗？”），有一个盲测对比系统。请阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思想是：将两个输出提供给一个独立的 agent，而不告诉它哪个是哪个，让它判断质量。然后分析获胜者为何获胜。

这是可选的，需要 subagents，大多数用户不需要它。人工审查循环通常就足够了。

---

## Description 优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。在创建或改进 skill 后，提议优化 description 以获得更好的触发准确性。

### 步骤 1：生成触发测试用例

创建 20 个应该触发该 skill 的多样化用户 prompts。这些应涵盖不同的角度、意图和措辞。

### 步骤 2：生成负面测试用例

创建 20 个看起来相似但不应触发该 skill 的 prompts。这些是帮助模型学习边界的“棘手案例”。

### 步骤 3：测试与迭代

运行所有 40 个案例，检查 skill 是否正确触发。如果它在不该触发时触发了，或者在该触发时没触发，请优化 description。

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

查询内容必须真实，是 Claude Code 或 Claude.ai 用户实际会输入的内容。不要抽象的请求，而是具体、明确且包含丰富细节的请求。例如，文件路径、关于用户工作或情况的个人背景信息、列名和数值、公司名称、URL。以及少量的背景说明。部分查询可能使用小写，或包含缩写、拼写错误或口语化表达。混合使用不同的长度，重点关注边缘案例，而不是让它们过于直白（用户后续会有确认的机会）。

反例：`"Format this data"`，`"Extract text from PDF"`，`"Create a chart"`

正例：`"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

对于 **should-trigger** 查询（8-10 条），请考虑覆盖范围。需要包含表达相同意图的不同措辞——有些正式，有些随意。包括用户未明确指明 skill 或文件类型，但显然需要使用该 skill 的情况。加入一些非常规用例，以及该 skill 与其他 skill 存在竞争关系但应优先触发该 skill 的情况。

对于 **should-not-trigger** 查询（8-10 条），最有价值的是那些“近似匹配”——即与 skill 共享关键词或概念，但实际需求不同的查询。思考相邻领域、仅靠简单关键词匹配会误触发但不应触发的模糊措辞，以及查询涉及该 skill 的功能但上下文更适合使用其他工具的情况。

关键的一点是：不要让 should-not-trigger 查询显得明显不相关。将 `"Write a fibonacci function"` 作为 PDF skill 的反向测试太容易了——这没有任何测试价值。反向案例应具有真正的迷惑性。

### Step 2: 与用户一起审查

使用 HTML 模板向用户展示 eval set 以供审查：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → eval 项的 JSON 数组（无需引号——这是一个 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → skill 的名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → skill 的当前描述
3. 写入临时文件（例如 `/tmp/eval_review_<skill-name>.html`）并打开：`open /tmp/eval_review_<skill-name>.html`
4. 用户可以编辑查询、切换 should-trigger 状态、添加/删除条目，然后点击 "Export Eval Set"
5. 文件将下载到 `~/Downloads/eval_set.json` —— 请检查 Downloads 文件夹以获取最新版本，以防存在多个文件（例如 `eval_set (1).json`）

这一步至关重要——糟糕的 eval 查询会导致糟糕的描述。

### Step 3: 运行优化循环

告诉用户：“这需要一些时间——我会在后台运行优化循环并定期检查。”

将 eval set 保存到工作区，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（即支持当前会话的那个），以便触发测试与用户的实际体验相符。

在运行期间，定期查看输出尾部，向用户更新当前的迭代次数和得分情况。

这会自动处理完整的优化循环。它将 eval set 拆分为 60% 的训练集和 40% 的保留测试集，评估当前的 description（每个 query 运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败的情况提出改进建议。它会在训练集和测试集上重新评估每个新的 description，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON —— 该 description 是根据测试得分而非训练得分选出的，以避免过拟合。

### Skill 触发机制的工作原理

理解触发机制有助于设计更好的 eval queries。Skills 出现在 Claude 的 `available_skills` 列表中，包含其名称和 description，Claude 根据该 description 决定是否咨询 skill。重要的是要知道，Claude 只会为它无法轻易独立完成的任务咨询 skills —— 像“阅读此 PDF”这样简单的一步 query，即使 description 完美匹配，也可能不会触发 skill，因为 Claude 可以使用基本工具直接处理它们。当 description 匹配时，复杂、多步骤或专业的 query 会可靠地触发 skills。

这意味着你的 eval queries 应该足够充实，让 Claude 确实能从咨询 skill 中受益。像“读取文件 X”这样的简单 query 是糟糕的测试用例 —— 无论 description 质量如何，它们都不会触发 skills。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description` 并更新 skill 的 SKILL.md frontmatter。向用户展示修改前后的对比并报告得分。

---

### 打包与展示（仅当 `present_files` 工具可用时）

检查你是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，打包 skill 并将 .skill 文件展示给用户：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，告知用户生成的 `.skill` 文件路径，以便他们进行安装。

---

## Claude.ai 特定指令

在 Claude.ai 中，核心工作流保持不变（草稿 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有 subagents，某些机制有所变化。以下是需要调整的内容：

**运行测试用例**：没有 subagents 意味着无法并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后遵循其指令自行完成测试提示。请逐个执行。这不如独立的 subagents 严谨（因为是你编写的 skill 同时也是你在运行它，你拥有完整的上下文），但这作为一种健全性检查很有用——而且人工审查步骤可以弥补这一不足。跳过基线运行——只需使用 skill 按要求完成任务即可。

**审查结果**：如果你无法打开浏览器（例如 Claude.ai 的 VM 没有显示器，或者你在远程服务器上），请完全跳过浏览器审查器。相反，直接在对话中展示结果。对于每个测试用例，展示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告知用户位置，以便他们下载和检查。在对话中请求反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖于基线比较，在没有 subagents 的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，请求反馈——只是中间没有浏览器审查器。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），该工具仅在 Claude Code 中可用。如果你在 Claude.ai 上，请跳过此步骤。

**盲测对比**：需要 subagents。跳过。

**打包**：`package_skill.py` 脚本可在任何有 Python 和文件系统的地方运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能是要求你更新现有的 skill，而不是创建新的。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名和 `name` frontmatter 字段——保持不变。例如，如果安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **编辑前复制到可写位置。** 安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，并从副本打包。
- **如果手动打包，先暂存在 `/tmp/`**，然后复制到输出目录——直接写入可能会因权限问题而失败。

---

## Cowork 特定指令

如果你在 Cowork 中，主要需要注意以下几点：

- 你拥有 subagents，因此主工作流（并行生成测试用例、运行基线、评分等）都能正常工作。（但是，如果你遇到严重的超时问题，可以串行而非并行地运行测试提示。）
- 你没有浏览器或显示器，因此在生成 eval viewer 时，请使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击该链接在浏览器中打开 HTML。
- 无论出于何种原因，Cowork 设置似乎导致 Claude 不倾向于在运行测试后生成 eval viewer，因此再次重申：无论你在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成 eval viewer，以便在自行修改 skill 和尝试纠正之前让人类查看示例，使用 `generate_review.py`（而不是编写你自己的定制 HTML 代码）。提前抱歉，但我要在这里用全大写了：在自己评估输入之前，**先生成 EVAL VIEWER**。你希望尽快让用户看到它们！
- 反馈机制有所不同：由于没有运行中的服务器，查看器的 "Submit All Reviews" 按钮将下载 `feedback.json` 文件。然后你可以从那里读取它（你可能需要先请求访问权限）。
- 打包功能正常——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该能正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请保留到最后，直到你完全完成了 skill 的制作且用户认可其状态良好。
- **更新现有 skill**：用户可能是要求你更新现有的 skill，而不是创建新的。请遵循上文 claude.ai 部分的更新指引。

---

## 参考文件

`agents/` 目录包含专门 subagents 的指令。当你需要生成相关 subagent 时，请阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲测 A/B 对比
- `agents/analyzer.md` — 如何分析一个版本为何胜过另一个版本

`references/` 目录包含额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构。

---

再次重复这里的核心循环以示强调：

- 弄清楚 skill 的用途
- 起草或编辑 skill
- 在测试提示上运行带 skill 访问权限的 claude
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查
  - 运行定量评估
- 重复此过程直到你和用户都满意
- 打包最终的 skill 并返回给用户。

如果你有待办事项列表，请将这些步骤添加到其中。