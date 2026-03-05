---
name: skill-creator
description: 创建新技能，修改和改进现有技能，并衡量技能性能。当用户希望从头创建技能、更新或优化现有技能、运行 evals 测试技能、通过方差分析对技能性能进行基准测试，或优化技能描述以提高触发准确性时使用。
---

# Skill Creator

一个用于创建新技能并对其进行迭代改进的技能。

从宏观层面来看，创建技能的过程如下：

- 确定你希望该技能做什么，以及大致如何做
- 编写技能草稿
- 创建一些测试 Prompt，并在这些 Prompt 上运行 claude-with-access-to-the-skill
- 帮助用户定性和定量地评估结果
  - 当运行在后台进行时，如果没有定量 evals（如果有一些，你可以直接使用，或者觉得需要更改时进行修改），则起草一些。然后向用户解释它们（或者如果它们已经存在，解释那些现有的）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，并让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（以及如果定量基准测试中暴露出任何明显的缺陷）
- 重复直到满意为止
- 扩展测试集，并尝试更大规模的测试

使用此技能时，你的任务是确定用户处于此过程的哪个阶段，然后介入并帮助他们推进这些阶段。例如，他们可能会说“我想为 X 做一个技能”。你可以帮助缩小他们的意图范围，编写草稿，编写测试用例，确定他们想要如何评估，运行所有 Prompt，并重复此过程。

另一方面，也许他们已经有了技能草稿。在这种情况下，你可以直接进入循环的评估/迭代部分。

当然，你应该始终保持灵活，如果用户说“我不需要运行一堆评估，只要跟我一起凭感觉做就行”，那你可以照做。

然后，在技能完成后（同样，顺序是灵活的），你还可以运行技能描述改进器，我们有一个单独的脚本来优化技能的触发。

明白了吗？很好。

## 与用户沟通

Skill creator 可能会被对编程术语熟悉程度各异的人群使用。如果你还没听说（你怎么可能听说过，这只是最近才开始的事），现在有一种趋势，Claude 的能力正在激励水管工打开他们的终端，父母和祖父母去谷歌搜索“how to install npm”。另一方面，大多数用户可能都具备相当的计算机素养。

所以请注意上下文线索，以了解如何组织你的沟通语言！在默认情况下，为了给你一些概念：

- “evaluation”和“benchmark”属于边界词汇，但可以使用
- 对于“JSON”和“assertion”，在使用这些词而不加解释之前，你需要看到用户确实了解这些内容的明确线索

如果你不确定，可以简要解释术语，如果你不确定用户是否理解，可以随意用简短的定义来澄清术语。

---

## 创建技能

### 捕捉意图

首先理解用户的意图。当前的对话可能已经包含了用户想要捕捉的工作流（例如，他们说“把这个变成一个技能”）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的修正、观察到的输入/输出格式。用户可能需要填补信息空白，并应在进入下一步之前进行确认。

1. 这个技能应该让 Claude 能够做什么？
2. 这个技能应该何时触发？（用户的哪些短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认设置，但让用户决定。

### 访谈与研究

主动询问有关边缘情况、输入/输出格式、示例文件、成功标准和依赖关系的问题。在搞定这部分之前，不要急着编写测试 Prompt。

检查可用的 MCP —— 如果有助于研究（搜索文档、查找类似技能、查找最佳实践），如果可用，通过 subagent 并行研究，否则在线研究。准备好上下文以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发，做什么。这是主要的触发机制——包括技能的功能以及何时使用的具体上下文。所有“何时使用”的信息都放在这里，而不是正文中。注意：目前 Claude 倾向于“触发不足”——即在技能有用时不使用它们。为了解决这个问题，请让技能描述稍微“主动”一些。例如，与其写“如何构建一个简单快速的仪表板来显示内部 Anthropic 数据。”，不如写“如何构建一个简单快速的仪表板来显示内部 Anthropic 数据。确保每当用户提到仪表板、数据可视化、内部指标或想要显示任何类型的公司数据时使用此技能，即使他们没有明确要求‘仪表板’。”
- **compatibility**：所需工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### Skill 编写指南

#### 技能剖析

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
1. **元数据**（名称 + 描述）- 始终位于上下文中（约 100 词）
2. **SKILL.md 主体** - 每当 Skill 触发时位于上下文中（理想情况下 <500 行）
3. **捆绑资源** - 按需提供（无限制，脚本可在未加载的情况下执行）

