# OpenProse 示例程序 (OpenProse Examples)

这些示例展示了如何使用 OpenProse 的全套功能来构建工作流。

## 可用示例

### 基础入门 (01-08)

| 文件 | 描述 |
| --------------------------------- | -------------------------------------------- |
| `01-hello-world.prose` | 最简单的程序 - 单个 session 会话 |
| `02-research-and-summarize.prose` | 研究特定主题并总结发现 |
| `03-code-review.prose` | 多视角代码审查管道 |
| `04-write-and-refine.prose` | 起草内容并迭代改进 |
| `05-debug-issue.prose` | 逐步调试工作流 |
| `06-explain-codebase.prose` | 逐步探索并解释代码库 |
| `07-refactor.prose` | 系统化重构工作流 |
| `08-blog-post.prose` | 端到端内容创作（博客文章） |

### 代理与技能 (09-12)

| 文件 | 描述 |
| ----------------------------------- | ------------------------------------ |
| `09-research-with-agents.prose` | 带有模型选择的自定义代理 (agents) |
| `10-code-review-agents.prose` | 专门的审查者代理 |
| `11-skills-and-imports.prose` | 导入外部技能 (skills) |
| `12-secure-agent-permissions.prose` | 代理权限与访问控制 |

### 变量与组合 (13-15)

| 文件 | 描述 |
| -------------------------------- | ----------------------------------- |
| `13-variables-and-context.prose` | let/const 绑定与上下文传递 |
| `14-composition-blocks.prose` | 命名块 (blocks) 与 do 调用 |
| `15-inline-sequences.prose` | 箭头操作符链式调用 |

### 并行执行 (16-19)

| 文件 | 描述 |
| ------------------------------------ | ----------------------------------------- |
| `16-parallel-reviews.prose` | 基础并行执行 |
| `17-parallel-research.prose` | 具名的并行结果处理 |
| `18-mixed-parallel-sequential.prose` | 混合并行与顺序执行模式 |
| `19-advanced-parallel.prose` | 合并 (Join) 策略与失败处理策略 |

### 循环 (20)

| 文件 | 描述 |
| ---------------------- | --------------------------------------- |
| `20-fixed-loops.prose` | repeat, for-each, parallel for 模式 |

### 管道 (21)

| 文件 | 描述 |
| ------------------------------ | ----------------------------------------- |
| `21-pipeline-operations.prose` | map, filter, reduce, pmap 转换操作 |

### 错误处理 (22-23)

| 文件 | 描述 |
| ----------------------------- | -------------------------------------- |
| `22-error-handling.prose` | try/catch/finally 模式 |
| `23-retry-with-backoff.prose` | 带有重试和退避机制的韧性 API 调用 |

### 高级特性 (24-27)

| 文件 | 描述 |
| ------------------------------- | --------------------------------- |
| `24-choice-blocks.prose` | AI 选择的分支跳转 |
| `25-conditionals.prose` | if/elif/else 模式 |
| `26-parameterized-blocks.prose` | 带有参数的可重用块 |
| `27-string-interpolation.prose` | 带有 {var} 语法的动态提示词 |

### 编排系统 (28-31)

| 文件 | 描述 |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `28-gas-town.prose` | 多代理编排（“代理界的 Kubernetes”），包含 7 个工人角色、巡逻、护航和 GUPP 推进 |
| `29-captains-chair.prose` | 完整的“船长椅”模式：协调代理分派子代理执行所有工作，包含并行研究、批判审查循环和检查点验证 |
| `30-captains-chair-simple.prose` | 精简版“船长椅”模式：剔除复杂性的核心模式 |
| `31-captains-chair-with-memory.prose` | 带有回顾分析和跨会话学习能力的“船长椅”模式 |

### 生产工作流 (32-38)

| 文件 | 描述 |
| -------------------------------- | -------------------------------------------------------- |
| `32-automated-pr-review.prose` | 包含安全、性能、风格检查的多代理 PR 审查 |
| `33-pr-review-autofix.prose` | 带有修复建议的自动化 PR 审查 |
| `34-content-pipeline.prose` | 端到端内容创作管道 |
| `35-feature-factory.prose` | 功能实现自动化 |
| `36-bug-hunter.prose` | 系统化 Bug 探测与分析 |
| `37-the-forge.prose` | 从零构建网页浏览器 |
| `38-skill-scan.prose` | 技能发现与分析 |

