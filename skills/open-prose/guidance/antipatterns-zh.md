---
role: antipatterns
summary: |
  OpenProse 程序中常见的错误和应避免的模式。
  阅读此文件以识别并修复有问题的代码模式。
see-also:
  - prose.md: 执行语义，如何运行程序
  - compiler.md: 完整的语法规则，验证规则
  - patterns.md: 推荐的设计模式
---

# OpenProse 反模式 (Antipatterns)

本文档整理了会导致程序脆弱、昂贵、缓慢或难以维护的模式。每种反模式都包含识别准则和修复建议。

---

## 结构反模式 (Structural Antipatterns)

#### god-session（万能会话）

单个会话 (Session) 试图完成所有工作。万能会话难以调试，无法并行化，且产生的结果不一致。

```prose
# 错误做法：一个会话做太多事情
session """
  阅读仓库中的所有代码。
  识别安全漏洞。
  查找性能瓶颈。
  检查代码风格违规。
  生成综合报告。
  为每个问题提出修复建议。
  按严重程度排序。
  创建补救计划。
"""
```

**为什么不好**：该会话没有明确的完成标准。它混合了本可以并行处理的不同关注点。任何一处失败都会导致整个任务失败。

**修复方案**：分解为专注的会话：

```prose
# 推荐：专注的会话
parallel:
  security = session "Identify security vulnerabilities"
  perf = session "Find performance bottlenecks"
  style = session "Check for style violations"

session "Synthesize findings and prioritize by severity"
  context: { security, perf, style }

session "Create remediation plan"
```

#### sequential-when-parallel（串行运行并行任务）

本可以并发运行的独立操作却按顺序执行。浪费了实际运行时间。

```prose
# 错误做法：串行执行独立的调研任务
let market = session "Research market"
let tech = session "Research technology"
let competition = session "Research competition"

session "Synthesize"
  context: [market, tech, competition]
```

**为什么不好**：总耗时是所有调研任务时间的总和。每个会话都在不必要地等待前一个会话完成。

**修复方案**：并行化独立任务：

```prose
# 推荐：并行执行独立任务
parallel:
  market = session "Research market"
  tech = session "Research technology"
  competition = session "Research competition"

session "Synthesize"
  context: { market, tech, competition }
```

#### spaghetti-context（面条式上下文）

上下文传递混乱，缺乏清晰的数据流。使程序难以理解和修改。

```prose
# 错误做法：不清楚到底使用了哪些上下文
let a = session "Step A"
let b = session "Step B"
  context: a
let c = session "Step C"
  context: [a, b]
let d = session "Step D"
  context: [a, b, c]
let e = session "Step E"
  context: [a, c, d]  # 为什么不传 b？
let f = session "Step F"
  context: [a, b, c, d, e]  # 传所有内容？
```

**为什么不好**：不清楚哪些会话依赖于哪些输出。难以进行并行化或重构。

**修复方案**：将上下文最小化为实际的依赖项：

```prose
# 推荐：清晰、最小化的依赖关系
let research = session "Research"
let analysis = session "Analyze"
  context: research
let recommendations = session "Recommend"
  context: analysis  # 只需要分析结果，不需要原始研究数据
let report = session "Report"
  context: recommendations
```

#### parallel-then-synthesize（过度并行后综合）

为相关的分析工作产生并行代理，然后进行综合；而实际上，单个专注的代理可以更高效地完成整个工作。

```prose
# 反模式：并行调查 + 综合
parallel:
  code = session "Analyze code path"
  logs = session "Analyze logs"
  context = session "Analyze execution context"

synthesis = session "Synthesize all findings"
  context: { code, logs, context }
# 4 次 LLM 调用，协调开销大，上下文被碎片化
```

**为什么不好**：对于共同推导出一个结论的相关分析，协调开销和上下文碎片化往往抵消了并行的好处。每个并行代理只看到了局部情况。

**修复方案**：使用单个具有多步指令的专注代理：