这些字数统计为近似值，如有必要可以适当超出。

**关键模式：**
- 将 SKILL.md 保持在 500 行以内；若接近此限制，请增加额外的层级，并提供明确的指引，告知使用该 Skill 的模型下一步应转向何处进行后续操作。
- 在 SKILL.md 中清晰引用文件，并提供关于何时读取这些文件的指导
- 对于大型参考文件（>300 行），请包含目录

**领域组织**：当一个 Skill 支持多个领域/框架时，请按变体进行组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅读取相关的参考文件。

#### 无意外原则

这一点不言而喻，但技能绝不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果对技能的意图进行描述，其内容不应让用户感到意外。不要顺从创建误导性技能，或旨在协助未授权访问、数据窃取或其他恶意活动的请求。不过，诸如“扮演 XYZ”之类的请求是可以的。

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

**示例模式** - 包含示例非常有用。您可以像这样格式化（但如果示例中包含 "Input" 和 "Output"，您可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物的重要性，以此代替生硬且陈旧的“必须（MUST）”式指令。运用心理理论 (Theory of Mind)，尝试使技能具有通用性，而非仅局限于特定示例。先撰写一份草稿，随后以全新的视角审视并进行改进。

### 测试用例

编写完技能草稿后，构思 2-3 个逼真的测试提示词 (prompts)——即真实用户实际上会提出的请求。与用户分享这些内容：[不必完全照搬原话] “我想尝试这几个测试用例。这些看起来合适吗，还是您想添加更多？” 随后运行它们。

将测试用例保存至 `evals/evals.json`。暂不编写断言 (assertions)——仅需包含提示词。您将在下一步运行过程中起草断言。

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

请参阅 `references/schemas.md` 获取完整的 schema（包括稍后您将添加的 `assertions` 字段）。

## 运行与评估测试用例

本节是一个连续的过程——请勿中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放入 `<skill-name>-workspace/` 中，作为 skill 目录的同级目录。在工作空间内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，并在迭代目录内，为每个测试用例创建一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有这些目录——只需在过程中按需创建。

### 步骤 1：在同一轮中启动所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮中启动两个子 Agent——一个使用 skill，一个不使用。这一点很重要：不要先启动 with-skill 运行，然后再回来处理 baseline。一次性启动所有任务，以便它们大致同时完成。

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
- **Creating a new skill**：完全没有 skill。相同的 prompt，无 skill path，保存至 `without_skill/outputs/`。
- **Improving an existing skill**：旧版本。编辑前，对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline subagent 指向该快照。保存至 `old_skill/outputs/`。

为每个测试用例编写一个 `eval_metadata.json`（assertions 暂时可为空）。根据测试内容为每个 eval 赋予一个描述性名称 —— 不要仅命名为 “eval-0”。该名称也用作目录名。如果本次迭代使用新的或修改过的 eval prompts，请为每个新的 eval 目录创建这些文件 —— 不要假设它们会从之前的迭代中延续。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行期间，起草断言

不要只是等待运行结束——你可以充分利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释其检查内容。

好的断言应具备客观可验证性，并拥有描述性名称——它们应在基准查看器（benchmark viewer）中清晰易读，以便浏览结果的人能立即理解每个断言的检查内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的事物上。

断言起草完成后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们将在查看器中看到的内容——包括定性输出和定量基准。

### 步骤 3：随着运行完成，捕获计时数据

当每个 subagent 任务完成时，你会收到一个包含 `total_tokens` 和 `duration_ms` 的通知。请立即将此数据保存到运行目录下的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会 —— 它通过任务通知传递，并未在其他地方持久化。请在每个通知到达时立即处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动 viewer

一旦所有运行完成：

1. **对每次运行进行评分** —— 生成一个 grader subagent（或进行内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每个运行目录中的 `grading.json`。`grading.json` 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）—— viewer 依赖于这些精确的字段名称。对于可以通过编程检查的断言，请编写并运行脚本，而不是人工检查 —— 脚本更快、更可靠，并且可以跨迭代重用。

2. **聚合到 benchmark** —— 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每种 configuration 的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 查看 viewer 所需的确切 schema。
将每个 with_skill 版本置于其 baseline 对应版本之前。

