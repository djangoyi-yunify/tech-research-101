---
name: skill-creator
description: 创建新技能、修改和改进现有技能，以及测量技能性能。当用户想要从头创建技能、编辑或优化现有技能、运行评估来测试技能、使用方差分析对技能性能进行基准测试，或优化技能描述以提高触发准确性时使用此技能。
---

# Skill Creator

用于创建新技能并迭代改进它们的能力。

从高层次来看，创建技能的过程如下：

- 决定你想让技能做什么，以及大致如何实现
- 撰写技能的初稿
- 创建一些测试提示，并在带有技能访问权限的 Claude 上运行它们
- 帮助用户定性和定量地评估结果
  - 在运行后台进行的同时，如果还没有定量评估就起草一些（如果已有，可以直接使用或在你认为需要改进时进行修改）。然后向用户解释这些评估（如果已存在，则解释现有的评估）
  - 使用 `eval-viewer/generate_review.py` 脚本向用户展示结果供其查看，同时让他们查看定量指标
- 根据用户对结果的评估反馈重写技能（如果定量基准测试中出现任何明显的缺陷，也一并考虑）
- 重复直到你满意为止
- 扩大测试集并尝试更大规模的测试

使用此技能时，你的工作是确定用户处于流程的哪个阶段，然后帮助他们推进这些阶段。例如，用户可能说"我想为 X 创建一个技能"。你可以帮助他们明确需求、撰写初稿、编写测试用例、确定评估方式、运行所有提示，然后重复迭代。

另一方面，如果用户已经有了一份技能初稿，你可以直接进入评估/迭代环节。

当然，你应该始终保持灵活性，如果用户说"我不需要运行一堆评估，一起随意探索吧"，你也可以那样做。

然后在技能完成后（同样，顺序是灵活的），你也可以运行技能描述优化器，我们有专门的脚本来做这件事，以优化技能的触发机制。

明白了吗？明白了。

## 与用户沟通

Skill Creator 可能被各种熟悉程度不同的用户使用——从对编码术语了解不多的人到专业人士。你应该注意上下文线索来理解如何措辞！在默认情况下，给你一些参考：

- "evaluation"和"benchmark"是边缘情况，但可以使用
- 对于"JSON"和"assertion"，你需要看到用户明显表现出知道这些术语的含义后，才能不加解释地使用它们

如果有疑问，可以简要解释术语；如果不确定用户是否理解某个概念，可以加上简短定义来澄清。

---

## 创建技能

### 明确意图

首先理解用户的意图。对话中可能已经包含了用户想要捕捉的工作流程（例如，他们说"把这个变成一个技能"）。如果是这样，先从对话历史中提取答案——使用的工具、步骤顺序、用户的修改、观察到的输入/输出格式。用户可能需要补充空白内容，在进入下一步之前应该确认。

1. 这个技能应该让 Claude 做什么？
2. 这个技能应该在什么时候触发？（什么样的用户措辞/上下文）
3. 期望的输出格式是什么？
4. 我们是否应该设置测试用例来验证技能有效？具有客观可验证输出的技能（文件转换、数据提取、代码生成、固定工作流程步骤）受益于测试用例。输出主观的技能（写作风格、艺术）通常不需要测试用例。根据技能类型建议适当的默认值，但由用户决定。

### 访谈与研究

主动询问边缘情况、输入/输出格式、示例文件、成功标准和依赖项等问题。等到这部分完善后再编写测试提示。

检查可用的 MCP——如果对研究有用（搜索文档、查找类似技能、查阅最佳实践），如果有子代理可用就通过子代理并行研究，否则直接内联研究。做好准备，带着上下文内容来减少用户的负担。

### 编写 SKILL.md

基于用户访谈，填写以下组件：

- **name**：技能标识符
- **description**：何时触发，做什么。这是主要的触发机制——要同时包含技能的功能描述和具体的使用场景。所有"何时使用"的信息都放在这里，不放在正文中。注：目前 Claude 有"触发不足"的倾向——在技能可能有用时不使用它们。为了解决这个问题，请让技能描述稍微"强势"一些。例如，不要写成"如何构建简单的快速仪表板来显示 Anthropic 内部数据"，你可以写成"如何构建简单的快速仪表板来显示 Anthropic 内部数据。当用户提到仪表板、数据可视化、内部指标，或想要展示任何类型的公司数据时，一定要使用此技能，即使他们没有明确要求'仪表板'。"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
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

技能使用三级加载系统：

