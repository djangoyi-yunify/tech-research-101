---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及测量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、通过方差分析进行技能性能基准测试，或优化技能描述以提高触发准确性时使用。
---

# 技能创建器

一个用于创建新技能并对其进行迭代改进的技能。

从高层次来看，创建技能的过程如下：

- 决定你希望技能做什么以及大致如何实现
- 起草技能的初稿
- 创建一些测试提示词，并在 claude-with-access-to-the-skill 上运行它们
- 帮助用户定性和定量地评估结果
  - 在后台运行的同时，如果没有现成的定量评估，就起草一些（如果有现成的，你可以直接使用或修改，如果你觉得需要改变某些内容）。然后向用户解释它们（或者如果已经存在，解释那些已存在的）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果让他们查看，同时也让他们查看定量指标
- 根据用户对结果的评估反馈（以及如果定量基准测试中出现任何明显缺陷）重写技能
- 重复直到你满意为止
- 扩大测试集并在更大规模上再次尝试

你使用这个技能时的任务是找出用户在这个过程中的位置，然后介入帮助他们完成这些阶段。例如，可能他们说"我想为 X 创建一个技能"。你可以帮助缩小他们的含义范围，编写初稿，编写测试用例，找出他们想要如何评估，运行所有提示词，并重复。

另一方面，也许他们已经有一个技能草案。在这种情况下，你可以直接进入评估/迭代循环部分。

当然，你应该保持灵活性，如果用户说"我不需要运行一堆评估，只是陪我聊聊"，你也可以这样做。

然后在技能完成后（但同样，顺序是灵活的），你还可以运行技能描述改进器，我们有一个单独的脚本，用于优化技能的触发。

明白了吗？明白了。

## 与用户沟通

技能创建器可能被各种熟悉程度不同的用户使用。如果你没有听说（你怎么能听说呢，这是最近才开始的现象），现在有一种趋势：Claude 的强大能力激励着水管工打开终端，父母和祖父母去谷歌"如何安装 npm"。另一方面，大多数用户可能相当精通计算机。

所以请注意上下文线索来理解如何措辞！默认情况下，给你一些概念：

- "evaluation"和"benchmark"是边界词，但可以使用
- 对于"JSON"和"assertion"，你需要看到用户确实知道这些东西的严重线索，然后再使用它们而无需解释

如果你有疑问，可以简要解释术语，如果你不确定用户是否能理解，可以随时用简短的定义来澄清术语。

---

## 创建技能

### 捕捉意图

首先理解用户的意图。当前对话可能已经包含用户想要捕获的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，首先从对话历史中提取答案——使用的工具、步骤顺序、用户所做的修正、观察到的输入/输出格式。用户可能需要填补空白，并应该在进入下一步之前确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么用户短语/上下文）
3. 预期的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能是否有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。具有主观输出的技能（写作风格、艺术）通常不需要它们。根据技能类型建议适当的默认值，但让用户决定。

### 访谈和研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖关系。在你完成这部分之前，先不要编写测试提示词。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查找最佳实践），如果可用，通过子代理并行研究，否则内联研究。准备好上下文以减少用户的负担。

### 编写 SKILL.md

根据用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发，触发后做什么。这是主要的触发机制——包括技能做什么以及具体的使用场景。所有"何时使用"的信息都放在这里，而不是放在正文部分。注意：目前 Claude 有一种"触发不足"的倾向——在技能有用的时候不使用它。为了解决这个问题，请把技能描述写得稍微"强势"一点。例如，不要写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据"，你可以写"如何构建一个简单的快速仪表板来显示内部 Anthropic 数据。每当用户提到仪表板、数据可视化、内部指标或想要显示任何类型的公司数据时，务必使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：必需工具、依赖项（可选，很少需要）
- **技能的其余部分 :)**

### 技能写作指南

#### 技能的解剖结构

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

1. **元数据**（名称 + 描述）- 始终在上下文中呈现（约 100 字）
2. **SKILL.md 正文** - 技能触发时加载（理想情况下 < 500 行）
3. **捆绑资源** - 按需加载（无限制，脚本可在不加载的情况下执行）

以上字数为近似值，如需要可以超出。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如接近此限制，请添加额外的层级结构，并提供清晰的指引，说明使用该技能的模型接下来应如何跟进。
- 在 SKILL.md 中清晰引用文件，并提供何时阅读的指导
- 对于大型引用文件（> 300 行），请包含目录

**领域组织**：当技能支持多个领域/框架时，按变体组织：

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

不言而喻，技能不能包含恶意软件、漏洞利用代码或任何可能危害系统安全的内容。如果对技能进行了描述，其内容不应在意图上让用户感到意外。不要配合创建误导性技能或设计用于未经授权访问、数据泄露或其他恶意活动的技能。不过，像"扮演 XYZ"这类角色扮演是可以的。