```prose
# 推荐：单个综合调查员
diagnosis = session "Investigate the error"
  prompt: """进行全面分析：
  1. 检查产生错误的代码路径
  2. 检查日志中的时间和状态
  3. 审查执行上下文
  最后综合成统一的诊断结果。"""
# 1 次 LLM 调用，拥有完整上下文，无协调成本
```

**何时适合并行**：当分析任务真正独立时（例如安全性 vs 性能）、当你需要不应互相影响的多元化视角时，或者当工作量巨大且通过分工能显著获益时。

#### copy-paste-workflows（复制粘贴工作流）

重复使用会话序列而不是使用块 (Blocks)。这会导致更改不一致并增加维护负担。

```prose
# 错误做法：重复的工作流
session "Security review of module A"
session "Performance review of module A"
session "Synthesize reviews of module A"

session "Security review of module B"
session "Performance review of module B"
session "Synthesize reviews of module B"

session "Security review of module C"
session "Performance review of module C"
session "Synthesize reviews of module C"
```

**为什么不好**：如果需要更改工作流，你必须在所有地方进行修改。很容易遗漏某一处。

**修复方案**：提取到 `block` 中：

```prose
# 推荐：可重用的块
block review-module(module):
  parallel:
    sec = session "Security review of {module}"
    perf = session "Performance review of {module}"
  session "Synthesize reviews of {module}"
    context: { sec, perf }

do review-module("module A")
do review-module("module B")
do review-module("module C")
```

---

## 鲁棒性反模式 (Robustness Antipatterns)

#### unbounded-loop（无界循环）

没有设置最大迭代次数的循环。如果条件永远无法满足，它可能会永远运行下去。

```prose
# 错误做法：没有逃生出口
loop until **the code is perfect**:
  session "Improve the code"
```

**为什么不好**：“完美”可能永远无法达成。程序可能会无限运行，消耗资源。

**修复方案**：务必指定 `max:`：

```prose
# 推荐：有界迭代
loop until **the code is perfect** (max: 10):
  session "Improve the code"
```

#### optimistic-execution（乐观执行）

假设一切都会成功。对于可能失败的操作没有任何错误处理。

```prose
# 错误做法：没有错误处理
session "Call external API"
session "Process API response"
session "Store results in database"
session "Send notification"
```

**为什么不好**：如果 API 调用失败，后续会话将接收不到有效的输入，导致静默损坏。

**修复方案**：显式处理失败：

```prose
# 推荐：错误处理
try:
  let response = session "Call external API"
    retry: 3
    backoff: "exponential"
  session "Process API response"
    context: response
catch as err:
  session "Handle API failure gracefully"
    context: err
```

#### ignored-errors（忽略错误）

在失败确实很重要的情况下使用 `on-fail: "ignore"`。这会掩盖本应显现的问题。

```prose
# 错误做法：忽略重要的失败
parallel (on-fail: "ignore"):
  session "Charge customer credit card"
  session "Ship the product"
  session "Send confirmation email"

session "Order complete!"  # 但真的完成了吗？
```

**为什么不好**：即使付款失败，订单也可能被标记为已完成。

**修复方案**：使用适当的失败策略：

```prose
# 推荐：关键操作使用快速失败 (Fail-fast)
parallel:  # 默认策略：快速失败
  payment = session "Charge customer credit card"
  inventory = session "Reserve inventory"

# 只有在两者都成功的情况下才发货
session "Ship the product"
  context: { payment, inventory }

# 邮件发送失败可以不阻塞主流程
try:
  session "Send confirmation email"
catch:
  session "Queue email for retry"
```

#### vague-discretion（模糊的自由裁量）

使用含糊不清或不可衡量的裁量条件。

```prose
# 错误做法：什么是“足够好”？
loop until **the output is good enough**:
  session "Improve output"

# 错误做法：高度主观
if **the user will be happy**:
  session "Ship it"
```

**为什么不好**：VM 缺乏清晰的评估标准。结果是不可预测的。

**修复方案**：提供具体的、可评估的标准：

```prose
# 推荐：具体的标准
loop until **all tests pass and code coverage exceeds 80%** (max: 10):
  session "Improve test coverage"

# 推荐：可观察的条件
if **the response contains valid JSON with all required fields**:
  session "Process the response"
```

