---
name: skill-creator
description: Create new skills, modify and improve existing skills, and measure skill performance. Use when users want to create a skill from scratch, edit, or optimize an existing skill, run evals to test a skill, benchmark skill performance with variance analysis, or optimize a skill's description for better triggering accuracy.
---

# Skill Creator

用于创建新 skill 和对其进行迭代改进的 skill。

从高层来看，创建 skill 的过程如下：

- 决定你想要这个 skill 做什么，以及大致如何实现
- 撰写 skill 草稿
- 创建一些测试 prompt，并在支持该 skill 的环境下运行 claude-with-access-to-the-skill
- 帮助用户对结果进行定性和定量评估
  - 在运行进行的同时，如果没有定量评估，可以起草一些（如果已有，可以直接使用或根据需要修改）。然后向用户解释这些评估（如果已有，解释现有的即可）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果，同时让他们查看定量指标
- 根据用户对结果的评估反馈（以及定量基准测试中出现的明显缺陷）重写 skill
- 重复上述步骤直到满意
- 扩大测试集，在更大规模上再次尝试

使用这个 skill 时，你的工作是确定用户处于哪个阶段，然后帮助他们推进这些阶段。例如，用户可能说"我想创建一个用于 X 的 skill"。你可以帮助他们明确需求、撰写草稿、编写测试用例、确定评估方式、运行所有 prompt，然后重复迭代。

另一方面，如果用户已经有一个 skill 草稿，你可以直接进入评估/迭代环节。

当然，你应该始终保持灵活性，如果用户说"不需要跑一堆评估，一起凭感觉来"，那也可以。

然后，在 skill 完成之后（顺序同样可以灵活调整），你还可以运行 skill description 优化器——我们有专门的脚本——来优化 skill 的触发机制。

明白了吗？明白了。

## 与用户沟通

Skill Creator 可能会被各种熟悉程度不同的用户使用，他们对编程术语的了解差异很大。如果你还没听说（怎么可能呢，毕竟这只是最近才流行的趋势），现在有一种潮流：Claude 的强大能力激励着水管工打开终端、父母和祖父母去搜索"如何安装 npm"。另一方面，大多数用户可能对计算机相当熟悉。

所以请注意上下文线索，理解如何措辞！在默认情况下，给你一些参考：

- "evaluation"和"benchmark"属于边缘情况，但可以使用
- 对于"JSON"和"assertion"，你需要看到用户明确表现出他们知道这些术语的含义，才可以在不解释的情况下使用

如果有疑问，可以简要解释术语；如果你不确定用户是否能理解，也可以用简短定义来澄清。

---

## 创建 Skill

### 捕获意图

首先理解用户的意图。当前对话中可能已经包含了用户想要捕获的工作流程（例如，他们说"把这个变成一个 skill"）。如果是的话，首先从对话历史中提取答案——使用的工具、步骤顺序、用户的更正、观察到的输入/输出格式。用户可能需要补充缺失的信息，并应在进入下一步之前进行确认。

1. 这个 skill 应该让 Claude 能够做什么？
2. 这个 skill 应该在什么时候触发？（什么用户话语/上下文）
3. 期望的输出格式是什么？
4. 我们是否应该设置测试用例来验证 skill 是否正常工作？具有客观可验证输出的 skill（文件转换、数据提取、代码生成、固定工作流程步骤）适合测试用例。输出主观的 skill（写作风格、艺术）通常不需要。根据 skill 类型建议适当的默认选项，但由用户决定。

### 访谈与研究

主动询问关于边界情况、输入/输出格式、示例文件、成功标准和依赖项的问题。等到这部分明确之后再编写测试 prompt。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似 skill、查阅最佳实践），如果有子代理可用则并行研究，否则内联研究。准备好相关背景知识以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填充以下组件：

