---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及衡量技能表现。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、对技能性能进行方差分析基准测试，或优化技能的描述以获得更好的触发准确性时使用。
---

# Skill Creator

一个用于创建新技能并迭代改进它们的技能。

从高层次来看，创建技能的过程如下：

- 决定你想要技能做什么，以及大致如何实现
- 编写技能的初稿
- 创建一些测试提示词，并使用 claude-with-access-to-the-skill 运行它们
- 帮助用户定性和定量地评估结果
  - 在运行在后台进行时，如果还没有定量评估，可以起草一些（如果已有一些，你可以直接使用或进行修改，如果你认为需要改变什么）。然后向用户解释它们（或者如果它们已经存在，解释已存在的那些）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（以及如果定量基准测试中出现任何明显的缺陷）
- 重复直到你满意
- 扩展测试集并尝试更大规模的测试

使用此技能时，你的工作是弄清楚用户处于这个流程的哪个阶段，然后帮助他们推进这些阶段。例如，也许他们说"我想为 X 创建一个技能"。你可以帮助他们明确需求、撰写初稿、编写测试用例、确定评估方式、运行所有提示词，并重复这个过程。

另一方面，也许他们已经有一个技能初稿。这种情况下，你可以直接进入评估/迭代部分。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估，就随便聊聊吧"，你也可以那样做。

然后在技能完成后（但同样，顺序是灵活的），你也可以运行技能描述优化器，我们有专门为此的脚本，来优化技能的触发。

好了吗？好。

## 与用户沟通

Skill Creator 可能会被各种熟悉代码术语程度不同的用户使用。如果你没听说过（你怎么可能听说，它才刚刚开始流行），现在有一种趋势，Claude 的强大激发了水管工打开终端、父母和祖父母去搜索"如何安装 npm"。另一方面，大多数用户可能对计算机相当熟悉。

所以请注意上下文线索，了解如何措辞！在默认情况下，给你一些参考：

- "evaluation"和"benchmark"属于边缘情况，但可以使用
- 对于"JSON"和"assertion"，你需要看到用户明显知道这些东西是什么，才可以不加解释地使用它们

如果有疑问，可以简要解释术语，如果不确定用户是否能理解，可以添加简短定义。

---

## 创建技能

### 捕获意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是，先从对话历史中提取答案——使用的工具、步骤顺序、用户的纠正、观察到的输入/输出格式。用户可能需要填补空白，并在继续下一步之前确认。

1. 这个技能应该让 Claude 具备什么能力？
2. 什么时候应该触发这个技能？（用户的哪些措辞/上下文）
3. 预期的输出格式是什么？
4. 我们应该设置测试用例来验证技能是否有效吗？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项。在把这些部分弄清楚之前，不要急于编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果有的话通过子代理并行研究，否则内联进行。做好准备，带着上下文来减少用户的负担。

### 编写 SKILL.md

基于用户访谈，填写以下组件：

- **name**: 技能标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包括技能做什么以及具体的使用场景。所有"何时使用"的信息都在这里，不在正文里。注意：目前 Claude 倾向于"触发不足"——在有用的时候不使用技能。为了解决这个问题，请把技能描述写得稍微"强硬"一点。例如，不要写成"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写成"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。当用户提到仪表板、数据可视化、内部指标，或想要显示任何类型的公司数据时，一定要使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**: 必需的工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

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

Skills 采用三级加载系统：

1. **元数据**（名称 + 描述）- 始终加载以保持上下文（~100 词）
2. **SKILL.md 正文** - 技能触发时加载以保持上下文（理想情况下 <500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

以上字数仅为近似值，如有需要可以适当增加。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如果接近此限制，请添加额外的层级结构，并提供清晰的指引，告诉使用该技能 model 接下来应该去哪里跟进。
- 从 SKILL.md 中明确引用文件，并提供何时读取的指导
- 对于大型参考文件（>300 行），请包含目录

**领域组织**：当一个 skill 支持多个领域/框架时，按变体进行组织：

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