#### 编写模式

在指令中优先使用祈使形式。

**定义输出格式** - 你可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例很有用。你可以这样格式化它们（不过如果示例中有 "Input" 和 "Output"，你可能需要稍微调整一下格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 编写风格

试着向模型解释为什么某些内容很重要，而不是使用陈旧生硬的「必须」指令。运用心智理论，尽量让技能具有通用性，而不是局限于具体例子。先写初稿，然后用新的眼光审视并改进它。

### 测试用例

写完技能初稿后，设计 2-3 个真实的测试提示——即用户实际会说的话。与用户分享这些提示：[不必使用完全相同的措辞]「这是我想测试的几个用例。这样可以吗，或者你想添加更多？」然后运行它们。

将测试用例保存到 `evals/evals.json`。暂不编写断言——只保存提示词。运行测试的同时，你将在下一步编写断言。

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

完整的 schema（包括你稍后要添加的 `assertions` 字段）请参阅 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续的完整序列——请勿中途停止。不要使用 `/skill-test` 或任何其他测试 skill。

将结果放在 skill 目录同级的 `<skill-name>-workspace/` 中。在工作区内，按迭代（`iteration-1/`、`iteration-2/` 等）组织结果，在每个迭代内，每个测试用例都有一个目录（`eval-0/`、`eval-1/` 等）。不要提前创建所有目录——随着进展逐步创建。

### 步骤 1：在同一轮次中启动所有运行（包括使用 skill 的和基线的）

对于每个测试用例，在同一轮次中启动两个子代理——一个带有 skill，一个不带。这很重要：不要先启动使用 skill 的运行，然后再回来启动基线的。一次全部启动，这样它们几乎同时完成。

**使用 skill 的运行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基线运行**（相同提示词，但基线取决于上下文）：

- **创建新技能**：完全没有技能。相同提示词，无技能路径，保存至 `without_skill/outputs/`。
- **改进现有技能**：旧版本。编辑前，对技能进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后将基线子代理指向该快照。保存至 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言暂时可留空）。为每个评估使用描述性名称——不要只写 "eval-0"。目录名称也使用此名称。如果本次迭代使用了新的或修改后的评估提示词，为每个新的评估目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第二步：在运行进行时起草断言

不要只是等待运行完成——你可以高效利用这段时间。为每个测试用例起草定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们在基准测试查看器中应该清晰易读，这样人们在浏览结果时能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要强行对需要人工判断的内容使用断言。

起草完断言后，更新 `eval_metadata.json` 文件和 `evals/evals.json`。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量基准测试。

### 第三步：运行完成时，捕获计时数据

每个子代理任务完成时，你都会收到包含 `total_tokens` 和 `duration_ms` 的通知。将此数据立即保存到运行目录中的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获此数据的唯一机会——它通过任务通知传递，不会持久化到其他位置。在通知到达时立即处理，而不是尝试批量处理。

### 第四步：评分、聚合并启动查看器

所有运行完成后：

1. **为每次运行评分** — 启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每个运行目录的 `grading.json`。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以编程检查的断言，编写并运行脚本而不是凭肉眼判断——脚本更快、更可靠，并且可以在迭代中复用。

2. **聚合到基准** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，其中包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器期望的确切 schema。
将每个 with_skill 版本放在其 baseline 版本之前。

3. **进行分析师审查** — 阅读基准数据，揭示总体统计可能隐藏的模式。请参阅 `agents/analyzer.md`（"Analyzing Benchmark Results"部分）了解需要关注的内容——例如无论技能如何都总是通过的断言（非区分性）、高方差的评估（可能不稳定）以及时间/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于迭代 2+，还需要传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作/无头环境：** 如果 `webbrowser.open()` 不可用或环境没有显示设备，使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作区目录，以便下一次迭代拾取。

注意：请使用 generate_review.py 来创建查看器；无需编写自定义 HTML。

5. **告知用户** 类似这样的话："我已在浏览器中打开了结果。有两个标签页——'Outputs' 允许你浏览每个测试用例并留下反馈，'Benchmark' 显示定量对比。完成后回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能内联渲染
- **Previous Output**（迭代 2+）：折叠区域，显示上一次迭代的输出
- **Formal Grades**（如果运行了分级）：折叠区域，显示断言的通过/失败情况
- **Feedback**：输入时自动保存的文本框
- **Previous Feedback**（迭代 2+）：他们上次的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的正确率、时间和 token 使用量，以及每个评估的细分和分析师观察。

通过 prev/next 按钮或方向键导航。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

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

空的反馈意味着用户认为没问题。将你的改进重点放在用户有具体投诉的测试用例上。

完成后关闭查看器服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 提升技能