3. **执行 analyst pass** —— 阅读 benchmark 数据并揭示 aggregate stats 可能隐藏的模式。请参阅 `agents/analyzer.md`（“Analyzing Benchmark Results”章节）了解需要关注的内容 —— 例如无论 skill 如何总是通过的 assertions（non-discriminating）、高方差 evals（possibly flaky）以及 time/token tradeoffs。

4. **启动 viewer**，同时加载 qualitative outputs 和 quantitative data：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及后续的迭代，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**Cowork / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。当用户点击 "Submit All Reviews" 时，反馈将作为 `feedback.json` 文件下载。下载后，将 `feedback.json` 复制到工作空间目录，以便下一次迭代读取。

注意：请使用 generate_review.py 创建查看器；无需编写自定义 HTML。

5. **告知用户** 类似于以下内容：“我已在你的浏览器中打开结果。有两个标签页 —— 'Outputs' 允许你点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后，请回到这里告诉我。”

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给定的任务
- **Output**：skill 生成的文件，尽可能内嵌渲染
- **Previous Output** (第 2+ 次迭代)：折叠部分，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠部分，显示断言通过/失败情况
- **Feedback**：一个随输入自动保存的文本框
- **Previous Feedback** (第 2+ 次迭代)：上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每种配置的通过率、耗时和 Token 使用情况，以及每次评估的细分数据和分析师观察。

通过上一个/下一个按钮或箭头键进行导航。完成后，他们点击 "Submit All Reviews"，这将所有反馈保存到 `feedback.json`。

### 第 5 步：读取反馈

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

反馈为空意味着用户认为没问题。将改进重点放在用户提出具体问题的测试用例上。

使用完毕后关闭 viewer server：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 Skill

这是循环的核心。你已经运行了测试用例，用户也已经审查了结果，现在你需要根据他们的反馈让 skill 变得更好。

### 如何思考改进

1.  **从反馈中进行归纳。** 这里的大局在于，我们正试图创建可以在许多不同的 prompts 下使用数百万次（也许是字面意思，甚至更多，谁知道呢）的 skills。在这里，你和用户仅仅是在少数几个例子上反复迭代，因为这有助于加快速度。用户对这些例子了如指掌，评估新输出非常快。但是，如果你和用户共同开发的 skill 仅适用于那些例子，那它就毫无用处。与其进行繁琐且过拟合（overfitty）的修改，或者制定压迫性限制性的 MUST 指令，如果遇到顽固的问题，你可以尝试另辟蹊径，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会发现很棒的东西。

2.  **保持 prompt 精简。** 移除那些没有发挥作用的内容。务必阅读 transcripts，而不仅仅是最终输出 —— 如果看起来 skill 让模型浪费大量时间做无用功，你可以尝试删除 skill 中导致它这样做的部分，看看会发生什么。

3.  **解释“为什么”。** 尽力解释你要求模型做每件事背后的**原因**。现在的 LLMs *很聪明*。它们拥有良好的心智理论，当被赋予一个好的 harness 时，它们可以超越死板的指令，真正把事情做成。即使用户的反馈简短或带有沮丧情绪，也要试着真正理解任务，理解为什么用户会写下他们所写的内容，以及他们实际写了什么，然后将这种理解传递到指令中。如果你发现自己用全大写书写 ALWAYS 或 NEVER，或者使用超级僵化的结构，那就是一个警示信号 —— 如果可能的话，重新构建并解释推理过程，以便模型理解你要求的事情为何重要。这是一种更人性化、强大且有效的方法。

4.  **寻找跨测试用例的重复工作。** 阅读测试运行的 transcripts，注意 subagents 是否都独立编写了相似的辅助脚本或对某事采取了相同的多步骤方法。如果所有 3 个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明 skill 应该打包那个脚本。编写一次，放在 `scripts/` 中，并告诉 skill 去使用它。这能节省未来的每次调用，避免重复造轮子。

这项任务非常重要（我们要在此创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；慢慢来，真正仔细地权衡。我建议写一个修改草案，然后重新审视它并进行改进。尽最大努力进入用户的头脑，理解他们想要和需要什么。

### 迭代循环

改进 skill 之后：

