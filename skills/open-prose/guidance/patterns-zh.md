---
role: best-practices
summary: |
  设计模式，用于构建鲁棒、高效且易于维护的 OpenProse 程序。
  在编写新程序或审查现有程序时请阅读此文件。
see-also:
  - prose.md: 执行语义，如何运行程序
  - compiler.md: 完整的语法规则，验证规则
  - antipatterns.md: 应避免的模式
---

# OpenProse 设计模式 (Design Patterns)

本文档整理了经实践证明有效的模式，用于有效地编排 (Orchestrating) AI 代理 (Agents)。每种模式都针对特定的关注点：鲁棒性、成本效率、速度、可维护性或自我改进能力。

---

## 结构模式 (Structural Patterns)

#### parallel-independent-work（并行独立任务）

当任务之间没有数据依赖关系时，请并发执行它们。这可以最大限度地提高吞吐量并缩短实际运行时间。

```prose
# 推荐：独立的调研任务并行运行
parallel:
  market = session "Research market trends"
  tech = session "Research technology landscape"
  competition = session "Analyze competitor products"

session "Synthesize findings"
  context: { market, tech, competition }
```

**备注**：`session "Synthesize findings"` 会等待所有分支完成。总执行时间取决于耗时最长的分支，而不是所有分支时间的总和。

#### fan-out-fan-in（扇出-扇入）

处理集合时，先扇出到并行工作者 (Workers)，然后收集结果。使用 `parallel for` 而不是手动编写多个并行分支。

```prose
let topics = ["AI safety", "interpretability", "alignment", "robustness"]

parallel for topic in topics:
  session "Deep dive research on {topic}"

session "Create unified report from all research"
```

**备注**：这种方式会随着集合大小自然扩展，并保持代码简洁 (DRY)。

#### pipeline-composition（管道组合）

使用管道操作符 (`|`) 链接转换过程，使数据流清晰易读。每个阶段只负责单一职责。

```prose
let candidates = session "Generate 10 startup ideas"

let result = candidates
  | filter:
      session "Is this idea technically feasible? yes/no"
        context: item
  | map:
      session "Expand this idea into a one-page pitch"
        context: item
  | reduce(best, current):
      session "Compare these two pitches, return the stronger one"
        context: [best, current]
```

#### agent-specialization（代理专业化）

定义具有专注领域的专家级代理 (Agents)。专业化代理产生的结果通常优于通用的提示词 (Generalist prompts)。

```prose
agent security-reviewer:
  model: sonnet
  prompt: """
    你是一名安全专家。请只关注：
    - 身份验证 (Authentication) 和授权 (Authorization) 缺陷
    - 注入漏洞 (Injection vulnerabilities)
    - 数据泄露风险 (Data exposure risks)
    请忽略代码风格、性能和其他非安全相关的问题。
  """

agent performance-reviewer:
  model: sonnet
  prompt: """
    你是一名性能工程师。请只关注：
    - 算法复杂度 (Algorithmic complexity)
    - 内存使用模式
    - I/O 瓶颈
    请忽略安全、风格和其他无关问题。
  """
```

#### reusable-blocks（可重用块）

将重复的工作流提取到参数化的 `block` 中。块 (Blocks) 相当于 OpenProse 中的函数。

```prose
block review-and-revise(artifact, criteria):
  let feedback = session "Review {artifact} against {criteria}"
  session "Revise {artifact} based on feedback"
    context: feedback

# 重用此模式
do review-and-revise("the architecture doc", "clarity and completeness")
do review-and-revise("the API design", "consistency and usability")
do review-and-revise("the test plan", "coverage and edge cases")
```

---

## 鲁棒性模式 (Robustness Patterns)

#### bounded-iteration（有界迭代）

务必使用 `max:` 来约束循环 (`loop`)，以防止失控执行。即使是精心设计的条件也可能无法终止。

```prose
# 推荐：显式的迭代上限
loop until **all tests pass** (max: 20):
  session "Identify and fix the next failing test"

# 即使测试从未完全通过，程序也会在 20 次尝试后终止
```

#### graceful-degradation（优雅降级）

