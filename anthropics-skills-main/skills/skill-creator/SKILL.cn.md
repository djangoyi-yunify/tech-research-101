---
name: skill-creator
description: 创建新 skill，修改和改进现有 skill，并衡量 skill 的性能。当用户希望从零开始创建 skill、更新或优化现有 skill、运行 evals 测试 skill、通过方差分析基准测试 skill 性能，或优化 skill 描述以提高触发准确性时使用。
---

# Skill Creator

一个用于创建新 skill 并对其进行迭代改进的 skill。

从宏观层面来看，创建 skill 的过程如下：

- 确定 skill 的功能及其大致实现方式
- 编写 skill 草稿
- 创建一些测试 prompt，并在其上运行 claude-with-access-to-the-skill
- 帮助用户定性和定量地评估结果
  - 当运行在后台进行时，如果没有现成的定量 evals，请起草一些（如果已有，可以按原样使用，或者如果你觉得需要更改，也可以修改）。然后向用户解释它们（或者如果它们已经存在，解释现有的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，并让他们查看定量指标
- 根据用户对结果评估的反馈重写 skill（以及从定量基准测试中发现的任何明显缺陷）
- 重复直到满意为止
- 扩大测试集并在更大规模上再次尝试

使用此 skill 时，你的任务是确定用户处于流程中的哪个阶段，然后介入并帮助他们推进这些阶段。例如，用户可能会说“我想为 X 创建一个 skill”。你可以帮助明确他们的具体意图，撰写草稿，编写测试用例，确定他们想要如何评估，运行所有 prompt，然后重复此过程。

另一方面，也许他们已经有了 skill 的草稿。在这种情况下，你可以直接进入循环中的评估/迭代环节。

当然，你应该始终保持灵活，如果用户表示“我不需要运行一堆评估，随便聊聊就行”，那你也可以照做。

然后，在 skill 完成后（同样，顺序是灵活的），你还可以运行 skill description improver（我们有一个专门的脚本），以优化 skill 的触发。

明白了吗？好的。

## 与用户沟通

Skill creator 可能会被对编程术语熟悉程度各异的人群使用。如果你还没听说过（你怎么可能听说过，毕竟这只是最近才开始的事情），现在有一种趋势，Claude 的强大能力正在激励水管工打开终端，父母和祖父母们在 Google 上搜索“how to install npm”。另一方面，大多数用户可能都具备相当的电脑素养。

因此，请注意语境线索，以确定如何组织你的语言！在默认情况下，给你一些参考：

- “evaluation”和“benchmark”处于临界点，但可以使用
- 对于“JSON”和“assertion”，在使用前你需要确认用户确实了解这些概念

如果你不确定，可以简要解释术语；如果你拿不准用户是否能理解，可以用简短的定义来澄清术语。

---

## 创建 skill

### 捕获意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的 workflow（例如，他们说“把这个变成一个 skill”）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的修正、观察到的输入/输出格式。用户可能需要填补空白，并应在进入下一步之前进行确认。

1. 这个 skill 应该让 Claude 能够做什么？
2. 这个 skill 何时应该触发？（什么用户短语/语境）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证 skill 是否有效？具有客观可验证输出的 skill（文件转换、数据提取、代码生成、固定的 workflow 步骤）受益于测试用例。具有主观输出的 skill（写作风格、艺术）通常不需要。根据 skill 类型建议合适的默认设置，但让用户做决定。

### 访谈与研究

主动询问有关边缘情况、输入/输出格式、示例文件、成功标准和依赖项的问题。在弄清楚这部分之前，暂缓编写测试 prompt。

检查可用的 MCP —— 如果对研究有用（搜索文档、查找类似的 skill、查询最佳实践），如果可用，通过 subagent 并行研究，否则内联进行。准备好上下文以减轻用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：Skill 标识符
- **description**：何时触发，做什么。这是主要的触发机制——包括 skill 的功能以及何时使用它的具体语境。所有“何时使用”的信息都放在这里，而不是正文中。注意：目前 Claude 有一种“触发不足”的倾向——即在 skill 有用时不使用它们。为了解决这个问题，请让 skill 描述稍微“强势”一点。例如，与其写“如何构建一个简单快速的 dashboard 来显示 Anthropic 内部数据”，不如写“如何构建一个简单快速的 dashboard 来显示 Anthropic 内部数据。每当用户提到 dashboard、数据可视化、内部指标，或想要显示任何类型的公司数据时，务必使用此 skill，即使他们没有明确要求‘dashboard’。”
- **compatibility**：所需工具、依赖项（可选，很少需要）
- **skill 的其余部分 :)**