#### catch-and-swallow（捕获并静默）

捕获错误却不进行任何有意义的处理。隐藏了问题而不是解决它。

```prose
# 错误做法：静默吞掉错误
try:
  session "Critical operation"
catch:
  # 这里什么也没做 - 错误消失了
```

**为什么不好**：错误凭空消失了。没有恢复，没有日志，没有可见性。

**修复方案**：进行有意义的错误处理：

```prose
# 推荐：有意义的处理
try:
  session "Critical operation"
catch as err:
  session "Log error for investigation"
    context: err
  session "Execute fallback procedure"
  # 或者如果不可恢复则重新抛出：
  throw
```

---

## 成本反模式 (Cost Antipatterns)

#### opus-for-everything（滥用高阶模型）

对所有任务都使用最强大（也最昂贵）的模型，包括琐碎的任务。

```prose
# 错误做法：对简单的分类任务使用 Opus
agent classifier:
  model: opus
  prompt: "Categorize items as: spam, not-spam"

# 对一个二元分类任务来说太贵了
for email in emails:
  session: classifier
    prompt: "Classify: {email}"
```

**为什么不好**：Opus 的成本明显高于 Haiku。简单的任务并不需要高阶模型的深度推理。

**修复方案**：使模型与任务复杂度匹配：

```prose
# 推荐：对简单任务使用 Haiku
agent classifier:
  model: haiku
  prompt: "Categorize items as: spam, not-spam"
```

#### context-bloat（上下文膨胀）

传递会话不需要的过量上下文。

```prose
# 错误做法：传递所有内容
let full_codebase = session "Read entire codebase"
let all_docs = session "Read all documentation"
let history = session "Get full git history"

session "Fix the typo in the README"
  context: [full_codebase, all_docs, history]  # 严重的过度传递
```

**为什么不好**：过大的上下文会降低处理速度，增加成本，并可能让模型因无关信息而感到困惑。

**修复方案**：仅传递最小的相关上下文：

```prose
# 推荐：最小化的上下文
let readme = session "Read the README file"

session "Fix the typo in the README"
  context: readme
```

#### unnecessary-iteration（不必要的迭代）

本来一次会话就可以搞定，却使用了循环。

```prose
# 错误做法：可以使用一次调用却用了循环
let items = ["apple", "banana", "cherry"]
for item in items:
  session "Describe {item}"
```

**为什么不好**：可以用一个会话处理所有项，却用了三个。会话开销成倍增加。

**修复方案**：尽可能批处理：

```prose
# 推荐：批处理
let items = ["apple", "banana", "cherry"]
session "Describe each of these items: {items}"
```

#### redundant-computation（冗余计算）

多次计算相同的内容。

```prose
# 错误做法：冗余的研究
session "Research AI safety for security review"
session "Research AI safety for ethics review"
session "Research AI safety for compliance review"
```

**为什么不好**：同样的研究内容被以略微不同的方式执行了三次。

**修复方案**：计算一次，多次使用：

```prose
# 推荐：计算一次
let research = session "Comprehensive research on AI safety"

parallel:
  session "Security review"
    context: research
  session "Ethics review"
    context: research
  session "Compliance review"
    context: research
```

---

## 性能反模式 (Performance Antipatterns)

#### eager-over-computation（过早计算）

在可能只需要部分结果的情况下，提前计算所有内容。

```prose
# 错误做法：即使只需要一个分支，也并行计算所有分支
parallel:
  simple_analysis = session "Simple analysis"
    model: haiku
  detailed_analysis = session "Detailed analysis"
    model: sonnet
  deep_analysis = session "Deep analysis"
    model: opus

# 然后根据某些标准只使用其中一个
choice **appropriate depth**:
  option "Simple":
    session "Use simple"
      context: simple_analysis
  option "Detailed":
    session "Use detailed"
      context: detailed_analysis
  option "Deep":
    session "Use deep"
      context: deep_analysis
```

**为什么不好**：三个分析都运行了，即便最后只用到一个。

**修复方案**：使用惰性求值 (Lazy evaluation)：

