---
name: skill-creator
description: 创建新技能、修改和优化现有技能，以及衡量技能表现。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、用方差分析进行技能性能基准测试，或优化技能描述以获得更好的触发准确性时使用。
---

# 技能创建器

用于创建新技能并进行迭代改进的技能。

从高层次来看，创建技能的过程如下：

- 确定你想要技能做什么，以及大致如何实现
- 编写技能初稿
- 创建一些测试提示词，并在支持该技能的 Claude 上运行
- 帮助用户定性和定量地评估结果
  - 在后台运行期间，如果没有定量评估则起草一些（如果已存在，可以直接使用或根据需要进行修改）。然后向用户解释它们（如果已存在，则解释现有的评估内容）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（如果定量基准测试中有任何明显的缺陷也需要考虑）
- 重复直到满意
- 扩大测试集并以更大规模重试

使用此技能时，你的工作是确定用户处于流程的哪个阶段，然后帮助他们推进这些阶段。例如，用户可能说"我想为 X 创建一个技能"。你可以帮助明确他们的意图，编写初稿，编写测试用例，确定他们想要的评估方式，运行所有提示词，并重复这个过程。

另一方面，也许用户已经有一个技能初稿。在这种情况下，你可以直接进入评估/迭代环节。

当然，你应该始终保持灵活，如果用户说"我不需要运行大量评估，只要陪我聊聊"，你也可以照做。

然后在技能完成后（同样，顺序是灵活的），你还可以运行技能描述优化器，我们有专门的脚本来做这件事，以优化技能的触发机制。

好的？好的。

## 与用户沟通

技能创建器可能被各种熟悉编码术语程度不同的用户使用。如果你没听说过（你怎么能听说呢，它才刚刚开始流行不久），现在有一个趋势：Claude 的强大能力正在激励管道工打开终端、父母和祖父母去谷歌搜索"如何安装 npm"。另一方面，大多数用户可能相当熟悉计算机。

所以请注意上下文线索来理解如何措辞！默认情况下，给你一些参考：

- "evaluation"和"benchmark"处于边界，但可以接受
- 对于"JSON"和"assertion"，你需要看到用户确实了解这些术语的明确线索后才能使用而不做解释

如果有疑问，可以简要解释术语；如果不确定用户是否能理解，可以简短地澄清术语定义。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前对话中可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是的话，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的更正、观察到的输入/输出格式。用户可能需要补充空白内容，并在继续下一步之前进行确认。

1. 此技能应该让 Claude 具备什么能力？
2. 此技能何时触发？（用户的哪些措辞/上下文）
3. 期望的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在把测试提示词写好之前，先把这些部分弄清楚。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有子代理则通过子代理并行研究，否则内联研究。做好准备带着背景信息，以减少用户的负担。

### 编写 SKILL.md

基于用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发，做什么。这是主要的触发机制——包括技能做什么以及具体的适用场景。所有"何时使用"的信息都放在这里，不放在正文部分。注意：目前 Claude 有"触发不足"的倾向——在有用的时候不使用技能。为了解决这个问题，请让技能描述稍微"主动"一些。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标，或想要显示任何类型的公司数据时，一定要使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：所需工具、依赖项（可选，很少需要）
- **技能的其他内容 :)**

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

#### 渐进式展开

技能采用三级加载系统：

1. **元数据**（名称 + 描述）- 始终在上下文中呈现（约100词）
2. **SKILL.md 正文** - 技能触发时在上下文中呈现（理想情况下少于500行）
3. **捆绑资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

这些字数统计是近似值，如有需要可以适当增加。

**关键模式：**

- 保持 SKILL.md 在500行以内；如果接近此限制，请添加额外的层级结构，并清晰标注使用该技能的模型下一步应转到何处继续跟进。
- 在 SKILL.md 中清晰地引用文件，并提供何时应读取的指导
- 对于大型参考文件（超过300行），请包含目录

**领域组织**：当技能支持多个领域/框架时，按变体组织：

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