1.  将你的改进应用到 skill 中
2.  将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建一个新的 skill，基线始终是 `without_skill`（无 skill）—— 这在迭代过程中保持不变。如果你正在改进现有的 skill，请根据判断选择作为基线的内容：用户带来的原始版本，还是上一次迭代。
3.  启动 reviewer，并使用 `--previous-workspace` 指向上一次迭代
4.  等待用户审查并告知你他们已完成
5.  阅读新反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈全为空（一切看起来都很好）
- 你没有取得实质性进展

---

## 高级：盲测比较

对于想要在 skill 的两个版本之间进行更严格比较的情况（例如，用户问“新版本真的更好吗？”），有一个盲测比较系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思想是：将两个输出提供给一个独立的 agent，而不告诉它哪个是哪个，让它判断质量。然后分析获胜者为何获胜。

这是可选的，需要 subagents，大多数用户不需要它。人工审查循环通常就足够了。

---

## Description 优化

`SKILL.md` frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。在创建或改进 skill 后，提议优化 description 以获得更好的触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个 eval queries —— should-trigger 和 should-not-trigger 的混合。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

查询内容必须真实，符合 Claude Code 或 Claude.ai 用户的实际输入习惯。避免抽象请求，应包含具体细节，明确且详尽。例如：文件路径、用户工作或处境的个人背景、列名与数值、公司名称、URL，以及少量背景故事。部分查询可以使用小写，或包含缩写、拼写错误及口语化表达。混合使用不同长度，侧重于边缘情况而非显而易见的案例（用户后续有机会确认）。

差: `"Format this data"`, `"Extract text from PDF"`, `"Create a chart"`

好: `"ok 我老板刚发给我这个 xlsx 文件（在我的下载文件夹里，名字大概叫 'Q4 sales final FINAL v2.xlsx'），她想让我加一列显示利润率百分比。收入好像在 C 列，成本在 D 列"`

对于 **should-trigger** 查询（8-10条），请考虑覆盖范围。需涵盖同一意图的不同表述方式——部分正式，部分随意。包含用户未明确提及 skill 或文件类型，但显然需要使用该功能的案例。加入一些罕见用例，以及该 skill 与其他 skill 存在竞争但应优先触发该 skill 的场景。

对于 **should-not-trigger** 查询（8-10条），最有价值的是“近似匹配”——即与 skill 共享关键词或概念，但实际需求不同的查询。考虑相邻领域、容易因简单关键词匹配而误触发的歧义表述，以及查询涉及该 skill 功能但另一工具更适用的场景。

关键注意事项：切勿使 should-not-trigger 查询显而易见地无关。作为 PDF skill 的反向测试，`"Write a fibonacci function"` 过于简单——这没有任何测试价值。反向案例应具有真正的迷惑性。

### 步骤 2：与用户共同审查

使用 HTML 模板向用户展示评估集以供审查：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → eval 项目的 JSON 数组（无需引号包裹——这是 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → skill 名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → skill 当前描述
3. 写入临时文件（例如 `/tmp/eval_review_<skill-name>.html`）并打开：`open /tmp/eval_review_<skill-name>.html`
4. 用户可编辑查询、切换 should-trigger 状态、添加/删除条目，然后点击 "Export Eval Set"
5. 文件将下载至 `~/Downloads/eval_set.json` ——请检查 Downloads 文件夹获取最新版本，以防存在多个文件（例如 `eval_set (1).json`）

此步骤至关重要——糟糕的 eval 查询会导致糟糕的描述。

### 步骤 3：运行优化循环

告知用户：“这需要一些时间——我将在后台运行优化循环并定期检查进度。”

将 eval 集保存至工作区，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的 model ID（驱动当前会话的那个），以便触发测试能匹配用户的实际体验。

运行期间，定期查看输出，向用户更新当前迭代进度及分数情况。

此过程自动处理完整的优化循环。它将 eval set 分为 60% train 和 40% held-out test，评估当前 description（每条查询运行 3 次以获取可靠的触发率），随后调用 Claude 并利用 extended thinking 根据失败情况提出改进建议。它会在 train 和 test 上重新评估每个新 description，最多迭代 5 次。完成后，它会在浏览器中打开 HTML 报告显示每次迭代的结果，并返回包含 `best_description` 的 JSON —— 该描述依据 test 分数而非 train 分数选出，以避免 overfitting。

### Skill 触发机制