### Skill 编写指南

#### Skill 的结构

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
2. **SKILL.md body** - 当 skill 触发时位于上下文中（理想情况 <500 行）
3. **Bundled resources** - 按需加载（无限制，脚本无需加载即可执行）

这些字数仅供参考，如有需要可适当增加。

**关键模式：**
- 保持 SKILL.md 在 500 行以内；若接近此限制，请增加一个层级结构，并提供明确指引，说明使用该 skill 的 model 接下来应转向何处进行后续操作。
- 在 SKILL.md 中清晰引用文件，并提供关于何时读取这些文件的指引
- 对于大型参考文件（>300 行），请包含目录

**Domain organization**：当 skill 支持多个 domains/frameworks 时，请按变体进行组织：

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

这一点不言而喻，但技能绝不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果对技能内容进行描述，其意图不应让用户感到意外。不要配合创建误导性技能，或旨在协助未授权访问、数据窃取或其他恶意活动的请求。不过，诸如“扮演 XYZ”之类的内容是可以的。

#### 写作模式

指令中优先使用祈使句形式。

**定义输出格式** - 你可以这样做：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例非常有帮助。您可以按以下方式格式化（但如果示例中包含 "Input" 和 "Output"，您可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为何重要，以代替生硬陈旧的“MUST”指令。运用心智理论，尝试让技能具备通用性，而非仅局限于特定示例。先起草一份草稿，然后以全新的视角审视并加以改进。

### 测试用例

编写完技能草稿后，设计 2-3 个切合实际的测试提示词——即真实用户实际会说的那种内容。与用户分享：[你不必完全使用这段原话] “我想尝试这几个测试用例。它们看起来合适吗，还是你想再增加一些？” 然后运行它们。

将测试用例保存到 `evals/evals.json`。暂不编写断言（assertions）——仅编写提示词即可。你将在下一步运行期间起草断言。

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

请参阅 `references/schemas.md` 查看完整 schema（包括稍后将添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的过程——不要中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放在技能目录同级的 `<skill-name>-workspace/` 中。在该 workspace 内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，而在迭代内部，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有这些——只需随着进度创建目录。

### 步骤 1：在同一轮次中生成所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮次中生成两个 subagent —— 一个使用 skill，一个不使用。这点很重要：不要先生成 with-skill 运行，然后再回头处理 baseline。同时启动所有任务，以便它们能在大致相同的时间完成。

**With-skill 运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline 运行**（提示词相同，但 baseline 取决于上下文）：
- **创建新 skill**：完全没有 skill。同样的提示词，没有 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：旧版本。编辑之前，对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将 baseline subagent 指向该快照。保存到 `old_skill/outputs`。

为每个测试用例编写一个 `eval_metadata.json`（断言暂时可以为空）。根据测试内容为每个 eval 起一个描述性的名称——不仅仅是 "eval-0"。目录名也使用此名称。如果本次迭代使用了新的或修改过的 eval prompts，请为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤 2：当 runs 正在进行时，起草 assertions

不要只是干等 runs 完成——你可以充分利用这段时间。为每个 test case 起草定量的 assertions，并向用户解释它们。如果 `evals/evals.json` 中已存在 assertions，请审查它们并解释它们检查的内容。

优秀的 assertions 应当客观可验证，并具有描述性名称——它们应在 benchmark viewer 中清晰易读，以便浏览结果的人立即理解每项检查的内容。主观技能（写作风格、设计质量）更适合进行定性评估——不要将 assertions 强加于需要人工判断的事物上。

assertions 起草完成后，请更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们将在 viewer 中看到的内容——包括定性输出和定量 benchmark。

### 步骤 3：当 runs 完成时，捕获计时数据

当每个 subagent task 完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。请立即将此数据保存到 run 目录下的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——数据通过任务通知传递，且未在其他地方持久化。请在每个通知到达时立即处理，而非尝试批量处理。

### Step 4: 评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分** —— 生成一个 grader subagent（或进行内联评分）来读取 `agents/grader.md`，并根据输出评估每个断言。将结果保存到每个运行目录下的 `grading.json` 中。`grading.json` 中的 expectations 数组必须使用字段 `text`、`passed` 和 `evidence`（而非 `name`/`met`/`details` 或其他变体）—— 查看器依赖于这些确切的字段名称。对于可通过编程方式检查的断言，请编写并运行脚本，而非人工查看 —— 脚本更快、更可靠，且可跨迭代复用。