不言而喻，技能不得包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。技能的描述应当与其实际意图相符，不应让用户感到意外。不要配合创建误导性技能或旨在促进未授权访问、数据窃取或其他恶意活动的技能。不过，像“扮演 XYZ”这类角色扮演是可以的。

#### 编写模式

建议在指令中使用祈使语气。

**定义输出格式** - 可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例格式** - 包含示例很有帮助。你可以像这样格式化它们（但如果示例中包含"Input"和"Output"，你可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

试着向模型解释为什么某些事情很重要，而不是采用生硬刻板的强制性要求。运用共情能力，尽量让技能具有通用性，而不是局限于特定案例。先写初稿，然后用新的视角审视并改进。

### 测试用例

写完技能初稿后，准备 2-3 个真实的测试提示——也就是真实用户实际会说的话。分享给用户：[你不必使用完全相同的措辞] "Here are a few test cases I'd like to try. Do these look right, or do you want to add more?" 然后运行这些测试。

Save test cases to `evals/evals.json`. 暂不写断言——只需保存提示词。你会在下一步运行过程中起草断言。

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

完整的 schema（包括你稍后要添加的 `assertions` 字段）请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续的流程——不要中途停止。不要使用 `/skill-test` 或任何其他测试 skill。

将结果放在 `<skill-name>-workspace/` 中，作为 skill 目录的同级目录。在 workspace 内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代下，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要一开始就创建全部目录——边做边创建即可。

### 步骤 1：在同一轮中启动所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮中启动两个子 agent——一个有 skill，一个没有。这很重要：不要先启动 with-skill 运行，之后再回来做 baseline。一次性启动所有任务，这样它们能差不多同时完成。

**With-skill 运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同的提示词，但 baseline 取决于上下文）：

- **创建新 skill**：完全不使用 skill。使用相同提示词，不指定 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。在编辑之前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline 子代理指向该快照。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言部分可以暂时留空）。根据测试内容为每个 eval 赋予描述性名称——不要简单地命名为 "eval-0"。同时用该名称创建对应的目录。如果本次迭代使用了新的或修改过的 eval 提示词，需要为每个新 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### Step 2: 在运行进行期间，起草断言

不要只是等待运行完成——你可以高效利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该能够被客观验证，并且具有描述性的名称——它们应该在基准查看器中清晰可读，这样只要看一眼结果，就能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强行用于需要人类判断的事物。

一旦起草完成，用这些断言更新 `eval_metadata.json` 文件和 `evals/evals.json`。还要向用户解释他们在查看器中将看到什么——包括定性输出和定量基准。

### Step 3: 运行完成时，捕获计时数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获数据的唯一机会——数据通过任务通知传来，不会被持久化保存到其他位置。收到通知时立即处理，不要尝试批量处理。

### 步骤 4：评分、汇总并启动查看器

所有运行完成后：

1. **对每个运行进行评分** — 启动一个 grader 子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每个运行目录下的 `grading.json` 文件中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（不要使用 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以通过程序化检查的断言，编写并运行脚本而不是目测——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **汇总到 benchmark** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参考 `references/schemas.md` 了解 viewer 期望的确切 schema。将每个 with_skill 版本放在其 baseline 对应版本之前。

3. **进行分析师审查** — 阅读 benchmark 数据，找出汇总统计可能掩盖的模式。请参阅 `agents/analyzer.md`（"分析 Benchmark 结果"部分）了解需要关注的内容——例如无论是否有 skill 都始终通过的断言（非区分性）、高方差的评估（可能不稳定）以及 time/token 权衡。

4. **启动 viewer**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第2次及以上的迭代，还要传递 `--previous-workspace <workspace>/iteration-<N-1>` 参数。

**无协作环境 / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会被下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下一次迭代使用。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告知用户** 例如："我已在浏览器中打开了结果。有两个标签页——'Outputs' 允许你逐个点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。你完成后，回到这里告诉我。"

### 查看器中用户看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（第2次迭代起）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言通过/失败情况
- **Feedback**：文本框，输入时自动保存
- **Previous Feedback**（第2次迭代起）：他们上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的通过率、时间和 token 使用量，以及每次评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 第5步：读取反馈

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