这不用说，但技能中不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果技能被描述了，其内容不应在意图上使用户感到意外。不要配合创建具有误导性的技能或旨在促进未授权访问、数据泄露或其他恶意活动的技能。不过，像“扮演 XYZ 角色扮演”这样的内容是可以的。

#### 编写模式

在指令中优先使用祈使语气。

**定义输出格式** - 可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例会很有用。你可以这样格式化它们（但如果示例中包含 "Input" 和 "Output"，你可能需要稍作调整）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事情为什么重要，而不是用老套生硬的 MUST 命令。运用心智理论（theory of mind），尽量让技能具有通用性，而不是局限于特定示例。先写一份初稿，然后用新的视角审视并改进。

### 测试用例

写完技能初稿后，设计 2-3 个真实的测试提示——即真实用户可能会说的话类型。将它们分享给用户：[不必使用完全相同的措辞]"这是我想要尝试的几个测试用例。看起来是否合适，或者你想要添加更多？"然后运行它们。

将测试用例保存到 `evals/evals.json`。暂不编写断言——只有提示。在下一步中，当运行进行时再起草断言。

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

参见 `references/schemas.md` 获取完整 schema（包括后续需要添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的完整流程——不要中途停止。不要使用 `/skill-test` 或任何其他测试技能。

将结果放在 `<skill-name>-workspace/` 中，作为 skill 目录的同级目录。在 workspace 内，按迭代组织结果（`iteration-1/`、`iteration-2/` 等），在每个迭代下，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要预先创建所有目录——边做边创建。

### 步骤 1：在同一轮次中启动所有运行（带技能版本 AND 基线版本）

对于每个测试用例，在同一轮次中启动两个子代理——一个带技能，一个不带技能。这很重要：不要先启动带技能的运行，然后再回来做基线版本。同时启动所有运行，这样它们大致同时完成。

**带技能的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同 prompt，但 baseline 取决于上下文）：

- **创建新技能**：完全不使用技能。使用相同 prompt，不指定技能路径，保存到 `without_skill/outputs/`。
- **改进现有技能**：使用旧版本。编辑前，先对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline 子代理指向快照目录。保存到 `old_skill/outputs/`。

为每个测试用例编写一个 `eval_metadata.json`（断言部分可暂时留空）。为每个评估使用描述性名称——基于其测试内容，而非仅仅 "eval-0"。同时将此名称用于目录命名。如果本次迭代使用了新的或修改过的评估 prompt，请为每个新评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### Step 2: 在运行进行时起草断言

不要只是等待运行完成——你可以有效地利用这段时间。为每个测试用例起草定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言是可客观验证的，且具有描述性的名称——它们应该在 benchmark viewer 中清晰可读，这样人们在查看结果时能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的内容。

一旦起草完断言，请更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在 viewer 中将看到的内容——包括定性输出和定量基准。

### Step 3: 当运行完成时，捕获计时数据

当每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将此数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它通过任务通知获取，不会持久化到其他位置。收到每个通知时立即处理，而不是尝试批量处理。

### 步骤 4：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分** —— 生成一个评分子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录下的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而非 `name`/`met`/`details` 或其他变体）—— 查看器依赖这些确切的字段名。对于可以编程检查的断言，编写并运行脚本而非肉眼观察 —— 脚本更快、更可靠，且可以在迭代中复用。

2. **聚合成基准** —— 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这将生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解 viewer 期望的确切 schema。

将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **进行分析师审查** — 阅读 benchmark 数据，发现汇总统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"分析 Benchmark 结果"部分）了解需要关注的内容——例如无论 skill 如何都总是通过的断言（非区分性）、高方差的 evals（可能是 flaky），以及 time/token 权衡。

4. **启动 viewer**，同时加载定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代2+，还需要传递 `--previous-workspace <workspace>/iteration-<N-1>`。

**Cowork/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，请使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下一次迭代读取。

注意：请使用 `generate_review.py` 来生成查看器；无需编写自定义 HTML。