2. **聚合为 benchmark** —— 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，其中包含每种配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 以了解查看器所需的确切 schema。
将每个 with_skill 版本置于其 baseline 对应版本之前。

3. **进行一轮分析师审查** —— 阅读基准测试数据并揭示汇总统计数据可能隐藏的模式。请参阅 `agents/analyzer.md`（“Analyzing Benchmark Results”章节）了解需要关注的内容——例如无论 skill 如何总是通过的断言、高方差评估（可能不稳定）以及 time/token 权衡。

4. **启动查看器**，同时加载定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及后续迭代，还需传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 写入一个独立的 HTML 文件，而不是启动服务器。当用户点击 "Submit All Reviews" 时，反馈将作为 `feedback.json` 文件下载。下载后，将 `feedback.json` 复制到 workspace 目录中，以便在下一次迭代中读取。

注意：请使用 generate_review.py 创建查看器；无需编写自定义 HTML。

5. **告诉用户** 类似这样的话：“我已经在浏览器中打开了结果。有两个标签页 —— 'Outputs' 允许你点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较结果。完成后，回到这里告诉我。”

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给定的任务
- **Output**：skill 生成的文件，尽可能内联渲染
- **Previous Output**（第 2 次及后续迭代）：显示上一次迭代输出的折叠区域
- **Formal Grades**（如果运行了评分）：显示断言通过/失败的折叠区域
- **Feedback**：输入时自动保存的文本框
- **Previous Feedback**（第 2 次及后续迭代）：上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每种配置的通过率、耗时和 token 使用情况，以及单次评估的细分数据和分析师观察。

导航通过上一个/下一个按钮或方向键完成。完成后，他们点击 "Submit All Reviews"，这会将所有反馈保存到 `feedback.json`。

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

反馈为空意味着用户认为没问题。

请将改进重点放在用户提出具体不满的 test cases 上。

使用完毕后，请终止 viewer server：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 Skill

这是循环的核心。你已经运行了测试用例，用户也审查了结果，现在你需要根据他们的反馈让 skill 变得更好。

### 如何思考改进

1. **从反馈中进行归纳。** 这里的宏观目标是我们试图创建可以在数百万次（可能是字面意义上的，甚至更多，谁知道呢）不同的 prompt 中使用的 skill。在这里，你和用户仅仅基于几个示例反复迭代，因为这有助于加快速度。用户对这些示例了如指掌，评估新输出对他们来说很快。但是，如果你和用户共同开发的 skill 仅适用于那些示例，那它就是无用的。与其进行繁琐的过拟合修改，或添加苛刻的限制性 MUST 指令，如果存在某个顽固问题，你可以尝试扩展思路，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会发现很棒的东西。

2. **保持 Prompt 精简。** 移除那些没有发挥作用的冗余内容。务必阅读运行记录，而不仅仅是最终输出——如果看起来 skill 正在让 model 浪费大量时间做无用功，你可以尝试删掉导致这种情况的 skill 部分，看看会发生什么。

3. **解释“为什么”。** 尽力解释你要求 model 做每件事背后的**原因**。如今的 LLM *非常聪明*。它们拥有良好的心智理论，当被赋予一个良好的机制时，它们能超越死板的指令并真正把事情做成。即使来自用户的反馈简短或带有挫败感，也要尝试真正理解任务，理解用户为什么写下他们所写的内容，以及他们实际上写了什么，然后将这种理解传达到的指令中。如果你发现自己在用全大写字母书写 ALWAYS 或 NEVER，或者使用超级僵硬的结构，那是一个危险信号——如果可能的话，重新构建并解释推理过程，让 model 理解你要求的事情为何重要。这是一种更人性化、更强大且更有效的方法。

4. **寻找跨测试用例的重复工作。** 阅读测试运行的记录，注意 subagent 是否都独立编写了相似的辅助脚本或对某事采取了相同的多步骤方法。如果所有 3 个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明该 skill 应该打包该脚本。编写一次，放入 `scripts/`，并告诉 skill 去使用它。这能节省未来每次调用的重复造轮子成本。

这项任务相当重要（我们正试图在这里创造每年数十亿美元的经济价值！），你的思考时间并不是瓶颈；慢慢来，仔细斟酌。我建议先写一份修改草案，然后重新审视并进行改进。真正尽力进入用户的脑海，理解他们想要和需要什么。

### 迭代循环

改进 skill 后：