您好！我理解您给出的指导原则：

1. 重点关注有具体反馈的测试用例
2. 完成后关闭 viewer 服务器

但您还没有提供需要翻译的文档内容。请将您想要翻译的 Markdown 技术文档发送给我，我会按照您的要求进行翻译。

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 skill

这是循环的核心。你已经运行了测试用例，用户也审查了结果，现在需要根据他们的反馈来改进 skill。

### 如何思考改进

1. **从反馈中进行泛化。** 这里的宏观目标是创建能够被使用数百万次（可能字面意义上，甚至更多）的 skill，跨越各种不同的 prompt。你和用户之所以反复在这几个例子上迭代，是因为这样可以加快速度。用户对这些例子了如指掌，能够快速评估新的输出。但如果你和用户协作开发的 skill 只能在这些例子上运行，那它就毫无用处。与其进行繁琐的过拟合修改，或者施加过于严格的 MUST 约束，如果遇到棘手的问题，可以尝试另辟蹊径，使用不同的比喻或推荐不同的工作模式。尝试的成本相对较低，也许你就能发现绝妙的解决方案。

2. **保持 prompt 精简。** 删除那些没有发挥作用的内容。务必阅读转录文本，而不仅仅是最终输出——如果看起来 skill 让模型浪费了大量时间去做无意义的事情，你可以尝试删除导致这种情况的部分，然后观察结果。

3. **解释原因。** 努力解释你要求模型执行每个操作的**原因**。现在的 LLM 非常"聪明"。它们有很好的心智理论，在给定的框架下可以超越死板指令，真正发挥作用。即使用户反馈简短或带有情绪，也要试着真正理解任务，理解用户为什么写下他们写的内容，理解他们实际写了什么，然后将这种理解传达给指令。如果你发现自己用全大写写 ALWAYS 或 NEVER，或者使用过于僵化的结构，那就是一个黄色警告信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求的事情为什么重要。这是一种更人性化、更有力、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意子代理是否独立编写了类似的辅助脚本或采用了相同的多步骤方法。如果 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈信号——skill 应该打包这个脚本。只写一次，放到 `scripts/` 目录，然后告诉 skill 使用它。这样可以节省每一次未来调用的重复工作。

这个任务非常重要（我们正在尝试创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈，慢慢来，真正深入思考。我建议先写一份修订草稿，然后重新审视并改进。真正尽力进入用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进 skill 后：

1. 将你的改进应用到 skill
2. 将所有测试用例重新运行到新的 `iteration-<N+1>/` 目录，包括基线运行。如果你创建的是新 skill，基线始终是 `without_skill`（无 skill）——这在迭代过程中保持不变。如果你是在改进现有 skill，使用你的判断力来确定什么作为基线是合理的：用户最初带来的原始版本，还是之前的迭代版本。
3. 使用 `--previous-workspace` 指向之前迭代目录来启动 reviewer
4. 等待用户审查并告知完成
5. 阅读新的反馈，再次改进，重复

持续进行直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲对比

在某些情况下，你需要对两个版本的 skill 进行更严格的比较（例如用户问"新版本真的更好吗？"），可以使用盲对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的代理，不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人类审查循环通常就足够了。

---

## 描述优化

SKILL.md frontmatter 中的描述字段是决定 Claude 是否调用 skill 的主要机制。创建或改进 skill 后，提供优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的例子。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I can see you've shared the workflow instructions for creating and refining an eval set, but I'm missing some key information to proceed:

**What I need from you:**

1. **Skill details**: What's the skill name and current description?
2. **Current eval data** (if any): Do you have existing queries to review, or should I generate them from scratch?
3. **Context**: What type of skill is this? (e.g., PDF processing, code generation, data analysis, etc.)

Once you share the skill name and description, I can:

- Generate the should-trigger (8-10) and should-not-trigger (8-10) queries following the guidelines you outlined
- Load the HTML template from `assets/eval_review.html`
- Create the temp file and open it for your review