这是循环的核心部分。你已经运行了测试用例，用户也已经审查了结果，现在你需要根据他们的反馈来改进技能。

### 如何思考改进

1. **从反馈中总结规律。** 这里的大局观是：我们正在尝试创建可以使用数百万次（也许真的可以，甚至更多，谁知道呢）的技能，跨越许多不同的提示词。你和用户之所以在这里反复迭代少数几个例子，是因为这样可以加快速度。用户对这些例子了如指掌，能够快速评估新的输出。但如果你和用户共同开发的技能只适用于这些例子，那它就毫无用处。与其进行繁琐的过拟合修改，或者使用压迫性的严格 MUST 指令，不如尝试分支出去，使用不同的隐喻，或者推荐不同的工作模式。尝试的成本相对较低，也许你会发现很棒的方法。

2. **保持提示词精简。** 移除没有发挥作用的东西。务必阅读转录稿，而不仅仅是最终输出——如果看起来技能正在让模型浪费大量时间做无产出的事情，你可以尝试删除导致这种行为的技能部分，然后看看会发生什么。

3. **解释原因。** 努力解释你要求模型做每件事的**原因**。当今的 LLM 很聪明。它们有很好的心智理论，如果给它们一个好的框架，就能超越死板的指令，真正让事情发生。即使用户的反馈很简短或很沮丧，也要试着真正理解任务，理解用户为什么写他们写的内容，以及他们实际写了什么，然后将这种理解传达给指令。如果你发现自己写了很多全大写的 ALWAYS 或 NEVER，或者使用超级僵化的结构，那就是一个黄色警告信号——如果可能的话，重新组织并解释推理，这样模型就能理解你要求的事情为什么重要。这是一个更人性化、更有力量、更有效的方法。

4. **寻找测试用例之间的重复工作。** 阅读测试运行的转录稿，注意子代理是否独立编写了类似的辅助脚本，或者对某件事情采取了相同的多步骤方法。如果所有 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明技能应该打包那个脚本。写一次，放到 `scripts/` 里，然后告诉技能使用它。这能让以后每次调用都无需重复造轮子。

这个任务相当重要（我们正在尝试创造数十亿美元的年度经济价值！），你的思考时间不是瓶颈；慢慢来，仔细思考。我建议你先写一份修改草稿，然后以新的视角审视它并做出改进。真的要想用户之所想，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能上
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录，包括基线运行。如果你正在创建一个新技能，基线始终是 `without_skill`（无技能）——这在所有迭代中保持不变。如果你在改进现有技能，使用你的判断力来判断什么作为基线有意义：用户最初带来的原始版本，还是前一个迭代。
3. 使用 `--previous-workspace` 指向前一个迭代来启动审查器
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续直到：
- 用户表示满意
- 反馈都是空的（一切都看起来很好）
- 你没有取得有意义的进展

---

## 高级：盲评对比

在某些情况下，你需要对技能的两个版本进行更严格的比较（例如，用户问"新版本真的更好吗？"），有一个盲评对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：把两个输出交给一个独立的代理，不告诉它哪个是哪个，让它判断质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常就足够了。

---

## 描述优化

SKILL.md _frontmatter_ 中的描述字段是决定 Claude 是否调用技能的主要机制。创建或改进技能后，建议优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——混合应该触发和不应该触发的情况。保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I see you've shared instructions for creating and reviewing evaluation sets for a skill, but I need some clarification to help you:

1. **Which skill is this for?** I need the skill name and description to create appropriate eval queries.

2. **What step are you on?**
   - **Step 1**: Create eval queries (should-trigger and should-not-trigger)
   - **Step 2**: Review with the HTML template
   - **Step 3**: Run the optimization loop

3. **Do you have existing eval data?** If so, please share it or let me know the file path.

Could you provide:
- The skill name (e.g., "PDF Extraction", "Excel Formula Editor")
- A brief description of what the skill does
- Which step you'd like me to help with

Once I have this context, I can help create realistic eval queries or assist with the review/optimization process.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用系统提示中的模型 ID（运行当前会话的模型），以便触达测试与用户实际体验一致。

在运行过程中，定期查看输出，向用户更新当前迭代次数和分数情况。

这会自动处理完整的优化循环。它将评估集分为 60% 训练集和 40% 保留测试集，评估当前描述（每个查询运行 3 次以获得可靠的触达率），然后调用 Claude 根据失败情况提出改进建议。它重新评估每个新描述在训练集和测试集上的表现，最多迭代 5 次。完成后，它会在浏览器中打开一份 HTML 报告，显示每次迭代的结果，并返回 JSON，其中包含 `best_description` —— 按测试分数而非训练分数选择，以避免过拟合。

### 技能触达机制的工作原理