1. 将你的改进应用到 skill 中
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建一个新的 skill，基线始终是 `without_skill`（无 skill）——这在迭代过程中保持不变。如果你正在改进现有的 skill，请根据判断决定什么作为基线有意义：用户带来的原始版本，还是上一次迭代。
3. 启动 reviewer 并使用 `--previous-workspace` 指向上一次迭代
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复此过程

持续进行直到：
- 用户表示满意
- 反馈全为空（一切看起来都很好）
- 你没有取得实质性的进展

---

## 高级：盲测对比

对于需要更严格比较两个 skill 版本的情况（例如，用户问“新版本真的更好吗？”），有一个盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出提供给一个独立的 agent，但不告诉它哪个是哪个，让它评判质量。然后分析获胜者为何获胜。

这是可选的，需要 subagent，大多数用户不需要。人工审查循环通常就足够了。

---

## Description 优化

SKILL.md 前言中的 description 字段是决定 Claude 是否调用 skill 的主要机制。在创建或改进 skill 后，主动提议优化 description 以获得更好的触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应触发和不应触发的查询。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

查询内容必须真实，且是 Claude Code 或 Claude.ai 用户实际会输入的内容。不要抽象的请求，而应是具体、特定且包含丰富细节的请求。例如，文件路径、关于用户工作或处境的个人背景信息、列名和数值、公司名称、URL。稍微带点背景故事。有些可能使用小写，或包含缩写、拼写错误或口语化表达。混合使用不同的长度，并关注边缘情况，而不是让它们一目了然（用户将有机会确认这些内容）。

差：`"Format this data"`，`"Extract text from PDF"`，`"Create a chart"`

好：`"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

对于 **should-trigger** 查询（8-10 条），请考虑覆盖范围。需要涵盖同一意图的不同表述方式——有些正式，有些随意。包括用户未明确指出 Skill 或文件类型，但显然需要使用该 Skill 的情况。加入一些非常见用例，以及该 Skill 与其他 Skill 存在竞争关系但应当优先触发的情况。

对于 **should-not-trigger** 查询（8-10 条），最有价值的是那些“近似匹配”的情况——即与 Skill 共享关键词或概念，但实际上需求不同的查询。思考相邻领域、简单关键词匹配会触发但不应触发的歧义表述，以及查询涉及该 Skill 功能但其他工具更适用的场景。

关键的避免事项：不要让 should-not-trigger 查询看起来显而易见地无关。针对 PDF Skill 的反例测试，如果用 "Write a fibonacci function" 就太简单了——这测试不出任何东西。负面案例应具有真正的迷惑性。

### Step 2: 与用户一起审查

使用 HTML 模板向用户展示评估集以供审查：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → eval 项的 JSON 数组（不加引号——这是一个 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → Skill 的名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → Skill 的当前描述
3. 写入临时文件（例如 `/tmp/eval_review_<skill-name>.html`）并打开它：`open /tmp/eval_review_<skill-name>.html`
4. 用户可以编辑查询、切换 should-trigger 状态、添加/删除条目，然后点击 "Export Eval Set"
5. 文件将下载至 `~/Downloads/eval_set.json` —— 请检查 Downloads 文件夹中的最新版本，以防存在多个文件（例如 `eval_set (1).json`）

这一步至关重要——糟糕的 eval 查询会导致糟糕的描述。

### Step 3: 运行优化循环

告诉用户：“这需要一些时间——我将在后台运行优化循环并定期检查。”

将评估集保存到工作区，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（支持当前会话的那个），以便触发测试与用户的实际体验相匹配。

在运行期间，定期查看输出以向用户更新当前迭代次数和分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 的训练集和 40% 的留出测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用具有 extended thinking 能力的 Claude，根据失败的情况提出改进建议。它会在训练集和测试集上重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON —— 该描述是根据测试分数而非训练分数选择的，以避免过拟合。

### Skill 触发机制原理

理解触发机制有助于设计更好的评估查询。Skill 会出现在 Claude 的 `available_skills` 列表中，包含其名称和描述，Claude 根据该描述决定是否咨询 skill。重要的是要知道，Claude 只会为它无法轻松独立处理的任务咨询 skill —— 诸如“阅读此 PDF”之类的简单单步查询可能不会触发 skill，即使描述完美匹配，因为 Claude 可以使用基本工具直接处理它们。当描述匹配时，复杂、多步骤或专门的查询会可靠地触发 skill。

这意味着你的评估查询应该足够充实，以便 Claude 确实能从咨询 skill 中受益。像“读取文件 X”这样的简单查询是糟糕的测试用例 —— 无论描述质量如何，它们都不会触发 skill。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description` 并更新 skill 的 SKILL.md frontmatter。向用户展示修改前后的对比并报告分数。