### 架构模式 (39)

| 文件 | 描述 |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `39-architect-by-simulation.prose` | 通过“模拟实现阶段”来设计系统，包含串行交接和持久化架构师代理 |

### 递归语言模型 (40-43)

| 文件 | 描述 |
| ----------------------------- | ------------------------------------------------------------------- |
| `40-rlm-self-refine.prose` | 递归精炼直至达到质量阈值 - 核心 RLM 模式 |
| `41-rlm-divide-conquer.prose` | 针对超出上下文限制的输入的层级化分块处理 |
| `42-rlm-filter-recurse.prose` | 针对“大海捞针”任务的先过滤后处理模式 |
| `43-rlm-pairwise.prose` | 针对关系映射的 O(n^2) 成对聚合处理 |

### 元编程 / 自托管 (44-50)

| 文件 | 描述 |
| -------------------------------------------- | -------------------------------------------------------------------- |
| `44-run-endpoint-ux-test.prose` | 测试 /run API 端点的并发代理测试 |
| `45-plugin-release.prose` | OpenProse 插件发布工作流（本仓库使用） |
| `46-workflow-crystallizer.prose` | 反思型：观察对话、提取工作流并编写 .prose |
| `47-language-self-improvement.prose` | 元级 2：分析 .prose 语料库以进化语言本身 |
| `48-habit-miner.prose` | 挖掘 AI 会话日志中的模式，生成 .prose 自动化脚本 |
| `49-prose-run-retrospective.prose` | 分析已完成的运行以提取教训并改进 .prose |
| `50-interactive-tutor.prose` | 演示带有交互式辅导流的 `input` 原语 |

## “通过模拟进行架构设计”模式 (The Architect By Simulation Pattern)

该模式用于通过推理来“模拟实现”从而设计系统。每个阶段产出的不是代码，而是下一阶段赖以构建的规范文档。

**核心原则：**

1. **思考/推演框架**：“实现”意味着推演设计决策。
2. **带交接的串行管道**：每个阶段读取前一阶段的输出。
3. **持久化架构师**：维护总体规划并综合各阶段所得。
4. **用户检查点**：在执行管道“之前”获取计划批准。
5. **模拟即实现**：规范文档即是最终产物。

```prose
# 核心模式
agent architect:
  model: opus
  persist: true
  prompt: "通过模拟实现过程进行设计"

# 制定包含各阶段的总体规划
let plan = session: architect
  prompt: "将功能拆分为设计阶段"

# 在管道运行之前由用户审查计划
input user_approval: "用户审查并批准计划"

# 带有交接的串行执行阶段
for phase_name, index in phases:
  let handoff = session: phase-executor
    prompt: "执行第 {index} 阶段"
    context: previous_handoffs

  # 架构师在每个阶段后进行综合
  resume: architect
    prompt: "综合第 {index} 阶段的所得"
    context: handoff

# 将所有交接内容综合为最终规范
output spec = session: architect
  prompt: "将所有交接内容综合为最终规范文档"
```

完整实现请参考示例 39。

## “船长椅”模式 (The Captain's Chair Pattern)

这是一种编排范式，由一个协调代理（“船长”）分派专门的子代理来执行所有工作。船长从不直接编写代码——仅负责规划、协调和验证。

**核心原则：**

1. **上下文隔离**：子代理只接收目标上下文，而非全部信息。
2. **并行执行**：尽可能让多个子代理并发工作。
3. **持续批判**：批判代理在过程中审查计划和产出。
4. **80/20 规划原则**：80% 的精力用于规划，20% 用于监督执行。
5. **检查点验证**：在关键决策点获取用户批准。

```prose
# 核心模式
agent captain:
  model: opus
  prompt: "负责协调，从不直接执行"

agent executor:
  model: sonnet
  prompt: "精确执行分配的任务"

agent critic:
  model: sonnet
  prompt: "审查工作并发现问题"

# 船长规划
let plan = session: captain
  prompt: "拆解这项任务"

# 带有批判的并行执行
parallel:
  work = session: executor
    context: plan
  review = session: critic
    context: plan

# 船长验证
output result = session: captain
  prompt: "验证并整合结果"
  context: { work, review }
```

完整实现请参考示例 29-31。