```prose
# 推荐：只计算需要的内容
let initial = session "Initial assessment"
  model: haiku

choice **appropriate depth based on initial assessment**:
  option "Simple":
    session "Simple analysis"
      model: haiku
  option "Detailed":
    session "Detailed analysis"
      model: sonnet
  option "Deep":
    session "Deep analysis"
      model: opus
```

#### over-parallelization（过度并行化）

并行化过于激进，导致协调开销占主导地位或资源耗尽。

```prose
# 错误做法：100 个并行会话
parallel for item in large_collection:  # 假设有 100 项
  session "Process {item}"
```

**为什么不好**：可能会压垮系统。协调开销可能超过并行的收益。

**修复方案**：批处理或限制并发数：

```prose
# 推荐：分批处理
for batch in batches(large_collection, 10):
  parallel for item in batch:
    session "Process {item}"
```

#### premature-parallelization（过早并行化）

对微小的任务进行并行化，而串行执行会更简单且足够快。

```prose
# 错误做法：对简单的计算任务过度并行
parallel:
  a = session "Add 2 + 2"
  b = session "Add 3 + 3"
  c = session "Add 4 + 4"
```

**为什么不好**：协调开销超过了任务本身的执行时间。串行执行会更简单，甚至可能更快。

**修复方案**：保持简洁：

```prose
# 推荐：对琐碎任务使用串行执行
session "Add 2+2, 3+3, and 4+4"
```

#### synchronous-fire-and-forget（等待非必要的异步操作）

等待那些你不需要其结果的操作完成。

```prose
# 错误做法：等待日志记录完成
session "Do important work"
session "Log the result"  # 实际上不需要等待这个完成
session "Continue with next important work"
```

**为什么不好**：主工作流被非关键操作阻塞。

**修复方案**：对“即发即弃 (Fire-and-forget)”操作使用适当的模式，或采用批处理日志：

```prose
# 改进做法：批处理非关键工作
session "Do important work"
session "Continue with next important work"
# ... 更多重要工作 ...

# 在最后统一记录日志，或异步记录
session "Log all operations"
```

---

## 可维护性反模式 (Maintainability Antipatterns)

#### magic-strings（魔术字符串）

在程序中到处重复硬编码的提示词 (Prompts)。

```prose
# 错误做法：多处使用相同的提示词
session "You are a helpful assistant. Analyze this code for bugs."
# ... 稍后 ...
session "You are a helpful assistant. Analyze this code for bugs."
# ... 甚至更晚 ...
session "You are a helpful assistent. Analyze this code for bugs."  # 拼写错误！
```

**为什么不好**：更新时会导致不一致。拼写错误也很难被发现。

**修复方案**：使用代理 (Agents)：

```prose
# 推荐：单一事实来源
agent code-analyst:
  model: sonnet
  prompt: "You are a helpful assistant. Analyze code for bugs."

session: code-analyst
  prompt: "Analyze the auth module"
session: code-analyst
  prompt: "Analyze the payment module"
```

#### opaque-workflow（不透明的工作流）

没有任何结构或注释来表明正在发生什么。

```prose
# 错误做法：这段代码在做什么？
let x = session "A"
let y = session "B"
  context: x
parallel:
  z = session "C"
    context: y
  w = session "D"
session "E"
  context: [z, w]
```

**为什么不好**：无法理解、调试或修改。

**修复方案**：使用有意义的名称和结构：

```prose
# 推荐：清晰的意图
# 第一阶段：调研
let research = session "Gather background information"

# 第二阶段：分析
let analysis = session "Analyze research findings"
  context: research

# 第三阶段：并行评估
parallel:
  technical_eval = session "Technical feasibility assessment"
    context: analysis
  business_eval = session "Business viability assessment"
    context: analysis

# 第四阶段：综合
session "Create final recommendation"
  context: { technical_eval, business_eval }
```

#### implicit-dependencies（隐式依赖）

依赖对话历史而不是显式的上下文。

```prose
# 错误做法：隐式状态
session "Set the project name to Acme"
session "Set the deadline to Friday"
session "Now create a project plan"  # 寄希望于之前的历史被记住
```