1. **元数据**（名称 + 描述）- 始终在上下文中（~100词）
2. **SKILL.md 主体** - 技能触发时在上下文中（理想<500行）
3. **捆绑资源** - 按需使用（无限，脚本可在不加载的情况下执行）

这些字数统计仅为近似值，必要时可以适当增加。

**关键模式：**

- 保持 SKILL.md 在 500 行以内；如果接近此限制，请添加额外的层级结构，并提供清晰的指引，说明使用该技能的大模型下一步应该去哪里跟进。
- 在 SKILL.md 中清晰引用文件，并提供何时阅读的指导
- 对于大型参考文件（>300行），包含目录

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

#### 缺乏惊喜原则

这不用说，但技能不得包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。技能的描述内容不应让用户对其意图感到意外。不要配合创建误导性技能或旨在促进未授权访问、数据泄露或其他恶意活动的技能请求。不过，像"扮演 XYZ 角色"这类是可以的。

#### 写作模式

在指令中优先使用祈使句。

**定义输出格式** - 你可以这样写：

```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**示例模式** - 包含示例会很有帮助。你可以这样格式化它们（但如果示例中有"Input"和"Output"，可能需要稍微调整格式）：

```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

I understand this is a skill draft about how to write effective AI skill documentation. The guidance emphasizes:

- Explaining the *why* behind suggestions rather than using prescriptive "MUST" language
- Using theory of mind to make skills generalizable rather than overly narrow
- Iterating with fresh eyes to improve clarity

---

**Test Cases for This Skill**

Here are a few realistic test prompts I'd like to try — the kinds of things a real user would actually say when looking for help with skill development:

**1.** "I want to create a skill that helps me debug Python errors step by step"

**2.** "Help me write a skill for summarizing research papers in a consistent format"

**3.** "I need a skill that explains code to beginners without assuming prior knowledge"

Do these look right? Would you like to add more, or adjust any of them before I save them to `evals/evals.json`?

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

有关完整 schema（包括你稍后会添加的 `assertions` 字段），请参见 `references/schemas.md`。

## 运行和评估测试用例

本节是一个连续的流程——请勿中途停止。请勿使用 `/skill-test` 或任何其他测试相关的 skill。

请将结果放在 skill 目录的同级目录 `<skill-name>-workspace/` 下。在 workspace 中，按迭代次数组织结果（`iteration-1/`、`iteration-2/` 等），在每个迭代目录下，每个测试用例对应一个目录（`eval-0/`、`eval-1/` 等）。不要提前创建所有目录——边走边创建即可。

### 步骤 1：在同一轮中启动所有运行（带 skill 的和基线的）

对于每个测试用例，在同一轮中启动两个子 agent——一个有 skill，一个没有。这很重要：请勿先启动带 skill 的运行，然后再回来处理基线。同时启动所有任务，这样它们能在相近的时间完成。

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

- **创建新 skill**：完全不使用 skill。使用相同 prompt，不指定 skill 路径，保存至 `without_skill/outputs/`。
- **改进现有 skill**：使用旧版本。在编辑之前，先对 skill 进行快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然后让 baseline subagent 指向该快照。保存至 `old_skill/outputs/`。

为每个测试用例编写 `eval_metadata.json`（断言部分可暂时留空）。根据测试内容为每个 eval 赋予描述性名称——不要仅仅使用 "eval-0" 这样的名称。目录名也使用该名称。如果本次迭代使用了新的或修改过的 eval prompt，需要为每个新的 eval 目录创建这些文件——不要假设它们会从之前的迭代中继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第 2 步：运行进行期间起草断言

不要只是等待运行完成——你可以利用这段时间做些有意义的事。为每个测试用例起草定量断言，并向用户解释它们。如果 `evals/evals.json` 中已存在断言，请审查它们并解释它们检查的内容。

好的断言应该是客观可验证的，并且具有描述性名称——它们应该在 benchmark 查看器中清晰易读，这样用户在浏览结果时能立即理解每个断言检查的内容。主观技能（写作风格、设计质量）更适合定性评估——不要将断言强加于需要人工判断的事物上。

一旦起草完成，请更新 `eval_metadata.json` 文件和 `evals/evals.json` 中的断言。同时向用户解释他们在查看器中会看到什么——包括定性输出和定量 benchmark。

### 第 3 步：运行完成后捕获时序数据

当每个 subagent 任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。请立即将此数据保存到运行目录的 `timing.json` 中：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是唯一捕获这些数据的机会——它通过任务通知传递，不会被持久化存储在其他地方。实时处理每条通知，而不是尝试批量处理。