---

### 打包并展示（仅当 `present_files` 工具可用时）

检查你是否有权使用 `present_files` 工具。如果没有，请跳过此步骤。如果有，请打包 skill 并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，指引用户找到生成的 `.skill` 文件路径，以便他们进行安装。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流程保持不变（草稿 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有 subagents，某些机制需要调整。以下是需要适配的内容：

**运行测试用例**：没有 subagents 意味着无法并行执行。对于每个测试用例，请阅读 skill 的 SKILL.md，然后按照其指令自行完成测试提示。请逐个执行。这比独立的 subagents 严谨性稍差（因为您编写了 skill 同时也在运行它，因此您拥有完整的上下文），但这是一种有用的健全性检查 —— 而且人工审查步骤可以弥补这一不足。跳过基线运行 —— 仅使用 skill 按要求完成任务。

**审查结果**：如果您无法打开浏览器（例如 Claude.ai 的虚拟机没有显示器，或者您在远程服务器上），请完全跳过浏览器审查步骤。相反，直接在对话中展示结果。对于每个测试用例，展示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），请将其保存到文件系统并告知用户位置，以便他们下载和检查。在对话中直接征求反馈：“看起来怎么样？有什么要修改的地方吗？”

**基准测试**：跳过定量基准测试 —— 它依赖于基线比较，而在没有 subagents 的情况下这种比较没有意义。重点收集用户的定性反馈。

**迭代循环**：与之前相同 —— 改进 skill，重新运行测试用例，征求反馈 —— 只是中间没有浏览器审查步骤。如果您有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：本节需要 `claude` CLI 工具（具体为 `claude -p`），该工具仅在 Claude Code 中可用。如果您在 Claude.ai 上，请跳过此步骤。

**盲测对比**：需要 subagents。请跳过。

**打包**：`package_skill.py` 脚本可在任何具备 Python 和文件系统的环境中运行。在 Claude.ai 上，您可以运行它，用户可以下载生成的 `.skill` 文件。

---

## Cowork 特定说明

如果您在 Cowork 中，主要需要了解以下几点：

- 您拥有 subagents，因此主工作流程（并行生成测试用例、运行基线、评分等）均有效。（但是，如果遇到严重的超时问题，可以串联运行测试提示而非并行。）
- 您没有浏览器或显示器，因此在生成评估查看器时，请使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，供用户点击在浏览器中打开该 HTML 文件。
- 无论出于何种原因，Cowork 环境似乎会让 Claude 在运行测试后不愿意生成评估查看器，所以再次重申：无论您是在 Cowork 还是 Claude Code 中，运行测试后，您都应该始终生成评估查看器供人类查看示例，然后再自己修订 skill 并尝试进行修正，请使用 `generate_review.py`（而不是编写您自己定制化的 html 代码）。提前抱歉，但我这里要全大写强调：在自己评估输入*之前*，**生成评估查看器**。您需要尽快将其展示给人类！
- 反馈机制有所不同：由于没有运行中的服务器，查看器的“Submit All Reviews”按钮会将 `feedback.json` 作为文件下载。随后您可以读取该文件（可能需要先请求访问权限）。
- 打包功能正常 —— `package_skill.py` 只需要 Python 和文件系统即可。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该能正常工作，因为它通过子进程使用 `claude -p` 而非浏览器，但请在您完全完成 skill 制作且用户认可其状态良好之后再进行此操作。

---

## 参考文件

`agents/` 目录包含专门 subagents 的说明。当需要生成相关 subagent 时请阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲测 A/B 对比
- `agents/analyzer.md` — 如何分析一个版本为何优于另一个版本

`references/` 目录包含其他文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次重复核心循环以示强调：

- 弄清楚 skill 的用途
- 起草或编辑 skill
- 在测试提示上运行具备 skill 访问权限的 claude
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 以帮助用户审查
  - 运行定量评估
- 重复此过程直到您和用户满意为止
- 打包最终的 skill 并将其返回给用户。

如果您有待办事项列表（TodoList），请将这些步骤添加进去，以确保不会遗忘。如果您在 Cowork 中，请务必将“创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类审查测试用例”放入待办事项列表，以确保执行该操作。

祝好运！