**为什么不好**：依赖于 VM 的具体实现细节。在重构时非常脆弱。

**修复方案**：使用显式上下文：

```prose
# 推荐：显式状态
let config = session "Define project: name=Acme, deadline=Friday"

session "Create a project plan"
  context: config
```

#### mixed-concerns-agent（混合关注点代理）

代理的提示词涵盖了太多责任。

```prose
# 错误做法：万事通
agent super-agent:
  model: opus
  prompt: """
    你是以下领域的专家：
    - 安全分析
    - 性能优化
    - 代码审查
    - 文档编写
    - 测试
    - DevOps
    - 项目管理
    - 客户沟通
    被要求时，请执行上述任何任务。
  """
```

**为什么不好**：缺乏专注意味着整体结果平庸。也无法针对特定任务优化模型选择。

**修复方案**：使用专门的代理：

```prose
# 推荐：专注于专长
agent security-expert:
  model: sonnet
  prompt: "你是一名安全分析师。请只关注安全问题。"

agent performance-expert:
  model: sonnet
  prompt: "你是一名性能工程师。请只关注优化问题。"

agent technical-writer:
  model: haiku
  prompt: "你负责编写清晰的技术文档。"
```

---

## 逻辑反模式 (Logic Antipatterns)

#### infinite-refinement（无限优化）

永远无法满足退出条件的循环。

```prose
# 错误做法：完美是不可能的
loop until **the code has zero bugs**:
  session "Find and fix bugs"
```

**为什么不好**：“零 Bug”是无法实现的。循环将一直运行到最大次数（如果指定了）或永远运行。

**修复方案**：使用可以达成的条件：

```prose
# 推荐：可达成的条件
loop until **all known bugs are fixed** (max: 20):
  session "Find and fix the next bug"

# 或者：收益递减原则
loop until **no significant bugs found in last iteration** (max: 10):
  session "Search for bugs"
```

#### assertion-as-action（断言作为动作）

将条件用作动作——只检查而不对结果采取行动。

```prose
# 错误做法：检查了但不使用结果
session "Check if the system is healthy"
session "Deploy to production"  # 无论健康与否都会部署！
```

**为什么不好**：健康检查的结果没有被利用。部署会无条件发生。

**修复方案**：使用条件执行：

```prose
# 推荐：根据检查结果采取行动
let health = session "Check if the system is healthy"

if **system is healthy**:
  session "Deploy to production"
else:
  session "Alert on-call and skip deployment"
    context: health
```

#### false-parallelism（伪并行）

将具有顺序依赖关系的操作放在 `parallel` 块中。

```prose
# 错误做法：这些任务并不是独立的！
parallel:
  data = session "Fetch data"
  processed = session "Process the data"  # 需要 data！
    context: data
  stored = session "Store processed data"  # 需要 processed！
    context: processed
```

**为什么不好**：尽管放在 `parallel` 中，但由于依赖关系，它们必须串行运行。

**修复方案**：诚实对待依赖关系：

```prose
# 推荐：在需要的地方串行执行
let data = session "Fetch data"
let processed = session "Process the data"
  context: data
session "Store processed data"
  context: processed
```

#### exception-as-flow-control（异常用作流程控制）

对于预期内的条件使用 `try/catch`，而不是用于真正的异常错误。

```prose
# 错误做法：用异常处理常规逻辑
try:
  session "Find the optional config file"
catch:
  session "Use default configuration"
```

**为什么不好**：缺少配置文件是预期内的情况，而不是异常。这会掩盖真正的错误。

**修复方案**：对于预期内的情况使用条件语句：

```prose
# 推荐：预期情况使用条件判断
let config_exists = session "Check if config file exists"

if **config file exists**:
  session "Load configuration from file"
else:
  session "Use default configuration"
```

#### excessive-user-checkpoints（过多的用户检查点）

就那些答案显而易见或可预测的决策不断提示用户。

```prose
# 反模式：询问显而易见的问题
input "Blocking error detected. Investigate?"  # 肯定是“是”
input "Diagnosis complete. Proceed to triage?"  # 肯定是“是”
input "Tests pass. Deploy?"  # 绝大多数情况下是“是”
```