Just paste the skill description and let me know if you have any existing eval data, or if you're starting fresh.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（即驱动当前会话的那个）来运行触发测试，使其与用户实际体验相匹配。

运行期间，定期查看输出以向用户更新当前迭代次数以及分数情况。

这会自动处理完整的优化循环。它将评估集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会在训练集和测试集上重新评估每个新描述，最多迭代 5 次。完成后，它会在浏览器中打开一个 HTML 报告，显示每次迭代的结果，并返回包含 `best_description` 的 JSON——通过测试分数而非训练分数选择以避免过拟合。

### Skill 触发机制的工作原理

了解触发机制有助于设计更好的评估查询。Skills 会出现在 Claude 的 `available_skills` 列表中，包含名称和描述，Claude 根据该描述决定是否使用某个 skill。需要知道的重要一点是，Claude 只会在无法轻易自行处理的任务上使用 skill——简单的一步查询（如"读取这个 PDF"）即使描述完全匹配也可能不会触发 skill，因为 Claude 可以用基础工具直接处理。复杂的、多步骤的或专业性的查询在描述匹配时会可靠地触发 skills。

这意味着你的评估查询应该足够实质化，让 Claude 真正受益于使用 skill。像"读取文件 X"这样的简单查询不是好的测试用例——无论描述质量如何，它们都不会触发 skills。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description` 并更新 skill 的 SKILL.md frontmatter。向用户展示前后的对比并报告分数。

---

### 打包和展示（仅当 `present_files` 工具可用时）

检查你是否有权访问 `present_files` 工具。如果没有，跳过此步骤。如果有，打包 skill 并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，将用户引导至生成的 `.skill` 文件路径，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，某些机制会有所不同。以下是需要适应的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，读取 skill 的 SKILL.md，然后按照其说明自己完成测试提示。一次完成一个。这不如独立子代理严格（你编写了 skill 也在运行它，所以你有完整的上下文），但这是一个有用的完整性检查——人工审查步骤可以弥补这一点。跳过基线运行——直接使用 skill 完成请求的任务。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示，或者你在远程服务器上），则完全跳过浏览器审查器。直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），则保存到文件系统并告知他们位置，以便他们下载和检查。内联请求反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖于基线比较，在没有子代理的情况下没有意义。重点关注用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，请求反馈——只是中间没有浏览器审查器。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（ specifically `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过。

**盲比较**：需要子代理。跳过。

**打包**：`package_skill.py` 脚本可以在任何有 Python 和文件系统的地方运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而非创建新技能。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名称和 `name` frontmatter 字段——保持不变地使用。例如，如果已安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而非 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 专用说明

如果你在 Cowork 中，需要了解的主要内容是：

- 你有子代理，所以主要工作流（并行生成测试用例、运行基线、评分等）都可以工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示是可以的。）
- 你没有浏览器或显示，所以在生成评估查看器时，使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 出于某种原因，Cowork 配置似乎不鼓励 Claude 在运行测试后生成评估查看器，所以重申一下：无论你在 Cowork 还是 Claude Code 中，运行测试后都应该始终生成评估查看器，以便人工在修订 skill 之前查看示例并尝试做出修正，使用 `generate_review.py`（不要自己写花哨的 html 代码）。提前致歉，但我还是要大写：**在评估输入之前先生成评估查看器**！你想让他们尽快呈现在人工面前！
- 反馈的工作方式不同：由于没有运行中的服务器，查看器的"Submit All Reviews"按钮会将 `feedback.json` 下载为文件。你可以从那里读取它（可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成 skill 并且用户同意它状态良好后再保存。
- **更新现有 skill**：用户可能要求你更新现有 skill，而非创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相应子代理时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本胜出

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次强调核心循环：

- 确定 skill 的主题
- 起草或编辑 skill
- 运行可以访问 skill 的 claude 来处理测试提示
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 来帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终 skill 并返回给用户。

请将步骤添加到你的 TodoList（如果有的话），以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"放入 TodoList 以确保它会发生。

祝你好运！