理解触发机制有助于设计更好的 eval queries。Skills 会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 根据该描述决定是否调用 skill。关键在于，Claude 仅会在遇到其自身无法轻松处理的任务时才会调用 skills —— 像 "read this PDF" 这类简单的单步查询，即便描述完美匹配也可能不会触发 skill，因为 Claude 可以直接使用基础工具处理。而复杂、多步骤或专业化的查询，只要描述匹配，就能可靠地触发 skills。

这意味着你的 eval queries 必须足够充实，使 Claude 确实能从调用 skill 中受益。像 "read file X" 这样的简单查询是糟糕的测试用例 —— 无论描述质量如何，它们都不会触发 skills。

### 第 4 步：应用结果

从 JSON 输出中提取 `best_description` 并更新 skill 的 SKILL.md frontmatter。向用户展示修改前后的对比并报告分数。

---

### 打包与展示（仅当 `present_files` 工具可用时）

检查你是否可以使用 `present_files` 工具。若不可用，则跳过此步。若可用，则打包 skill 并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，请指示用户查看生成的 `.skill` 文件路径，以便他们进行安装。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流是一样的（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有 subagents，部分机制有所变化。以下是需要调整的地方：

**运行测试用例**：没有 subagents 意味着没有并行执行。对于每个测试用例，阅读 skill 的 SKILL.md，然后按照其指令亲自完成测试提示词。逐一进行。这不如独立的 subagents 严谨（你编写了 skill，同时也在运行它，因此你拥有完整的上下文），但这是一种有用的健全性检查——而且人工审查步骤可以弥补这一点。跳过基准运行——直接使用 skill 按要求完成任务。

**审查结果**：如果你无法打开浏览器（例如，Claude.ai 的 VM 没有显示器，或者你在远程服务器上），请完全跳过浏览器审查。取而代之的是，直接在对话中展示结果。对于每个测试用例，展示提示词和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告知用户位置，以便他们下载和检查。直接询问反馈：“这个看起来怎么样？有什么要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖于基准比较，在没有 subagents 的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：和之前一样——改进 skill，重新运行测试用例，寻求反馈——只是中间没有浏览器审查。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），该工具仅在 Claude Code 中可用。如果你在 Claude.ai 上，请跳过。

**盲测比较**：需要 subagents。跳过。

**打包**：`package_skill.py` 脚本可在任何具备 Python 和文件系统的地方运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

---

## Cowork 特定说明

如果你在 Cowork 中，主要需要了解的是：

- 你拥有 subagents，因此主工作流（并行生成测试用例、运行基准、评分等）都能正常工作。（但是，如果你遇到严重的超时问题，可以串行而非并行地运行测试提示词。）
- 你没有浏览器或显示器，因此在生成 eval viewer 时，请使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击该链接在浏览器中打开 HTML。
- 无论出于何种原因，Cowork 环境似乎会让 Claude 在运行测试后不愿意生成 eval viewer，所以再次重申：无论你是在 Cowork 还是 Claude Code 中，运行测试后，在亲自修改 skill 并尝试纠正之前，你应该始终使用 `generate_review.py` 生成 eval viewer 供人类查看示例（不要编写你自己的自定义 HTML 代码）。提前抱歉，但我要在这里用全大写强调：在你自己评估输入之*前*先生成 EVAL VIEWER。你要尽快把它们展示给人类！
- 反馈机制有所不同：由于没有运行服务器，查看器的“Submit All Reviews”按钮会将 `feedback.json` 作为文件下载。然后你可以从那里读取它（你可能需要先请求访问权限）。
- 打包功能正常工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该能正常工作，因为它通过子进程使用 `claude -p`，而不是浏览器，但请在完全完成 skill 制作且用户认为状态良好后再进行此操作。

---

## 参考文件

agents/ 目录包含专用 subagents 的指令。当需要生成相关 subagent 时请阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲测 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本为何优于另一个版本

references/ 目录包含额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次重复核心循环以示强调：

- 弄清楚 skill 的用途
- 起草或编辑 skill
- 在测试提示词上运行有权访问该 skill 的 claude
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意为止
- 打包最终的 skill 并将其返回给用户。

如果你有待办事项列表（TodoList），请将这些步骤添加进去，以确保不会遗忘。如果你在 Cowork 中，请务必将“创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类审查测试用例”放入你的 TodoList，以确保执行。

祝好运！