- **name**: Skill 标识符
- **description**: 何时触发，做什么。这是主要的触发机制——包含 skill 的功能 AND 具体的使用场景。所有"何时使用"的信息都放在这里，不放在正文里。注意：目前 Claude 有"触发不足"的倾向——在有用的时候没有使用 skill。为了解决这个问题，请把 skill 描述写得稍微"强硬"一些。例如，不要写成"How to build a simple fast dashboard to display internal Anthropic data."，可以写成"How to build a simple fast dashboard to display internal Anthropic data. Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'"
- **compatibility**: 必需的工具、依赖项（可选，一般很少需要）
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

Skill 使用三级加载系统：

1. **元数据**（名称 + 描述）— 始终处于上下文中（约 100 词）
2. **SKILL.md 主体**— 在 skill 触发时处于上下文中（理想情况 <500 行）
3. **捆绑资源**— 按需加载（无限制，脚本可在不加载的情况下执行）

这些字数仅为近似值，如需可以适当增加。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如果接近此限制，请添加额外的层级结构，并提供清晰的指引，告知使用该 skill 的模型接下来应该去哪里跟进。
- 从 SKILL.md 中清晰地引用文件，并提供何时阅读它们的指导
- 对于大型参考文件（>300 行），包含目录

**领域组织**：当一个 skill 支持多个领域/框架时，按变体组织：