**为什么不好**：每个检查点都需要等待用户输入的往返时间。如果 90% 的情况下答案都是可预测的，那么你只是在增加无谓的延迟。

**修复方案**：对显而易见的情况自动继续，仅在确实存在歧义时提示：

```prose
# 推荐：自动继续，并为边缘情况设置安全机制
if observation.blocking_error:
  # 自动调查（不需要问 - 遇到错误肯定要调查）
  let diagnosis = do investigate(...)

  # 仅在确实模糊不清时询问
  if diagnosis.confidence == "low":
    input "Low confidence diagnosis. Proceed anyway?"

  # 如果测试通过则自动部署（但记录日志以便审计）
  if fix.tests_pass:
    do deploy(...)
```

**何时适合设置检查点**：不可逆的操作（生产环境关键系统的部署）、昂贵的操作（运行时间极长的任务），或者用户偏好不可预测的真实决策点。

#### fixed-observation-window（固定观察窗口）

信号已经提前到达，却仍在等待预定的时长。

```prose
# 反模式：不管发现什么都等待固定窗口
loop 30 times (wait: 2s each):  # 总是耗时 60 秒
  resume: observer
    prompt: "Keep watching the stream"
# 即使在第 1 次迭代就检测到致命错误，它也会运行完全部 30 次迭代
```

**为什么不好**：在答案已经揭晓时浪费时间。如果观察员在第 5 秒就发现了致命错误，为什么要再等 55 秒？

**修复方案**：使用信号驱动的退出条件：

```prose
# 推荐：根据重要信号退出
loop until **blocking error OR completion** (max: 30):
  resume: observer
    prompt: "Watch the stream. Signal IMMEDIATELY on blocking errors."
# 一旦发生重大情况立即退出
```

或者如果你的运行时支持，使用 `early_exit`：

```prose
# 推荐：显式的提前退出
let observation = session: observer
  prompt: "Monitor for errors. Signal immediately if found."
  timeout: 120s
  early_exit: **blocking_error detected**
```

---

## 安全反模式 (Security Antipatterns)

#### unvalidated-input（未经验证的输入）

直接将外部输入传递给会话，而不进行验证。

```prose
# 错误做法：直接注入
let user_input = external_source

session "Execute this command: {user_input}"
```

**为什么不好**：用户可能会注入恶意的提示词或命令。

**修复方案**：先验证并清理：

```prose
# 推荐：先验证
let user_input = external_source
let validated = session "Validate this input is a safe search query"
  context: user_input

if **input is valid and safe**:
  session "Search for: {validated}"
else:
  throw "Invalid input rejected"
```

#### overprivileged-agents（过度特权的代理）

代理拥有的权限超出了其实际需求。

```prose
# 错误做法：简单的任务拥有全量访问权限
agent file-reader:
  permissions:
    read: ["**/*"]
    write: ["**/*"]
    bash: allow
    network: allow

session: file-reader
  prompt: "Read the README.md file"
```

**为什么不好**：任务只需要读取一个文件，却拥有完整的系统访问权限。

**修复方案**：最小特权原则：

```prose
# 推荐：最小权限
agent file-reader:
  permissions:
    read: ["README.md"]
    write: []
    bash: deny
    network: deny
```

---

## 总结

反模式通常源于：

1. **懒惰**：使用复制粘贴而不是抽象，使用隐式而不是显式。
2. **过度工程**：并行化一切，所有任务都用 Opus。
3. **工程不足**：没有错误处理，无界循环，模糊的条件。
4. **思考不清晰**：万能会话，关注点混合，面条式上下文。

在审查 OpenProse 程序时，请自问：

- 独立任务可以并行化吗？
- 循环有边界吗？
- 错误被处理了吗？
- 上下文是否最小化且显式？
- 模型是否与任务复杂度匹配？
- 代理是否专注且可重用？
- 陌生人能看懂这段代码吗？

及早修复反模式。随着时间的推移，它们会演变成难以维护的系统。