了解触达机制有助于设计更好的评估查询。技能会以名称和描述的形式出现在 Claude 的 `available_skills` 列表中，Claude 会根据该描述决定是否查阅技能。需要知道的重要一点是，Claude 只会在无法自行轻松处理的任务上查阅技能——简单、单步的查询（如"读取这个 PDF"）可能不会触达技能，即使描述完全匹配，因为 Claude 可以直接用基础工具处理它们。复杂、多步或专业化的查询在描述匹配时能可靠地触达技能。

这意味着您的评估查询应该有足够的实质性内容，让 Claude 真正能从查阅技能中受益。简单的查询如"读取文件 X"是糟糕的测试用例——无论描述质量如何，它们都不会触达技能。

### 步骤 4：应用结果

从 JSON 输出中获取 `best_description`，并更新技能的 SKILL.md 前置元数据。向用户展示修改前后的对比，并报告分数。

---

### 打包和演示（仅在 `present_files` 工具可用时）

检查您是否有权访问 `present_files` 工具。如果没有，请跳过此步骤。如果有，请打包技能并向用户演示 .skill 文件：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后，将用户指向生成的 `.skill` 文件路径，以便他们可以安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，一些机制会有所不同。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，先阅读 skill 的 SKILL.md，然后按照其说明自行完成测试提示。按顺序逐一执行。这不如独立子代理那么严格（因为你既编写了 skill 又在运行它，所以你有完整的上下文），但这是一个有用的健全性检查——而人工审查步骤可以弥补这一点。跳过基线运行——直接使用 skill 完成请求的任务。

**审查结果**：如果你无法打开浏览器（例如，Claude.ai 的虚拟机没有显示，或者你在远程服务器上），则完全跳过浏览器审查器。相反，直接在对话中呈现结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），将其保存到文件系统并告知他们所在位置，以便他们可以下载和检查。inline 征求反馈：“看起来怎么样？有什么需要修改的吗？”

**基准测试**：跳过定量基准测试——它依赖基线比较，在没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进 skill，重新运行测试用例，征求反馈——只是中间没有浏览器审查器。如果有文件系统，你仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，请跳过此部分。

**盲态对比**：需要子代理。跳过此部分。

**打包**：`package_skill.py` 脚本可以在任何有 Python 和文件系统的地方运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新。在这种情况下：
- **保留原始名称。** 记下 skill 的目录名称和 `name` 前端字段——保持不变。例如，如果安装的 skill 是 `research-helper`，则输出 `research-helper.skill`（而不是 `research-helper-v2`）。
- **在编辑之前复制到可写位置。** 安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在副本中进行编辑，然后从副本打包。
- **如果手动打包，先暂存到 `/tmp/`**，然后复制到输出目录——由于权限限制，直接写入可能会失败。

---

## Cowork 专用说明

如果在 Cowork，需要了解的主要事项是：

- 你有子代理，所以主工作流程（并行生成测试用例、运行基线、评分等）都能正常运作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示也是可以的。）
- 你没有浏览器或显示器，所以在生成评估查看器时，使用 `--static <output_path>` 编写独立的 HTML 文件，而不是启动服务器。然后提供一个链接，用户可以点击在浏览器中打开 HTML。
- 由于某种原因，Cowork 配置似乎不利于 Claude 在运行测试后生成评估查看器，所以再次强调一下：无论是在 Cowork 还是 Claude Code 中，运行测试后，你都应该始终生成评估查看器，让人工在你自己评估输入之前查看示例，使用 `generate_review.py`（不要自己编写定制的 html 代码）。事先道歉，但我还是要大写强调：在评估输入之前生成评估查看器。你希望尽快让人工看到这些示例！
- 反馈的工作方式不同：由于没有运行的服务器，查看器的"提交所有审查"按钮会将 `feedback.json` 下载为文件。然后你可以从那里读取它（可能需要先请求访问权限）。
- 打包可以工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 subprocess 通过 `claude -p`，而不是浏览器，但请等到你完全完成 skill 的制作并且用户同意其状态良好后再进行。
- **更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新 skill。请遵循上面 claude.ai 部分中的更新指南。

---

## 参考文件

agents/ 目录包含专业子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` —— 如何针对输出评估断言
- `agents/comparator.md` —— 如何对两个输出进行盲态 A/B 对比
- `agents/analyzer.md` —— 如何分析一个版本为何胜过另一个版本

references/ 目录包含其他文档：
- `references/schemas.md` —— evals.json、grading.json 等的 JSON 结构

---

为强调起见，这里再次重复核心循环：

- 弄清楚 skill 是关于什么的
- 起草或编辑 skill
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终的 skill 并将其返回给用户。

如果你的任务清单上有此类内容，请添加步骤以确保你不会忘记。如果在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py` 以便人工审查测试用例"放入你的任务清单中，以确保它会发生。

祝你好运！