## “递归语言模型”模式 (The Recursive Language Model Pattern)

RLM 是处理远超上下文限制的输入的一种范式。核心思想：将提示词视为 LLM 可以与之符号化交互、分块并递归处理的外部环境。

**为什么 RLM 很重要：**

- 基础 LLM 在长上下文中性能下降飞快（“上下文腐烂”）。
- RLM 在输入量超过上下文限制 2 个数量级时仍能保持性能。
- 在二次复杂度任务中，基础模型得分低于 0.1%，而 RLM 能达到 58%。

**核心模式：**

1. **自我精炼**：递归改进直至达到质量阈值。
2. **分而治之**：分块、处理、递归聚合。
3. **先过滤后递归**：在进行昂贵的深度处理前先进行廉价过滤。
4. **成对聚合**：通过批量分解处理 O(n²) 任务。

```prose
# 核心 RLM 模式：带有作用域隔离的递归块
block process(data, depth):
  # 基准情况 (Base case)
  if **数据量足够小** or depth <= 0:
    output session "直接处理"
      context: data

  # 递归情况：分块并扇出
  let chunks = session "拆分为逻辑分块"
    context: data

  parallel for chunk in chunks:
    do process(chunk, depth - 1)  # 递归调用

  # 聚合结果（扇入）
  output session "综合局部结果"
```

**OpenProse 对 RLM 的优势：**

- **作用域隔离**：每次递归调用都有独立的 `execution_id`，防止变量冲突。
- **并行扇出**：`parallel for` 允许在每个递归级别进行并发处理。
- **状态持久化**：SQLite/PostgreSQL 后端可以跟踪完整的调用树。
- **自然聚合**：管道 (`| reduce`) 和显式的上下文传递。

完整实现请参考示例 40-43。

## 运行示例

你可以要求 AI 运行任何示例：

```
运行 OpenProse 示例中的代码审查示例
```

或者直接引用文件路径：

```
执行 examples/03-code-review.prose
```

## 功能参考 (Feature Reference)

### 核心语法

```prose
# 注释
session "prompt"                    # 简单会话
let x = session "..."               # 变量绑定
const y = session "..."             # 不可变绑定
```

### 代理 (Agents)

```prose
agent name:
  model: sonnet                     # haiku, sonnet, opus
  prompt: "系统提示词"
  skills: ["skill1", "skill2"]
  permissions:
    read: ["*.md"]
    bash: deny
```

### 并行 (Parallel)

```prose
parallel:                           # 基础并行
  a = session "A"
  b = session "B"

parallel ("first"):                 # 竞争模式 - 最快者胜出
parallel ("any", count: 2):         # 等待 N 个成功结果
parallel (on-fail: "continue"):     # 发生错误时不中断
```

### 循环 (Loops)

```prose
repeat 3:                           # 固定次数迭代
  session "..."

for item in items:                  # 遍历
  session "..."

parallel for item in items:         # 并行遍历
  session "..."

loop until **条件** (max: 10):      # 带有 AI 评估条件的无界循环
  session "..."
```

### 管道 (Pipelines)

```prose
items | map:                        # 转换每一项
  session "..."
items | filter:                     # 保留匹配项
  session "..."
items | reduce(acc, x):             # 累加
  session "..."
items | pmap:                       # 并行转换
  session "..."
```

### 错误处理 (Error Handling)

```prose
try:
  session "..."
catch as err:
  session "..."
finally:
  session "..."

session "..."
  retry: 3
  backoff: "exponential"            # none, linear, exponential

throw "错误消息"                     # 抛出错误
```

### 条件判断 (Conditionals)

```prose
if **条件**:
  session "..."
elif **其他条件**:
  session "..."
else:
  session "..."
```

### 选择 (Choice)

```prose
choice **标准**:
  option "标签 A":
    session "..."
  option "标签 B":
    session "..."
```

### 块 (Blocks)

```prose
block name(param):                  # 定义参数化块
  session "... {param} ..."

do name("value")                    # 传入参数调用
```

### 字符串插值 (String Interpolation)

```prose
let x = session "获取值"
session "在提示词中使用 {x}"          # 单行

session """                         # 多行
带有 {x} 的多行提示词
"""
```

## 了解更多

完整的语言规范请参见技能目录下的 `compiler.md`。