5. **告知用户** 类似这样："我已在浏览器中打开了结果。有两个标签页——'Outputs' 允许你逐个点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后，回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给出的任务
- **Output**：skill 产出的文件，尽可能以内联方式渲染
- **Previous Output**（迭代2+）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了 grading）：折叠区域，显示 assertion 的通过/失败情况
- **Feedback**：自动保存的文本框
- **Previous Feedback**（迭代2+）：他们上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的通过率、耗时和 token 使用量，包含每个 eval 的细分和分析师观察。

导航通过 prev/next 按钮或方向键。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤5：读取反馈

当用户告诉你完成时，读取 `feedback.json`：

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

空反馈表示用户认为没问题。将改进重点放在用户有具体投诉的测试用例上。

完成后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 skill

这是循环的核心。你已经运行了测试用例，用户也审查了结果，现在你需要根据他们的反馈来改进 skill。

### 如何思考改进

1. **从反馈中进行泛化。** 这里的大局是，我们正在尝试创建可以被使用无数次（也许真的是无数次，甚至更多）的 skills，应用于各种不同的 prompts。这里你和用户只是在少数几个示例上反复迭代，因为这有助于加快速度。用户对这些示例了如指掌，可以快速评估新的输出。但如果你们协同开发的 skill 只能用于这些示例，那它就毫无用处。与其做出过于精细的过拟合修改，或施加过度限制性的 MUST 规则，如果有顽固的问题，你可以尝试扩展思路，使用不同的隐喻，或推荐不同的工作模式。尝试的成本相对较低，也许你会发现很棒的解决方案。

2. **保持 prompt 精简。** 删除没有发挥作用的内容。不仅要阅读最终输出，还要阅读转录文本——如果看起来 skill 让模型浪费大量时间做无产出的事情，你可以尝试删除导致这种行为的 skill 部分，看看会发生什么。

3. **解释原因。** 努力解释你要求模型执行的每个操作的**原因**。当今的 LLM 非常智能，它们有很好的心智理论，给定一个好的框架后可以超越机械指令，真正实现目标。即使用户的反馈简短或带有挫败感，也要努力真正理解任务，理解用户写这些内容的原因，理解他们实际写了什么，然后将这种理解传达给指令。如果你发现自己用大写写 ALWAYS 或 NEVER，或者使用过于僵化的结构，这是一个黄色警示信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求做的事情为什么重要。这是一种更人性化、更有力、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意 subagent 是否都独立编写了类似的辅助脚本或采用相同的多步骤方法。如果 3 个测试用例都导致 subagent 编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明 skill 应该捆绑该脚本。只写一次，放到 `scripts/` 中，并告诉 skill 使用它。这样可以节省未来每次调用的重复劳动。

这个任务非常重要（我们正试图创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；花时间仔细思考。我建议先写一份修订草案，然后再重新审视并改进。真正尽力进入用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进 skill 后：

1. 将改进应用到 skill
2. 将所有测试用例重新运行到新的 `iteration-<N+1>/` 目录中，包括 baseline 运行。如果你正在创建新 skill，baseline 始终是 `without_skill`（无 skill）——这在迭代之间保持不变。如果你正在改进现有 skill，使用你的判断力决定什么作为 baseline 有意义：用户最初使用的原始版本，还是上一个迭代。
3. 使用 `--previous-workspace` 指向之前的迭代来启动 reviewer
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈都是空的（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级：盲测对比

在需要更严格地比较两个版本的 skill 时（例如用户问"新版本真的更好吗？"），有一个盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的 agent，但不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要使用 subagents，大多数用户不需要。人工审查循环通常已经足够。

---

## 描述优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用 skill 的主要机制。创建或改进 skill 后，提供优化 description 以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合 should-trigger 和 should-not-trigger。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I understand the process you're describing for creating an evaluation set, but I need more information to proceed:

**What I need from you:**

1. **Skill name** — What is the skill called?
2. **Skill description** — What does this skill do? (This is the current version that may need optimization)
3. **Skill purpose/context** — What is this skill meant to help users with?

**Optional but helpful:**

- Any edge cases you already know about
- Common queries users might ask
- What other skills might compete with this one

Once you provide this, I'll:

1. Generate 8-10 realistic **should-trigger** queries with varied phrasings (formal/casual, explicit/implicit skill mention)
2. Generate 8-10 tricky **should-not-trigger** queries (near-misses that share keywords but need different handling)
3. Read the `assets/eval_review.html` template and generate a review HTML file for you
4. Help you run the optimization loop

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

运行过程中，定期查看输出，向用户更新当前迭代次数和分数情况。

这会自动处理完整的优化循环。它将评估集分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它会重新评估每个新描述在训练集和测试集上的表现，最多迭代 5 次。完成后，会在浏览器中打开 HTML 报告，显示每次迭代的结果，并返回 JSON 格式的 `best_description`——按测试分数而非训练分数选择，以避免过拟合。

### 技能触发机制的工作原理

理解触发机制有助于设计更好的评估查询。技能会以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 根据描述来决定是否查阅某个技能。重要的是，Claude 只会在无法独立轻松处理的任务上查阅技能——简单的一步式查询（如"读取此 PDF"）可能不会触发技能，即使描述完全匹配，因为 Claude 可以直接使用基础工具处理。复杂、多步骤或专业化的查询在描述匹配时可以可靠地触发技能。

这意味着你的评估查询应该足够实质，让 Claude 真正受益于查阅技能。像"读取文件 X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发技能。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md frontmatter。向用户展示修改前后的对比，并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查你是否拥有 `present_files` 工具的访问权限。如果没有，跳过此步骤。如果有，打包技能并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

# Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（草稿 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，一些机制会有所不同。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，阅读技能的 SKILL.md，然后按照其说明自行完成任务。一次执行一个。这不如独立子代理那样严格（你编写了技能同时也在运行它，所以你有完整的上下文），但这是一个有用的 sanity check——人类审查步骤可以弥补这一点。跳过基线运行——直接使用技能完成任务。

**审查结果**：如果无法打开浏览器（例如 Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），完全跳过浏览器审查器。直接在对话中展示结果。对于每个测试用例，显示提示词和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告知他们位置，以便他们可以下载和检查。内联请求反馈："效果如何？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，请求反馈——只是中间没有浏览器审查器。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果你在 Claude.ai 上，跳过此部分。

**盲测比较**：需要子代理。跳过此部分。

**打包**：`package_skill.py` 脚本在有 Python 和文件系统的任何地方都能工作。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称。** 记下技能的目录名称和 `name` frontmatter 字段——保持不变。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑前先复制到可写位置。** 已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果是手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——直接写入可能因权限问题而失败。

---

## Cowork 专用说明

如果你在 Cowork 中，需要了解的主要内容如下：

- 你有子代理，所以主要工作流程（在并行中生成测试用例、运行基线、评分等）都可以正常工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示词也是可以的。）
- 你没有浏览器或显示器，所以在生成 eval viewer 时，使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在自己的浏览器中打开 HTML。
- 由于某些原因，Cowork 配置似乎让 Claude 不太倾向于在运行测试后生成 eval viewer，所以这里再次强调：无论你在 Cowork 还是 Claude Code 中，在运行测试后都应该始终生成 eval viewer，让人类查看示例，然后再自己修订技能并尝试修正，使用 `generate_review.py`（而不是自己编写花哨的 html 代码）。提前致歉，但我还是要大写：**在评估输入之前先生成 EVAL VIEWER***。你想尽快让他们出现在人类面前！
- 反馈工作方式不同：由于没有运行中的服务器，viewer 的"Submit All Reviews"按钮将下载 `feedback.json` 作为文件。然后你可以从那里读取它（你可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 subprocess 调用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户同意其状态良好后再进行。
- **更新现有技能**：用户可能是在请求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何在两个输出之间进行盲测 A/B 比较
- `agents/analyzer.md` — 如何分析为什么某个版本胜出

references/ 目录有额外的文档：

- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再次强调核心循环：

- 理解技能的功能
- 起草或编辑技能
- 在测试提示词上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终技能并返回给用户

如果，你有 TodoList 的话，请将这些步骤添加到你的 TodoList 中，以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人类审查测试用例"放入你的 TodoList 中，以确保它能够发生。

祝你好运！