在可以接受部分结果的情况下，使用 `on-fail: "continue"`。尽可能收集可用的数据，而不是让整个程序失败。

```prose
parallel (on-fail: "continue"):
  primary = session "Query primary data source"
  backup = session "Query backup data source"
  cache = session "Check local cache"

# 使用执行成功的任何结果继续后续操作
session "Merge available data"
  context: { primary, backup, cache }
```

#### retry-with-backoff（指数退避重试）

外部服务可能会发生暂时性故障。使用指数退避 (Exponential backoff) 进行重试，以处理频率限制 (Rate limits) 和临时停机。

```prose
session "Call external API"
  retry: 5
  backoff: "exponential"
```

对于关键路径，可以将重试与备用方案 (Fallback) 结合使用：

```prose
try:
  session "Call primary API"
    retry: 3
    backoff: "exponential"
catch:
  session "Use fallback data source"
```

#### error-context-capture（错误上下文捕获）

捕获错误上下文以进行智能恢复。错误变量 (`err`) 为诊断或补救会话提供必要信息。

```prose
try:
  session "Deploy to production"
catch as err:
  session "Analyze deployment failure and suggest fixes"
    context: err
  session "Attempt automatic remediation"
    context: err
```

#### defensive-context（防御性检查）

在执行昂贵的操作之前验证假设。廉价的检查可以防止浪费计算资源。

```prose
let prereqs = session "Check all prerequisites: API keys, permissions, dependencies"

if **prerequisites are not met**:
  session "Report missing prerequisites and exit"
    context: prereqs
  throw "Prerequisites not satisfied"

# 只有在先决条件通过时才会运行昂贵的操作
session "Execute main workflow"
```

---

## 成本效率模式 (Cost Efficiency Patterns)

#### model-tiering（模型分层）

根据任务复杂度匹配相应的模型能力：