```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 只读取相关的参考文件。

#### 不意外原则

这是不言而喻的，技能不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。如果技能有描述，其内容不应在意图上让用户感到意外。不要配合创建具有误导性的技能或旨在促进未授权访问、数据泄露或其他恶意活动的技能。不过，像“扮演 XYZ 角色扮演”之类的内容是可以的。

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

**示例格式** - 加入示例会很有帮助。你可以这样格式化它们（但如果示例中包含 "Input" 和 "Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 写作风格

尝试向模型解释事物为什么重要，而不是使用沉重的、刻板的 MUST 指令。运用心智理论，尽量让技能具有通用性，而不是局限于特定例子。先写一份初稿，然后用全新的视角审视并改进。

### 测试用例

写完技能初稿后，设计 2-3 个真实的测试提示——用户实际会说的那种类型。与用户分享这些测试用例：「我想尝试以下几个测试用例。你觉得这样可以吗，或者想添加更多？」然后运行它们。

将测试用例保存到 `evals/evals.json`。暂不编写断言——只提供提示语。你会在下一步（运行进行中）时起草断言。

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

请参阅 `references/schemas.md` 查看完整 schema（包括你稍后会添加的 `assertions` 字段）。

## 运行和评估测试用例

本节是一个连续的序列——不要中途停止。请勿使用 `/skill-test` 或任何其他测试技能。

将结果放在 skill 目录的同级目录 `<skill-name>-workspace/` 中。在 workspace 内，按迭代组织结果（`iteration-1/`、`iteration-2/` 等），在每个迭代目录下，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要提前创建所有目录——边走边创建即可。

### 步骤 1：在同一轮中启动所有运行（带 skill 和 baseline）

对于每个测试用例，在同一轮中启动两个子 agent——一个带 skill，一个不带。这很重要：不要先启动带 skill 的运行，然后再回来运行 baseline。一次性启动所有任务，使其大致同时完成。

**带 skill 的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**Baseline run**（相同 prompt，但 baseline 取决于上下文）：

- **创建新 skill**：完全不使用 skill。使用相同 prompt，不指定 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。在编辑之前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline 子代理指向快照目录。保存到 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言部分可以暂时留空）。为每个 eval 设置一个描述性名称，基于其测试内容命名——不要简单地叫 "eval-0"。目录名也使用这个名称。如果本次迭代使用了新的或修改过的 eval prompt，需要在每个新 eval 目录下创建这些文件——不要假设它们会从上一次迭代继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步骤2：在运行进行中，起草断言

不要只是等待运行完成——你可以有效地利用这段时间。为每个测试用例起草定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请检查它们并解释它们检查的内容。

好的断言是客观可验证的，并且具有描述性的名称——它们应该在 benchmark 查看器中清晰易读，这样人们在浏览结果时就能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）最好进行定性评估——不要将断言强加于需要人类判断的事物上。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量 benchmark。

### 步骤3：当运行完成时，捕获计时数据

当每个子代理任务完成时，你将收到包含 `total_tokens` 和 `duration_ms` 的通知。立即将这些数据保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这些数据的唯一机会——它通过任务通知传来，不会被持久化存储在其他地方。在通知到达时立即处理，而不是试图批量处理。

### 步骤4：评分、聚合并启动查看器

所有运行完成后：

1. **对每次运行进行评分** — 启动一个 grader 子代理（或内联评分），读取 `agents/grader.md` 并针对输出评估每个断言。将结果保存到每次运行目录下的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 这些字段（不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些精确的字段名。对于可以通过编程检查的断言，编写并运行脚本而不是手动检查——脚本更快、更可靠，并且可以跨迭代重用。

2. **聚合到基准测试** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果需要手动生成 benchmark.json，请参阅 `references/schemas.md` 了解 viewer 期望的确切 schema。

将每个 with_skill 版本放在其对应的 baseline 版本之前。

3. **进行分析师审查** — 阅读基准数据，发现聚合统计可能隐藏的模式。参见 `agents/analyzer.md`（"分析基准结果"部分）了解需要关注的内容，例如始终通过的断言（无论是否有 skill）（无区分度）、高方差的评估（可能不稳定）以及时间/token 权衡。

4. **启动 viewer**，同时提供定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及以后的迭代，还要传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**Cowork / 无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示器，使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作目录中，以便下一次迭代读取。

注意：请使用 `generate_review.py` 来创建查看器，无需编写自定义 HTML。

5. **告知用户**类似："我已在浏览器中打开了结果。有两个标签页——'Outputs' 可以逐个点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后，回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给定的任务
- **Output**：技能生成的文件，尽可能以内联方式渲染
- **Previous Output**（第 2+ 次迭代）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言的通过/失败情况
- **Feedback**：输入框，随用户输入自动保存
- **Previous Feedback**（第 2+ 次迭代）：上次迭代的评论，显示在输入框下方

"Benchmark" 标签页显示统计摘要：每个配置的正确率、耗时和 token 使用量，并附有每个评估的细分和分析师观察。

导航通过 prev/next 按钮或方向键完成。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 第 5 步：读取反馈

当用户完成时，读取 `feedback.json`：

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

空反馈意味着用户认为没有问题。将改进重点放在用户有具体抱怨的测试用例上。完成后关闭 viewer server：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 skill

这是整个流程的核心环节。你已经运行了测试用例，用户也审查了结果，现在你需要根据他们的反馈来改进这个 skill。

### 如何思考改进

1. **从反馈中提炼共性。** 这里的核心目标是创建可以被使用无数次（可能真的如此，甚至更多）且适用于多种不同提示词的 skill。你和用户之所以反复迭代少量的示例，是因为这样可以加快速度。用户对这些示例了如指掌，评估新输出也很快。但如果你和用户共同开发的 skill 只适用于这些示例，那它就毫无用处。与其做一些细碎的过拟合修改，或设置过于严苛的强制要求，不如尝试分支探索，使用不同的比喻或推荐不同的工作模式。尝试的成本相对较低，说不定就能找到绝佳的方案。

2. **保持提示词精简。** 删除那些没有发挥作用的内容。务必阅读转录文本，而不仅仅是最终输出——如果看起来 skill 让模型浪费大量时间做无意义的事情，可以尝试删除导致这种行为的部分，然后观察结果。

3. **解释背后的原因。** 尽力解释你要求模型执行的每项操作的**原因**。当今的 LLM 非常**智能**，它们具备良好的心智理论，在良好的框架下能够超越机械指令，真正发挥作用。即使用户反馈简洁或带有情绪，也要试着真正理解任务、理解用户为什么这样写、他们实际写了什么，然后将这种理解传达给指令。如果你发现自己用大写 ALWAYS 或 NEVER，或使用过于僵化的结构，那就是一个黄色警告信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求做的事情为什么重要。这是一种更人性化、更有力、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录文本，注意子代理是否独立编写了相似的辅助脚本或采用相同的多步骤方法。如果所有 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明 skill 应该捆绑这个脚本。写一次，放到 `scripts/` 目录中，然后告诉 skill 使用它。这样可以节省未来每次调用的重复劳动。

这个任务非常重要（我们正在尝试创造每年数十亿美元的经济价值！）你的思考时间不是瓶颈，慢慢来，真正深入思考。我建议你先写一份修订草案，然后再重新审视并改进。真正尽力代入用户角色，理解他们想要什么、需要什么。

### 迭代循环

改进 skill 后：

1. 将你的改进应用到 skill
2. 将所有测试用例重新运行到新的 `iteration-<N+1>/` 目录，包括基线运行。如果你创建的是新 skill，基线始终是 `without_skill`（无 skill）——这在迭代过程中保持不变。如果你是在改进现有 skill，使用你的判断来决定基线应该是什么：用户最初提交的原始版本，还是上一次迭代。
3. 使用 `--previous-workspace` 指向上一次迭代，启动审查者
4. 等待用户审查并告知你完成
5. 阅读新的反馈，再次改进，重复

持续进行直到：
- 用户表示满意
- 反馈全部为空（一切看起来都很好）
- 你没有取得有意义的进展

---

## 高级用法：盲测对比

在某些情况下，你需要对两个版本的 skill 进行更严格的比较（例如用户问"新版本真的更好吗？"），这时可以使用盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：将两个输出交给一个独立的代理，不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常已经足够。

---

## 描述优化

SKILL.md 前置元数据中的 description 字段是决定 Claude 是否调用 skill 的主要机制。创建或改进 skill 后，建议优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的情况。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I need some clarification to create this eval set:

1. **Which skill is this for?** — I need the skill's name and description to create appropriate test queries. For example:
   - A PDF processing skill?
   - A data analysis skill?
   - An image manipulation skill?
   - Something else?

2. **Where is the template?** — You mentioned `assets/eval_review.html` but I don't have access to this file. Could you paste the HTML template here so I can use it?

Once you provide these details, I can:
- Generate 8-10 "should-trigger" queries (covering various phrasings, edge cases, implicit requests)
- Generate 8-10 "should-not-trigger" queries (near-misses that share keywords but need different tools)
- Populate the HTML template and write it to a temp file for your review
- Start the optimization loop once you approve

What's the skill you want to build evals for?

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（驱动当前会话的模型），使触发测试与用户实际体验相匹配。

运行过程中，定期 tail 输出，向用户更新当前迭代次数和分数情况。

这会自动处理完整的优化循环。它将 eval 数据集拆分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触发率），然后调用 Claude 根据失败情况提出改进建议。它在训练集和测试集上重新评估每个新描述，迭代最多 5 次。完成后，在浏览器中打开 HTML 报告，展示每次迭代的结果，并返回包含 `best_description` 的 JSON——根据测试分数而非训练分数选择，以避免过拟合。

### Skill 触发机制的工作原理

理解触发机制有助于设计更好的 eval 查询。Skill 以其名称和描述出现在 Claude 的 `available_skills` 列表中，Claude 根据描述决定是否查阅某个 skill。需要注意的是，Claude 只会在自身难以轻松处理的任务中查阅 skill——简单的一次性查询（如"读取此 PDF"）可能不会触发 skill，即使描述完全匹配，因为 Claude 可以使用基础工具直接处理。复杂的、多步骤的或专业性的查询在描述匹配时会可靠地触发 skill。

这意味着你的 eval 查询应该足够实质，让 Claude 真正受益于查阅 skill。像"读取文件 X"这样的简单查询是糟糕的测试用例——无论描述质量如何，它们都不会触发 skill。

### 第 4 步：应用结果

从 JSON 输出中获取 `best_description`，并更新 skill 的 SKILL.md frontmatter。向用户展示前后的对比，并报告分数。

---

### 打包和展示（仅在 `present_files` 工具可用时）

检查你是否有权访问 `present_files` 工具。如果没有，请跳过此步骤。如果有，请打包 skill 并向用户展示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

## 打包后，将生成的 `.skill` 文件路径告知用户，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程是相同的（起草 → 测试 → 审核 → 改进 → 重复），但由于 Claude.ai 没有 subagent，某些机制会发生变化。以下是需要调整的内容：

**运行测试用例**：没有 subagent 意味着无法并行执行。对于每个测试用例，阅读技能的 SKILL.md，然后按照其说明自己完成测试提示。一次做一个。这不如独立 subagent 那样严格（因为你既编写了技能又在运行它，所以你有完整的上下文），但这是一个有用的合理性检查——人类审核步骤可以弥补。跳过基准运行——直接使用技能完成任务即可。

**审核结果**：如果无法打开浏览器（例如 Claude.ai 的 VM 没有显示器，或者你在远程服务器上），完全跳过浏览器审核器。直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告诉他们位置以便下载检查。直接请求反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖于基线比较，没有 subagent 就没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能、重新运行测试用例、请求反馈——只是中间没有浏览器审核器。如果有文件系统，你仍然可以将结果组织成迭代目录。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），该工具仅在 Claude Code 中可用。如果你在 Claude.ai 上，请跳过此部分。

**盲测比较**：需要 subagent。跳过。

**打包**：`package_skill.py` 脚本可在任何有 Python 和文件系统的环境中运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称。** 记下技能的目录名称和 `name` frontmatter 字段——原样使用它们。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（不是 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装的技能路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——直接写入可能因权限而失败。

---

## Cowork 专用说明

如果你在 Cowork 中，需要了解的主要内容是：

- 你有 subagent，所以主要工作流程（并行生成测试用例、运行基准、评分等）都能正常工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示而不是并行运行也是可以的。）
- 你没有浏览器或显示器，所以在生成评估查看器时，使用 `--static <output_path>` 来写入独立的 HTML 文件，而不是启动服务器。然后提供链接供用户点击在浏览器中打开 HTML。
- 由于某种原因，Cowork 配置似乎使 Claude 在运行测试后不倾向于生成评估查看器，所以请再重申一遍：无论你在 Cowork 还是 Claude Code 中，在运行测试后，你都应该始终生成评估查看器供人工查看示例，然后再自己修改技能并尝试进行修正，使用 `generate_review.py`（不要自己编写定制的 html 代码。提前致歉，但我还是要大写：**自己在评估输入之前，先生成评估查看器。你想尽快将结果展示给人工！**）
- 反馈的工作方式不同：由于没有运行中的服务器，查看器的"提交所有审核"按钮将下载 `feedback.json` 作为文件。然后你可以从这里读取它（你可能需要先请求访问权限）。
- 打包功能正常——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它通过 subprocess 使用 `claude -p`，而不是浏览器，但请等到你完全完成技能制作并且用户确认其状态良好后再进行此步骤。
- **更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专用 subagent 的说明。当需要生成相关 subagent 时，请阅读它们。

- `agents/grader.md` — 如何根据输出评估断言
- `agents/comparator.md` — 如何对两个输出进行盲测 A/B 比较
- `agents/analyzer.md` — 如何分析为什么一个版本优于另一个

references/ 目录包含其他文档：

- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

这里再次重复核心循环以强调：

- 确定技能的功能
- 起草或编辑技能
- 运行具有技能访问权限的 claude 进行测试提示
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 来帮助用户审核它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终技能并返回给用户。

如果你有 TodoList，请将其添加到其中以确保不会忘记。如果你在 Cowork 中，请特别添加"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审核测试用例"到你的 TodoList 中以确保它被执行。

祝你好运！