### 步骤 4：评分、汇总并启动查看器

所有运行完成后：

1. **为每次运行评分** — 启动一个 grader 子代理（或内联评分），读取 `agents/grader.md` 并根据输出评估每个断言。将结果保存到每次运行目录的 `grading.json` 中。grading.json 的 expectations 数组必须使用 `text`、`passed` 和 `evidence` 字段（而不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些精确的字段名。对于可以通过编程检查的断言，编写并运行脚本而不是目测检查——脚本更快、更可靠，并且可以在迭代中重复使用。

2. **汇总到基准测试** — 从 skill-creator 目录运行聚合脚本：

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
```

这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置的 pass_rate、time 和 tokens，以及 mean ± stddev 和 delta。如果手动生成 benchmark.json，请参阅 `references/schemas.md` 了解查看器期望的确切 schema。
将每个 with_skill 版本放在其基准版本之前。

3. **进行分析师分析** — 阅读基准数据，发现汇总统计可能隐藏的模式。参见 `agents/analyzer.md`（"分析基准结果"部分）了解需要关注的内容——比如无论技能如何都总是通过的断言（无区分度）、高方差评估（可能不稳定）以及时间/token 权衡。

4. **启动查看器**，同时加载定性输出和定量数据：

```bash
nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
```

对于第 2 次及之后的迭代，还需要传入 `--previous-workspace <workspace>/iteration-<N-1>`。

**协作/无头环境**：如果 `webbrowser.open()` 不可用或环境没有显示器，可以使用 `--static <output_path>` 来生成独立的 HTML 文件，而不是启动服务器。当用户点击"Submit All Reviews"时，反馈会被下载为 `feedback.json` 文件。下载后，将 `feedback.json` 复制到工作目录中，以便下次迭代拾取。

注意：请使用 `generate_review.py` 来创建查看器，无需编写自定义 HTML。

5. **告知用户**类似信息："我已经在浏览器中打开了结果。有两个标签页——'Outputs' 让你逐个点击查看每个测试用例并留下反馈，'Benchmark' 显示定量比较结果。完成后，回到这里告诉我。"

### 用户在查看器中看到的内容

"Outputs" 标签页一次显示一个测试用例：

- **Prompt**：给出的任务
- **Output**：技能生成的文件，尽可能内联渲染
- **Previous Output**（第 2 次及之后的迭代）：折叠部分，显示上一次迭代的输出
- **Formal Grades**（如果运行了评分）：折叠部分，显示断言通过/失败情况
- **Feedback**：自动保存的文本框，用户输入时即时保存
- **Previous Feedback**（第 2 次及之后的迭代）：上次用户的评论，显示在文本框下方

"Benchmark" 标签页显示统计摘要：每个配置的通过率、时间和 token 使用情况，包括每次评估的细分组和分析师观察。

导航通过上一个/下一个按钮或方向键实现。完成后，点击"Submit All Reviews"将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

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

空反馈意味着用户认为没有问题。请专注于改进用户有具体抱怨的测试用例。

完成测试后关闭 viewer 服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进技能

这是循环的核心环节。你已经运行了测试用例，用户也审查了结果，现在需要根据他们的反馈来提升技能。

### 如何思考改进

1. **从反馈中提炼通用规律。** 这里的大局是：我们正在尝试创建可以在无数次（也许是字面意义上的无数次，甚至更多）不同 prompt 中使用的技能。你和用户之所以一遍又一遍地迭代少数几个示例，只是为了加快速度。用户对这些示例了如指掌，可以快速评估新的输出。但如果你和用户合作开发的技能只能应对这些示例，那它就毫无用处。与其做些零碎的过拟合修改，或施加过于严格的 MUST 约束，不如尝试分支出去，使用不同的类比，或推荐不同的工作模式。尝试的成本相对较低，说不定你就能发现真正出色的方案。

2. **保持 prompt 精简。** 删除没有发挥作用的内容。务必阅读转录文本，而不仅仅是最终输出——如果看起来技能在浪费模型大量时间做无效的事情，你可以尝试删除导致这种行为的技能部分，然后观察结果。

3. **解释原因。** 努力解释你让模型执行的每项操作的 **原因**。如今的 LLM 非常智能。它们有很好的心智理论，当给定的框架得当，可以超越机械指令，真正把事情做成。即使用户的反馈简洁或带有情绪，也要尝试真正理解任务本身，理解用户为什么写他们写的内容，理解他们实际写了什么，然后将这种理解传达给指令。如果你发现自己满篇都在用大写的 ALWAYS 或 NEVER，或者使用过于僵化的结构，这就是一个黄色警告信号——如果可能的话，重新措辞并解释推理过程，让模型理解你要求做的事情为什么重要。这是一种更人性化、更有力和更有效的方法。

4. **寻找测试用例间的重复工作。** 阅读测试运行的转录文本，注意是否有子代理独立编写了相似的辅助脚本或对某些内容采用了相同的多步骤方法。如果 3 个测试用例都导致子代理编写了 `create_docx.py` 或 `build_chart.py`，这就是一个强烈的信号，表明技能应该打包这个脚本。只写一次，放在 `scripts/` 中，然后告诉技能使用它。这样可以为以后每次调用节省重新发明轮子的时间。

这个任务相当重要（我们正在尝试创造每年数十亿美元的经济价值！）你的思考时间不是瓶颈；慢慢来，真正深入思考。我建议先写一份修订草案，然后重新审视它并改进。真正努力走进用户的内心，理解他们想要什么、需要什么。

### 迭代循环

改进技能后：

1. 将你的改进应用到技能中
2. 将所有测试用例重新运行到一个新的 `iteration-<N+1>/` 目录中，包括基线运行。如果你正在创建新技能，基线始终是 `without_skill`（无技能）——它在整个迭代过程中保持不变。如果你在改进现有技能，运用你的判断力决定什么作为基线有意义：用户最初使用的原始版本，还是上一个迭代版本。
3. 使用 `--previous-workspace` 指向之前的迭代来启动审查者
4. 等待用户审查并告诉你他们完成了
5. 阅读新的反馈，再次改进，重复

继续下去直到：
- 用户表示满意
- 反馈全是空的（一切都看起来很好）
- 你没有取得有意义的进展

---

## 高级：盲测对比

在某些情况下，你需要对两个版本的技能进行更严格的比较（例如，用户问"新版本真的更好吗？"），有一个盲测对比系统。阅读 `agents/comparator.md` 和 `agents/analyzer.md` 了解详情。基本思路是：把两个输出交给一个独立的代理，不告诉它哪个是哪个，让它评判质量。然后分析获胜者为什么获胜。

这是可选的，需要子代理，大多数用户不需要。人工审查循环通常已经足够。

---

## 描述优化

SKILL.md frontmatter 中的 description 字段是决定 Claude 是否调用技能的主要机制。在创建或改进技能后，主动提出优化描述以提高触发准确性。

### 步骤 1：生成触发评估查询

创建 20 个评估查询——包括应该触发和不应该触发的混合。将结果保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

I need some clarification to create the evaluation queries for you. I don't have information about the skill you're working with. Could you please provide:

1. **The skill name** - e.g., "PDF manipulation", "Excel processing", etc.
2. **The skill description** - what does this skill do? What are its capabilities?
3. **Any specific requirements** - are there particular edge cases, file formats, or use cases you want covered?

Once you provide this information, I can create the should-trigger and should-not-trigger queries following the template and guidelines you've outlined.

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

Thanks for sharing this workflow description! This explains the automated skill description optimization process:

**Key points:**
- Uses model ID from current session for accurate testing
- 60/40 train/test split to prevent overfitting
- 3 runs per query for reliable trigger rates
- Up to 5 iterations of evaluation + improvement proposals
- Final selection based on **test score**, not train score

**Important triggering insight:**
Skills only trigger for tasks Claude *can't easily handle on its own* — simple queries like "read file X" won't trigger skills even with a perfect description. Eval queries need to be substantive and complex enough that consulting a skill would genuinely help.

---

Since you mentioned you're going to try something first — if you need help optimizing a skill's description or working through any of these steps, just let me know. Otherwise, I'll wait until you have results or run into issues.

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包完成后，将用户引导至生成的 `.skill` 文件路径，以便他们安装。

---

## Claude.ai 专用说明

在 Claude.ai 中，核心工作流程相同（起草 → 测试 → 审查 → 改进 → 重复），但由于 Claude.ai 没有子代理，一些机制会有所变化。以下是需要调整的内容：

**运行测试用例**：没有子代理意味着无法并行执行。对于每个测试用例，阅读技能的 SKILL.md，然后按照其说明完成测试提示。逐个进行。这比独立子代理的测试方法（你编写了技能同时也在运行它，拥有完整的上下文）不够严格，但这是一个有用的健全性检查——人工审查步骤可以弥补这一点。跳过基线运行——直接使用技能完成任务。

**审查结果**：如果你无法打开浏览器（例如 Claude.ai 的虚拟机没有显示器，或者你在远程服务器上），完全跳过浏览器审查。直接在对话中展示结果。对于每个测试用例，显示提示和输出。如果输出是用户需要查看的文件（如 .docx 或 .xlsx），保存到文件系统并告知他们位置，以便他们可以下载检查。在线征求反馈："看起来怎么样？有什么需要修改的吗？"

**基准测试**：跳过定量基准测试——它依赖基线比较，没有子代理的情况下没有意义。专注于用户的定性反馈。

**迭代循环**：与之前相同——改进技能，重新运行测试用例，征求反馈——只是中间没有浏览器审查。如果你有文件系统，仍然可以将结果组织到迭代目录中。

**描述优化**：此部分需要 `claude` CLI 工具（特别是 `claude -p`），仅在 Claude Code 中可用。如果在 Claude.ai 上，跳过此步骤。

**盲对比**：需要子代理。跳过。

**打包**：`package_skill.py` 脚本在有 Python 和文件系统的任何地方都可以运行。在 Claude.ai 上，你可以运行它，用户可以下载生成的 `.skill` 文件。

**更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。在这种情况下：
- **保留原始名称。** 注意技能的目录名称和 `name` frontmatter 字段——保持不变。例如，如果已安装的技能是 `research-helper`，则输出 `research-helper.skill`（不是 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装技能的路径可能是只读的。复制到 `/tmp/skill-name/`，在那里编辑，然后从副本打包。
- **如果手动打包，先在 `/tmp/` 中暂存**，然后复制到输出目录——由于权限限制，直接写入可能会失败。

---

## Cowork 专用说明

如果你在 Cowork 中，主要需要了解的是：

- 你有子代理，所以主要工作流程（并行生成测试用例、运行基线、评分等）都可以工作。（但是，如果你遇到严重的超时问题，按顺序运行测试提示是可以的。）
- 你没有浏览器或显示器，所以在生成评估查看器时，使用 `--static <output_path>` 写入独立的 HTML 文件，而不是启动服务器。然后提供链接，用户可以点击在浏览器中打开 HTML。
- 无论出于什么原因，Cowork 环境似乎会让 Claude 在运行测试后不太愿意生成评估查看器，所以再次强调：无论你在 Cowork 还是 Claude Code 中，运行测试后都应该始终生成评估查看器，以便人类在修改技能之前查看示例并尝试进行修正，使用 `generate_review.py`（而不是自己编写花哨的 html 代码）。提前道歉，但我还是要大写：**在评估输入之前先生成评估查看器**！你想让人类尽快看到这些！
- 反馈的工作方式不同：由于没有运行中的服务器，查看器的"提交所有审查"按钮将下载 `feedback.json` 作为文件。然后你可以从这里读取它（可能需要先请求访问权限）。
- 打包功能可以正常工作——`package_skill.py` 只需要 Python 和文件系统。
- 描述优化（`run_loop.py` / `run_eval.py`）在 Cowork 中应该可以正常工作，因为它使用 subprocess 调用 `claude -p`，而不是浏览器，但请等到你和用户都认为技能完善后再进行。
- **更新现有技能**：用户可能要求你更新现有技能，而不是创建新技能。遵循上面 claude.ai 部分中的更新指导。

---

## 参考文件

agents/ 目录包含专门子代理的说明。在需要生成相关子代理时阅读它们。

- `agents/grader.md` — 如何针对输出评估断言
- `agents/comparator.md` — 如何对两个输出进行盲 A/B 比较
- `agents/analyzer.md` — 如何分析一个版本为何胜出

references/ 目录有额外的文档：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 结构

---

再重复一遍核心循环以强调：

- 明确技能的功能
- 起草或编辑技能
- 在测试提示上运行 claude-with-access-to-the-skill
- 与用户一起评估输出：
  - 创建 benchmark.json 并运行 `eval-viewer/generate_review.py` 帮助用户审查它们
  - 运行定量评估
- 重复直到你和用户都满意
- 打包最终技能并返回给用户。

如果可以的话，请在待办事项列表中添加步骤，以确保不会忘记。如果你在 Cowork 中，请特别将"创建 evals JSON 并运行 `eval-viewer/generate_review.py`，以便人类审查测试用例"添加到待办事项列表中。

祝你好运！