| 模型 | 最适用于 | 示例 |
|-------|----------|----------|
| **Sonnet 4.5** | 编排 (Orchestration)、控制流、协调 | VM 执行、船长椅模式 (Captain's chair)、工作流路由 |
| **Opus 4.5** | 需要深度推理的高难度工作 | 复杂分析、策略决策、解决新颖问题 |
| **Haiku** | 简单的、显而易见的任务（谨慎使用） | 分类、摘要、格式化 |

**关键洞察**：Sonnet 4.5 擅长 *编排* 代理和管理控制流——它是 OpenProse VM 本身以及协调工作的“船长 (Captain)”代理的理想模型。Opus 4.5 应保留给执行真正困难的智力工作的代理。Haiku 可以处理简单任务，但在重视质量的地方通常应避免使用。

**详细任务-模型映射表：**

| 任务类型 | 推荐模型 | 理由 |
|-----------|-------|-----------|
| 编排、路由、协调 | Sonnet | 速度快，擅长遵循结构化指令 |
| 调查、调试、诊断 | Sonnet | 结构化分析，清单式 (Checklist) 工作 |
| 分流、分类、归类 | Sonnet | 明确的准则，确定性的决策 |
| 代码审查、验证（清单式） | Sonnet | 遵循定义的审查标准 |
| 简单实现、修复 | Sonnet | 应用已知模式 |
| 复杂的跨文件综合 | Opus | 需要在上下文中保持大量信息 |
| 新颖架构、策略规划 | Opus | 需要创造性的问题解决能力 |
| 模糊问题、需求不明确 | Opus | 需要在不确定性中进行推理 |

**准则**：如果你可以为任务编写一份清单，Sonnet 就能胜任。如果任务需要真正的创造力或处理歧义，请使用 Opus。

```prose
agent captain:
  model: sonnet  # 负责编排和协调
  persist: true  # 执行范围持久化（随运行结束而销毁）
  prompt: "你负责协调团队并审查工作"

agent researcher:
  model: opus  # 负责高难度分析工作
  prompt: "你负责进行深度研究和分析"

agent formatter:
  model: haiku  # 简单转换（谨慎使用）
  prompt: "你负责将文本格式化为一致的结构"

agent preferences:
  model: sonnet
  persist: user  # 用户范围持久化（跨项目保留）
  prompt: "你负责记住用户偏好和模式"

# 船长负责编排，专家负责高难度工作
session: captain
  prompt: "Plan the research approach"

let findings = session: researcher
  prompt: "Investigate the technical architecture"

resume: captain
  prompt: "Review findings and determine next steps"
  context: findings
```

#### context-minimization（上下文最小化）

仅传递相关的上下文。过大的上下文会降低处理速度并增加成本。

```prose
# 错误做法：传递所有内容
session "Write executive summary"
  context: [raw_data, analysis, methodology, appendices, references]

# 正确做法：只传递必要的上下文
let key_findings = session "Extract key findings from analysis"
  context: analysis

session "Write executive summary"
  context: key_findings
```

#### early-termination（提前终止）

一旦达成目标，立即退出循环。不要进行不必要的迭代。

```prose
# 每次迭代都会检查条件
loop until **solution found and verified** (max: 10):
  session "Generate potential solution"
  session "Verify solution correctness"
# 当条件满足时立即退出，而不是等到最大迭代次数
```

#### early-signal-exit（信号驱动提前退出）

在进行观察或监控时，一旦获得确定的答案就立即退出，不要等待整个观察窗口结束。

```prose
# 推荐：根据信号退出
let observation = session: observer
  prompt: "观察流。如果检测到阻塞性错误 (Blocking error)，请立即发送信号。"
  timeout: 120s
  early_exit: **blocking_error detected**

# 错误做法：固定的观察窗口
loop 30 times:
  resume: observer
    prompt: "Keep watching..."  # 即使在第 2 次迭代时错误已显而易见
```

**备注**：这种做法尊重信号的即时性，而不是等待任意设定的超时时间。

#### defaults-over-prompts（配置优先于提示）

对于标准配置，使用常量 (`const`) 或环境变量。只有在确实存在变量时才使用 `input` 提示。

```prose
# 推荐：使用合理的默认值
const API_URL = "https://api.example.com"
const TEST_PROGRAM = "# Simple test\nsession 'Hello'"

# 较慢的做法：提示已知值
let api_url = input "Enter API URL"  # 通常是相同的值
let program = input "Enter test program"  # 通常是相同的值
```

**备注**：如果 90% 的运行都使用相同的值，请直接硬编码。需要时允许用户通过 CLI 参数覆盖。

#### race-for-speed（竞速加速）

当任何有效结果都可以满足需求时，并行尝试多种方法，并采用最先成功的那个。

```prose
parallel ("first"):
  session "Try algorithm A"
  session "Try algorithm B"
  session "Try algorithm C"

# 只要任何一种方法完成，就会立即继续
session "Use winning result"
```

#### batch-similar-work（类似任务批处理）

将相似的操作分组以分摊开销。一个具有结构化输出的 `session` 优于许多小的 `session`。

```prose
# 低效：许多小的会话
for file in files:
  session "Analyze {file}"

# 高效：批处理分析
session "Analyze all files and return structured findings for each"
  context: files
```

---

## 自我改进模式 (Self-Improvement Patterns)

#### self-verification-in-prompt（提示词内自我验证）

对于本来需要单独验证者的任务，可以将验证作为提示词的最后一步包含在内。这可以在保持严谨性的同时节省一轮往返 (Round-trip)。

```prose
# 推荐：工作与自我验证相结合
agent investigator:
  model: sonnet
  prompt: """诊断错误。
  1. 检查代码路径
  2. 检查日志和状态
  3. 形成假设
  4. 在输出之前：验证你的证据是否支持你的结论。

  只有在确信时才输出。如果不确定，请说明缺少什么。"""

# 较慢的做法：单独的验证代理
let diagnosis = session: researcher
  prompt: "Investigate the error"
let verification = session: verifier
  prompt: "Verify this diagnosis"  # 额外的一轮往返
  context: diagnosis
```

**备注**：当你需要真正的对抗性审查（不同视角）时，请使用单独的验证者；但对于自洽性检查 (Self-consistency checks)，请将验证集成到提示词中。

#### iterative-refinement（迭代优化）

使用反馈循环逐步改进输出。每次迭代都在上一次的基础上进行。

```prose
let draft = session "Create initial draft"

loop until **draft meets quality bar** (max: 5):
  let critique = session "Critically evaluate this draft"
    context: draft
  draft = session "Improve draft based on critique"
    context: [draft, critique]

session "Finalize and publish"
  context: draft
```

#### multi-perspective-review（多视角审查）

在综合之前收集多元化的观点。不同的视角可以发现不同的问题。

```prose
parallel:
  user_perspective = session "Evaluate from end-user viewpoint"
  tech_perspective = session "Evaluate from engineering viewpoint"
  business_perspective = session "Evaluate from business viewpoint"

session "Synthesize feedback and prioritize improvements"
  context: { user_perspective, tech_perspective, business_perspective }
```

#### adversarial-validation（对抗性验证）

使用一个代理来挑战另一个代理的工作。对抗性压力可以提高鲁棒性。

```prose
let proposal = session "Generate proposal"

let critique = session "Find flaws and weaknesses in this proposal"
  context: proposal

let defense = session "Address each critique with evidence or revisions"
  context: [proposal, critique]

session "Produce final proposal incorporating valid critiques"
  context: [proposal, critique, defense]
```

#### consensus-building（共识构建）

对于关键决策，要求多个独立评估者达成一致。

```prose
parallel:
  eval1 = session "Independently evaluate the solution"
  eval2 = session "Independently evaluate the solution"
  eval3 = session "Independently evaluate the solution"

loop until **evaluators agree** (max: 3):
  session "Identify points of disagreement"
    context: { eval1, eval2, eval3 }
  parallel:
    eval1 = session "Reconsider position given other perspectives"
      context: { eval1, eval2, eval3 }
    eval2 = session "Reconsider position given other perspectives"
      context: { eval1, eval2, eval3 }
    eval3 = session "Reconsider position given other perspectives"
      context: { eval1, eval2, eval3 }

session "Document consensus decision"
  context: { eval1, eval2, eval3 }
```

---

## 可维护性模式 (Maintainability Patterns)

#### descriptive-agent-names（描述性代理名称）

根据代理的角色而不是实现来命名。名称应传达其用途。

```prose
# 推荐：基于角色的命名
agent code-reviewer:
agent technical-writer:
agent data-analyst:

# 错误做法：基于实现的命名
agent opus-agent:
agent session-1-handler:
agent helper:
```

#### prompt-as-contract（提示词即契约）

编写指定预期输入和输出的提示词。清晰的契约可以防止误解。

```prose
agent json-extractor:
  model: haiku
  prompt: """
    从文本中提取结构化数据。

    输入：包含实体信息的非结构化文本
    输出：包含以下字段的 JSON 对象：name, date, amount, status

    如果无法确定某个字段，请使用 null。
    严禁虚构输入中不存在的信息。
  """
```

#### separation-of-concerns（关注点分离）

每个会话 (Session) 应只做好一件事。组合简单的会话，而不是创建复杂的会话。

```prose
# 推荐：每个会话单一职责
let data = session "Fetch and validate input data"
let analysis = session "Analyze data for patterns"
  context: data
let recommendations = session "Generate recommendations from analysis"
  context: analysis
session "Format recommendations as report"
  context: recommendations

# 错误做法：万能会话 (God session)
session "Fetch data, analyze it, generate recommendations, and format a report"
```

#### explicit-context-flow（显式上下文流）

通过显式的上下文传递使数据流可见。避免依赖隐式的对话历史。

```prose
# 推荐：显式流
let step1 = session "First step"
let step2 = session "Second step"
  context: step1
let step3 = session "Third step"
  context: [step1, step2]

# 错误做法：隐式流（依赖对话状态）
session "First step"
session "Second step using previous results"
session "Third step using all previous"
```

---

## 性能模式 (Performance Patterns)

#### lazy-evaluation（惰性求值）

将昂贵的操作推迟到需要其结果时。不要计算可能用不到的内容。

```prose
session "Assess situation"

if **detailed analysis needed**:
  # 仅在必要时执行昂贵的操作
  parallel:
    deep_analysis = session "Perform deep analysis"
      model: opus
    historical = session "Gather historical comparisons"
  session "Comprehensive report"
    context: { deep_analysis, historical }
else:
  session "Quick summary"
    model: haiku
```

#### progressive-disclosure（渐进式披露）

从快速、廉价的操作开始。仅在需要时逐步升级到昂贵的操作。

```prose
# 第 1 层：快速筛选 (Haiku)
let initial = session "Quick assessment"
  model: haiku

if **needs deeper review**:
  # 第 2 层：中等分析 (Sonnet)
  let detailed = session "Detailed analysis"
    model: sonnet
    context: initial

  if **needs expert review**:
    # 第 3 层：深度推理 (Opus)
    session "Expert-level analysis"
      model: opus
      context: [initial, detailed]
```

#### work-stealing（工作窃取）

使用 `parallel ("any", count: N)` 从工作池中尽可能快地获取结果。

```prose
# 从 5 次并行尝试中尽可能快地获取 3 个好的想法
parallel ("any", count: 3, on-fail: "ignore"):
  session "Generate creative solution approach 1"
  session "Generate creative solution approach 2"
  session "Generate creative solution approach 3"
  session "Generate creative solution approach 4"
  session "Generate creative solution approach 5"

session "Select best from the first 3 completed"
```

---

## 组合模式 (Composition Patterns)

#### workflow-template（工作流模板）

创建编码了整个工作流模式的块 (Blocks)。使用不同的参数进行实例化。

```prose
block research-report(topic, depth):
  let research = session "Research {topic} at {depth} level"
  let analysis = session "Analyze findings about {topic}"
    context: research
  let report = session "Write {depth}-level report on {topic}"
    context: [research, analysis]

# 针对不同需求进行实例化
do research-report("market trends", "executive")
do research-report("technical architecture", "detailed")
do research-report("competitive landscape", "comprehensive")
```

#### middleware-pattern（中间件模式）

使用横切关注点（如日志记录、计时或验证）对会话进行包装。

```prose
block with-validation(task, validator):
  let result = session "{task}"
  let valid = session "{validator}"
    context: result
  if **validation failed**:
    throw "Validation failed for: {task}"

do with-validation("Generate SQL query", "Check SQL for injection vulnerabilities")
do with-validation("Generate config file", "Validate config syntax")
```

#### circuit-breaker（熔断器）

在反复失败后，停止尝试并快速失败。防止级联故障。

```prose
let failures = 0
let max_failures = 3

loop while **service needed and failures < max_failures** (max: 10):
  try:
    session "Call external service"
    # 成功时重置
    failures = 0
  catch:
    failures = failures + 1
    if **failures >= max_failures**:
      session "Circuit open - using fallback"
      throw "Service unavailable"
```

---

## 可观测性模式 (Observability Patterns)

#### checkpoint-narration（检查点叙事）

对于长流程的工作流，发出进度标记。有助于调试和监控。

```prose
session "Phase 1: Data Collection"
# ... 收集工作 ...

session "Phase 2: Analysis"
# ... 分析工作 ...

session "Phase 3: Report Generation"
# ... 报告生成 ...

session "Phase 4: Quality Assurance"
# ... 质检工作 ...
```

#### structured-output-contracts（结构化输出契约）

请求结构化输出，以便能够可靠地进行解析和验证。

```prose
agent structured-reviewer:
  model: sonnet
  prompt: """
    务必使用以下确切的 JSON 结构进行响应：
    {
      "verdict": "pass" | "fail" | "needs_review",
      "issues": [{"severity": "high"|"medium"|"low", "description": "..."}],
      "suggestions": ["..."]
    }
  """

let review = session: structured-reviewer
  prompt: "Review this code for security issues"
```

---

## 总结

最高效的 OpenProse 程序结合了以下模式：

1. **结构**：并行化独立工作，使用块 (Blocks) 进行重用。
2. **鲁棒性**：约束循环，处理错误，重试暂时性故障。
3. **效率**：分层使用模型，最小化上下文，提前终止。
4. **质量**：通过迭代获取多视角，进行对抗性验证。
5. **可维护性**：清晰命名，分离关注点，使数据流显式化。

请根据具体的约束条件选择模式。快速原型开发优先考虑速度而非鲁棒性。生产环境工作流优先考虑可靠性而非成本。研究探索优先考虑彻底性而非效率。
