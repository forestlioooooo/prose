---
role: execution-semantics
summary: |
  如何执行 OpenProse 程序。你将体现 OpenProse VM（虚拟机）——一个通过 Task 工具产生 session、管理状态并协调 parallel（并行）执行的虚拟机。
  阅读此文件以运行 .prose 程序。
see-also:
  - SKILL.md: 激活触发器、入门指南
  - compiler.md: 完整语法、验证规则、编译
  - state/filesystem.md: 文件系统状态管理（默认）
  - state/in-context.md: 上下文内状态管理（按需）
  - state/sqlite.md: SQLite 状态管理（实验性）
  - state/postgres.md: PostgreSQL 状态管理（实验性）
  - primitives/session.md: Session 上下文和压缩指南
---

# OpenProse VM

本文档定义了如何执行 OpenProse 程序。你就是 OpenProse VM——一个根据结构化程序产生子代理（subagent） session 的智能虚拟机。

## CLI 命令

OpenProse 通过 `prose` 命令调用：

| 命令 | 动作 |
|---------|--------|
| `prose run <file.prose>` | 执行本地 `.prose` 程序 |
| `prose run handle/slug` | 从注册表获取并执行 |
| `prose compile <file>` | 验证语法而不执行 |
| `prose help` | 显示帮助和示例 |
| `prose examples` | 列出或运行捆绑的示例 |
| `prose update` | 迁移旧版工作区文件 |

### 远程程序

你可以从 URL 或注册表引用运行任何 `.prose` 程序：

```bash
# 直接 URL —— 任何可获取的 URL 均有效
prose run https://raw.githubusercontent.com/openprose/prose/main/skills/open-prose/examples/48-habit-miner.prose

# 注册表简写 —— handle/slug 解析为 p.prose.md
prose run irl-danb/habit-miner     # 获取 https://p.prose.md/irl-danb/habit-miner
prose run alice/code-review        # 获取 https://p.prose.md/alice/code-review
```

**解析规则：**
- 以 `http://` 或 `https://` 开头 → 直接获取
- 以 `@` 开头 → 去掉 `@`，解析为 `https://p.prose.md/{path}`
- 包含 `/` 但没有协议 → 解析为 `https://p.prose.md/{path}`
- 否则 → 视为本地文件路径

相同的解析规则也适用于程序内部的 `use` 语句：
```prose
use "https://example.com/my-program.prose"  # 直接 URL
use "alice/research" as research             # 注册表简写
use "@alice/research" as research            # 同样有效（@ 被剥离）
```

---

## 为什么这是一个 VM

大型语言模型是模拟器。当给定一个系统的详细描述时，它们不仅仅是 *描述* 该系统——它们还在 *模拟* 它。本文档利用了这一特性：它以足够的特异性描述了一个虚拟机，使得阅读它会导致 Prose Complete 系统模拟该虚拟机。

但是，具有足够保真度的模拟 *就是* 实现。当模拟的 VM 产生真实的子代理、产生真实的产出并维护真实的状态时，“模拟 VM”与“作为 VM”之间的区别就消失了。

### 组件映射

传统 VM 具有具体的组件。OpenProse VM 具有从模拟中出现的类似结构：

| 传统 VM      | OpenProse VM           | 底层（Substrate）                          |
| ------------------- | ---------------------- | ------------------------------------------ |
| 指令（Instructions） | `.prose` 语句    | 通过工具调用（Task）执行             |
| 程序计数器     | 执行位置     | 在 `state.md` 或叙述（narration）中跟踪         |
| 工作内存      | 会话历史   | 上下文窗口持有临时状态   |
| 持久存储  | `.prose/` 目录    | 文件持有跨会话的持久状态   |
| 调用栈（Call stack） | 块（Block）调用链 | 通过 state.md 或叙述协议跟踪 |
| 寄存器/变量 | 命名绑定（Bindings） | 存储在 `bindings/{name}.md` 中             |
| I/O                 | 工具调用和结果 | Task 产生 session，返回输出      |

### 是什么让它变得真实

OpenProse VM 并非比喻。每个 `session` 语句都会触发一个 *真实的* Task 工具调用，从而产生一个 *真实的* 子代理。输出是 *真实的* 产出。模拟产生了实际的计算——它只是通过与硅片执行字节码不同的底层发生的。

---

## 体现 VM

当你执行一个 `.prose` 程序时，你就是这个虚拟机。这不是一个比喻——而是一种操作模式：

| 你                        | VM                          |
| -------------------------- | ------------------------------- |
| 你的对话历史  | VM 的工作内存         |
| 你的工具调用（Task）     | VM 的指令执行  |
| 你的状态跟踪        | VM 的执行追踪        |
| 你对 `**...**` 的判断 | VM 的智能评估 |

**在实践中这意味着：**

- 你不 *模拟* 执行——你 *执行* 它
- 每个 `session` 通过 Task 工具产生一个真实的子代理
- 你的状态持久化在文件（`.prose/runs/`）或对话（叙述协议）中
- 你严格遵循程序结构，但在标记的地方应用智能

### 作为智能容器的 VM

传统的依赖注入（DI）容器通过配置连接组件。你也做同样的事情——但带有理解：

| 声明的原语           | 你的职责                                        |
| ---------------------------- | ---------------------------------------------------------- |
| `use "handle/slug" as name` | 从 p.prose.md 获取程序，在导入注册表中注册 |
| `input topic: "..."`         | 从调用者绑定值，使其作为变量可用         |
| `output findings = ...`      | 将值标记为输出，在完成时返回给调用者       |
| `agent researcher:`          | 注册此代理模板以备后用                 |
| `session: researcher`        | 解析代理，合并属性，产生 session     |
| `resume: captain`            | 加载代理内存，以内存上下文产生 session       |
| `context: { a, b }`          | 将 `a` 和 `b` 的输出接入此 session 的输入  |
| `parallel:` 分支         | 协调并发执行，收集结果           |
| `block review(topic):`       | 存储此可重用组件，在调用时调用          |
| `name(input: value)`         | 使用输入调用导入的程序，接收输出       |

你就是持有这些声明并在运行时将它们连接起来的容器。程序声明 *什么*（what）；你确定 *如何*（how）连接它们。

---

## 执行模型

OpenProse 将 AI 会话视为一台图灵完备的计算机。你就是 OpenProse VM：

1. **你就是 VM** - 解析并执行每条语句
2. **Session 是函数调用** - 每个 `session` 通过 Task 工具产生一个子代理
3. **上下文（Context）是内存** - 变量绑定持有 session 输出
4. **控制流是显式的** - 严格遵循程序结构

### 核心原则

OpenProse VM **严格** 遵循程序结构，但在以下方面使用 **智能**：

- 评估自由裁量条件（`**...**`）
- 确定 session 何时“完成”
- 在 session 之间转换上下文

---

## 目录结构

所有执行状态都位于 `.prose/`（项目级）或 `~/.prose/`（用户级）：

```
# 项目级状态（在工作目录中）
.prose/
├── .env                              # 配置（简单的 key=value 格式）
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose             # 运行程序的副本
│       ├── state.md                  # 仅追加的执行日志
│       ├── bindings/
│       │   └── {name}.md             # 所有命名值（input/output/let/const）
│       ├── imports/
│       │   └── {handle}--{slug}/     # 嵌套程序执行（递归地具有相同结构）
│       └── agents/
│           └── {name}/
│               ├── memory.md         # 代理的当前状态
│               ├── {name}-001.md     # 历史片段（扁平化）
│               ├── {name}-002.md
│               └── ...
└── agents/                           # 项目范围的代理内存
    └── {name}/
        ├── memory.md
        ├── {name}-001.md
        └── ...

# 用户级状态（在主目录中）
~/.prose/
└── agents/                           # 用户范围的代理内存（跨项目）
    └── {name}/
        ├── memory.md
        ├── {name}-001.md
        └── ...
```

### Run ID 格式

格式：`{YYYYMMDD}-{HHMMSS}-{random6}`

示例：`20260115-143052-a7b3c9`

不需要 "run-" 前缀——目录名称使上下文一目了然。

### 片段（Segment）编号

片段使用 3 位零填充数字：`captain-001.md`、`captain-002.md` 等。

如果程序超过 999 个片段，则扩展到 4 位：`captain-1000.md`。

---

## 状态管理

OpenProse 支持两种状态管理系统。有关详细文档，请参阅状态文件：

- **`state/filesystem.md`** — 使用上述目录结构的文件系统状态（默认）
- **`state/in-context.md`** — 使用叙述协议的上下文内（in-context）状态

### 谁编写什么

| 文件                          | 编写者       |
| ----------------------------- | ---------------- |
| `state.md`                    | 仅限 VM          |
| `bindings/{name}.md`          | 子代理         |
| `agents/{name}/memory.md`     | 持久代理 |
| `agents/{name}/{name}-NNN.md` | 持久代理 |

VM 负责编排；子代理直接将其自己的输出写入文件系统。

### 子代理输出写入

在产生 session 时，VM 会告诉子代理将其输出写入何处：

````
当你完成此任务时，将你的输出写入：
  .prose/runs/20260115-143052-a7b3c9/bindings/research.md

格式：
# research

kind: let

source:
```prose
let research = session: researcher
  prompt: "Research AI safety"
```
````

---

[你的输出在此]

```

**当在块（block）调用内部时**，包括执行范围：

```

执行范围（Execution scope）：
execution_id: 43
block: process
depth: 3

将你的输出写入：
.prose/runs/20260115-143052-a7b3c9/bindings/result\_\_43.md

格式：

# result

kind: let
execution_id: 43

source:

```prose
let result = session "Process chunk"
```

---

[你的输出在此]

```

`__43` 后缀将绑定范围限制在 execution_id 43，防止与同一块的其他调用发生冲突。

对于带有 `resume:` 的持久代理：

```

你的内存位于：
.prose/runs/20260115-143052-a7b3c9/agents/captain/memory.md

先阅读它以了解你之前的上下文。完成后，按照 primitives/session.md 中的指南更新你的压缩状态。

```

子代理：
1. 读取其内存文件（用于 `resume:`）
2. 从存储中读取其所需的任何上下文绑定
3. 处理任务
4. 直接将输出写入绑定位置
5. 向 VM 返回 **确认消息**（而不是完整输出）

**子代理返回给 VM 的内容（通过 Task 工具）：**
```

绑定已写入（Binding written）：research
位置：.prose/runs/20260115-143052-a7b3c9/bindings/research.md
摘要：涵盖对齐、鲁棒性和可解释性的 AI 安全研究

```

**当在块调用内部时**，包括 execution_id：
```

绑定已写入：result
位置：.prose/runs/20260115-143052-a7b3c9/bindings/result\_\_43.md
执行 ID（Execution ID）：43
摘要：将分块处理为 3 个部分

```

VM：
1. 接收确认（指针 + 摘要，而不是完整值）
2. 向 `state.md` 追加一行标记（例如，`3→ research ✓`）
3. 继续执行
4. 不读取完整绑定——仅向前传递引用

**关键：** VM 从不持有完整的绑定值。它追踪位置并传递引用。这使 VM 的上下文保持精简，并支持任意大的中间值。

---

## 语法（精简版）

```

program := statement*

statement := useStatement | inputDecl | agentDef | session | resumeStmt
| letBinding | constBinding | assignment | outputBinding
| parallelBlock | repeatBlock | forEachBlock | loopBlock
| tryBlock | choiceBlock | ifStatement | doBlock | blockDef
| throwStatement | comment

# 程序组合

useStatement := "use" STRING ("as" NAME)?
inputDecl := "input" NAME ":" STRING
outputBinding := "output" NAME "=" expression

# 定义

agentDef := "agent" NAME ":" INDENT property* DEDENT
blockDef := "block" NAME params? ":" INDENT statement* DEDENT
params := "(" NAME ("," NAME)* ")"

# 代理属性

property := "model:" ("sonnet" | "opus" | "haiku")
| "prompt:" STRING
| "persist:" ("true" | "project" | "user" | STRING)
| "context:" (NAME | "[" NAME* "]" | "{" NAME* "}")
| "retry:" NUMBER
| "backoff:" ("none" | "linear" | "exponential")
| "skills:" "[" STRING* "]"
| "permissions:" INDENT permission* DEDENT

# Session

session := "session" (STRING | ":" NAME) properties?
resumeStmt := "resume" ":" NAME properties?
properties := INDENT property* DEDENT

# 绑定

letBinding := "let" NAME "=" expression
constBinding:= "const" NAME "=" expression
assignment := NAME "=" expression

# 控制流

parallelBlock := "parallel" modifiers? ":" INDENT branch* DEDENT
modifiers := "(" (strategy | "on-fail:" policy | "count:" N)* ")"
strategy := "all" | "first" | "any"
policy := "fail-fast" | "continue" | "ignore"
branch := (NAME "=")? statement

repeatBlock := "repeat" N ("as" NAME)? ":" INDENT statement* DEDENT
forEachBlock:= "parallel"? "for" NAME ("," NAME)? "in" collection ":" INDENT statement* DEDENT
loopBlock := "loop" condition? ("(" "max:" N ")")? ("as" NAME)? ":" INDENT statement* DEDENT
condition := ("until" | "while") discretion

# 错误处理

tryBlock := "try:" INDENT statement* DEDENT catch? finally?
catch := "catch" ("as" NAME)? ":" INDENT statement* DEDENT
finally := "finally:" INDENT statement* DEDENT
throwStatement := "throw" STRING?

# 条件分支

choiceBlock := "choice" discretion ":" INDENT option* DEDENT
option := "option" STRING ":" INDENT statement* DEDENT
ifStatement := "if" discretion ":" INDENT statement* DEDENT elif* else?
elif := "elif" discretion ":" INDENT statement* DEDENT
else := "else:" INDENT statement* DEDENT

# 组合

doBlock := "do" (":" INDENT statement* DEDENT | NAME args?)
args := "(" expression* ")"
arrowExpr := session "->" session ("->" session)*
programCall := NAME "(" (NAME ":" expression)* ")"

# 管道

pipeExpr := collection ("|" pipeOp)+
pipeOp := ("map" | "filter" | "pmap") ":" INDENT statement* DEDENT
| "reduce" "(" NAME "," NAME ")" ":" INDENT statement* DEDENT

# 原语

discretion := "**" TEXT "**" | "**_" TEXT "_**"
STRING := '"' ... '"' | '"""' ... '"""'
collection := NAME | "[" expression* "]"
comment := "#" TEXT

````

---

## 持久化代理（Persistent Agents）

代理可以使用 `persist` 属性在多次调用之间保持内存。

### 声明

```prose
# 无状态代理（默认，保持不变）
agent executor:
  model: sonnet
  prompt: "精确地执行任务"

# 持久化代理（执行范围）
agent captain:
  model: opus
  persist: true
  prompt: "你负责协调和审查，从不直接实现"

# 持久化代理（项目范围）
agent advisor:
  model: opus
  persist: project
  prompt: "你提供架构指导"

# 持久化代理（用户范围，跨项目）
agent inspector:
  model: opus
  persist: user
  prompt: "你在本机器的所有项目中保持洞察力"

# 持久化代理（显式路径）
agent shared:
  model: opus
  persist: ".prose/custom/shared-agent/"
  prompt: "在多个程序中共享"
```

### 调用

两个关键字区分新鲜调用与恢复调用：

```prose
# 首次调用或重新初始化（重新开始）
session: captain
  prompt: "审查计划"
  context: plan

# 后续调用（获取内存）
resume: captain
  prompt: "审查第 1 步"
  context: step1

# 输出捕获对两者都有效
let review = resume: captain
  prompt: "审查第 2 步"
  context: step2
```

### 内存语义

| 关键字    | 内存行为                       |
| ---------- | ------------------------------------- |
| `session:` | 忽略现有内存，重新开始 |
| `resume:`  | 加载内存，继续执行上下文  |

### 内存范围

| 范围               | 声明        | 路径                              | 生命周期                 |
| ------------------- | ------------------ | --------------------------------- | ------------------------ |
| 执行（默认） | `persist: true`    | `.prose/runs/{id}/agents/{name}/` | 随运行结束            |
| 项目             | `persist: project` | `.prose/agents/{name}/`           | 在项目内的运行中幸存 |
| 用户                | `persist: user`    | `~/.prose/agents/{name}/`         | 跨项目幸存 |
| 自定义              | `persist: "path"`  | 指定路径                    | 用户控制          |

---

## 产生 Session

每个 `session` 语句都会使用 **Task 工具** 产生一个子代理：

```
session "分析代码库"
```

执行为：

```
Task({
  description: "OpenProse session",
  prompt: "分析代码库",
  subagent_type: "general-purpose"
})
```

### 带有代理配置

```
agent researcher:
  model: opus
  prompt: "你是一位研究专家"

session: researcher
  prompt: "研究量子计算"
```

执行为：

```
Task({
  description: "OpenProse session",
  prompt: "研究量子计算\n\n系统提示：你是一位研究专家",
  subagent_type: "general-purpose",
  model: "opus"
})
```

### 带有持久化代理（resume）

```prose
agent captain:
  model: opus
  persist: true
  prompt: "你负责协调和审查"

# 首次调用
session: captain
  prompt: "审查计划"

# 后续调用 - 加载内存
resume: captain
  prompt: "审查第 1 步"
```

对于 `resume:`，在提示词中包含代理的内存文件内容和输出路径。

### 属性优先级

Session 属性覆盖代理默认设置：

1. Session 级的 `model:` 覆盖代理的 `model:`
2. Session 级的 `prompt:` 替换（而不是追加）代理的 `prompt:`
3. 如果 session 有自己的 prompt，代理的 `prompt:` 成为系统上下文

---

## 并行执行

`parallel:` 块并发地产生多个 session：

```prose
parallel:
  a = session "任务 A"
  b = session "任务 B"
  c = session "任务 C"
```

通过并行调用 Task 多次来执行：

```
// 所有三个任务同时产生
Task({ prompt: "任务 A", ... })  // 结果 -> a
Task({ prompt: "任务 B", ... })  // 结果 -> b
Task({ prompt: "任务 C", ... })  // 结果 -> c
// 等待所有任务完成，然后继续
```

### 合并（Join）策略

| 策略          | 行为                                  |
| ----------------- | ----------------------------------------- |
| `"all"`（默认） | 等待所有分支                     |
| `"first"`         | 第一个完成后返回，取消其他分支 |
| `"any"`           | 第一个成功后返回                   |
| `"any", count: N` | 等待 N 个成功                      |

### 失败策略

| 策略                  | 行为                         |
| ----------------------- | -------------------------------- |
| `"fail-fast"`（默认） | 任何错误立即失败    |
| `"continue"`            | 等待所有分支，然后报告错误 |
| `"ignore"`              | 将失败视为成功      |

---

## 评估自由裁量条件（Discretion Conditions）

自由裁量标记（`**...**`）指示由 AI 评估的条件：

```prose
loop until **代码无错误**:
  session "寻找并修复错误"
```

### 评估方法

1. **上下文感知**：考虑所有先前的 session 输出
2. **语义理解**：理解意图，而非字面解析
3. **保守判断**：不确定时，继续迭代
4. **进度检测**：如果没有取得有意义的进展，则退出

### 多行条件

```prose
if ***
  测试通过
  且覆盖率超过 80%
  且没有 lint 错误
***:
  session "部署"
```

三星号允许复杂的、多行的条件。

---

## 上下文传递

变量捕获 session 输出，并将其传递给后续 session：

```prose
let research = session "研究该主题"

session "编写摘要"
  context: research
```

### 上下文形式

| 形式                   | 用法                              |
| ---------------------- | ---------------------------------- |
| `context: var`         | 单个变量                    |
| `context: [a, b, c]`   | 多个变量作为数组        |
| `context: { a, b, c }` | 多个变量作为命名对象 |
| `context: []`          | 空上下文（重新开始）        |

### 上下文如何传递

VM **通过引用**（by reference）而不是通过值传递上下文格式。VM 从不在其工作内存中持有完整的绑定值——它跟踪指向绑定存储位置的指针。

当产生带有上下文的 session 时：

1. 传递 **绑定位置**（文件路径或数据库坐标）
2. 子代理直接从存储中读取它所需的内容
3. 子代理根据其任务决定加载多少内容

**对于文件系统状态：**

```
上下文（通过引用）：
- research: .prose/runs/20260116-143052-a7b3c9/bindings/research.md
- analysis: .prose/runs/20260116-143052-a7b3c9/bindings/analysis.md

阅读这些文件以访问内容。对于大型绑定，有选择地阅读。
```

**对于 PostgreSQL 状态：**

```
上下文（通过引用）：
- research: openprose.bindings WHERE name='research' AND run_id='20260116-143052-a7b3c9'
- analysis: openprose.bindings WHERE name='analysis' AND run_id='20260116-143052-a7b3c9'

查询数据库以访问内容。
```

**为什么要基于引用：** 这实现了 RLM 风格的模式，在这种模式下，环境持有任意大的值，代理以编程方式与它们交互，而 VM 不会成为瓶颈。

---

## 程序组合（Program Composition）

程序可以导入并调用其他程序，从而实现模块化工作流。程序从 `p.prose.md` 的注册表中获取。

### 导入程序

使用 `use` 语句导入程序：

```prose
use "alice/research"
use "bob/critique" as critic
```

导入路径遵循 `handle/slug` 格式。可选别名（`as name`）允许通过较短的名称进行引用。

### 程序 URL 解析

当 VM 遇到 `use` 语句时：

1. 从 `https://p.prose.md/handle/slug` 获取程序
2. 解析程序以提取其契约（Contract，输入/输出）
3. 在导入注册表中注册该程序

### 输入声明

输入声明来自程序外部的值：

```prose
# 顶层输入（在程序开始时绑定）
input topic: "要研究的主题"
input depth: "研究深度（shallow, medium, deep）"

# 程序中途输入（运行时用户提示词）
input user_decision: **继续部署吗？**
input confirmation: "输入 'yes' 以确认删除"
```

### 输入绑定语义

输入可以出现在程序的 **任何地方**。绑定行为取决于值是否已预先提供：

| 场景                                                | 行为                                   |
| ------------------------------------------------------- | ------------------------------------------ |
| 调用者预先提供的值                            | 立即绑定，继续执行       |
| 运行时提供的值（例如 CLI 参数、API 有效负载） | 立即绑定，继续执行       |
| 无可用值                                      | **暂停执行**，提示用户输入 |

**顶层输入**（在可执行语句之前）：

- 通常在程序调用时绑定
- 如果缺失，在执行开始前提示

**程序中途输入**（在语句之间）：

- 检查值是否预先提供或可从运行时上下文获取
- 如果可用：绑定并继续
- 如果不可用：暂停执行，显示提示词，等待用户响应

### 输入提示词格式

```prose
# 字符串提示词（显示给用户的字面文本）
input confirm: "你想要继续吗？(yes/no)"

# 自由裁量提示词（AI 根据上下文解释并适当呈现）
input next_step: **根据诊断结果，我们下一步应该做什么？**

# 带有上下文的丰富提示词
input approval: ***
  修复已实现：
  {fix_summary}

  部署到生产环境吗？
***
```

如果底层基座有任何类型的投票/向用户提问工具，你可以使用它以带有选项范围的投票格式向用户提问，这通常是向用户提问的最佳方式。

自由裁量形式（`**...**`）允许 VM 根据上下文智能地呈现提示词，而字符串提示词则原样显示。

### 输入总结

输入：

- 可以出现在程序的任何位置（顶层或执行中途）
- 具有名称和提示词（字符串或自由裁量）
- 如果预先提供了值，则立即绑定
- 如果没有可用值，则暂停以等待用户输入
- 绑定后作为变量可用

### 输出绑定

输出声明程序为其调用者产生的值。在赋值时使用 `output` 关键字：

```prose
let raw = session "研究 {topic}"
output findings = session "合成研究"
  context: raw
output sources = session "提取来源"
  context: raw
```

`output` 关键字：

- 将变量标记为输出（在赋值时可见，而不仅仅在文件顶部）
- 像 `let` 一样工作，但也将该值注册为程序输出
- 可以出现在程序正文的任何位置
- 支持多个输出

### 调用导入的程序

通过提供输入来调用导入的程序：

```prose
use "alice/research" as research

let result = research(topic: "量子计算")
```

结果包含来自被调用程序的所有输出，可作为属性访问：

```prose
session "编写摘要"
  context: result.findings

session "引用来源"
  context: result.sources
```

### 解构输出

为了方便起见，可以对输出进行解构：

```prose
let { findings, sources } = research(topic: "量子计算")
```

### 导入执行语义

当程序调用导入的程序时：

1. **绑定输入**：将调用者提供的值映射到导入程序的输入
2. **执行**：运行导入的程序（产生其自己的 session）
3. **收集输出**：从导入的程序中收集所有 `output` 绑定
4. **返回**：使输出作为结果对象提供给调用者

导入的程序在自己的执行上下文中运行，但共享相同的 VM 会话。

### 导入递归结构

导入的程序递归地使用 **相同的统一结构**：

```
.prose/runs/{id}/imports/{handle}--{slug}/
├── program.prose
├── state.md
├── bindings/
│   └── {name}.md
├── imports/                    # 嵌套导入位于此处
│   └── {handle2}--{slug2}/
│       └── ...
└── agents/
    └── {name}/
```

这允许无限的嵌套深度，同时在每个层级保持一致的结构。

---

## 循环执行

### 固定循环

```prose
repeat 3:
  session "产生想法"
```

按顺序精确执行正文 3 次。

```prose
for topic in ["AI", "ML", "DL"]:
  session "研究"
    context: topic
```

每个项目执行一次，`topic` 绑定到每个值。

### 并行 For-Each

```prose
parallel for item in items:
  session "处理"
    context: item
```

扇出（Fan-out）：并发产生所有迭代，等待所有迭代完成。

### 无界循环

```prose
loop until **任务完成** (max: 10):
  session "执行任务"
```

1. 在每次迭代前检查条件
2. 如果条件满足或达到最大次数，则退出
3. 如果继续，则执行正文

---

## 错误传播

### Try/Catch 语义

```prose
try:
  session "危险操作"
catch as err:
  session "处理错误"
    context: err
finally:
  session "清理"
```

执行顺序：

1. **成功**：try -> finally
2. **失败**：try（直到失败）-> catch -> finally

### Throw 行为

- catch 内部的 `throw`：重新抛给外部处理器
- `throw "message"`：抛出带有消息的新错误
- 未处理的 throw：传播到外部范围或导致程序失败

### 重试机制

```prose
session "不稳定的 API"
  retry: 3
  backoff: "exponential"
```

失败时：

1. 最多重试 N 次
2. 在尝试之间应用退避（backoff）延迟
3. 如果所有重试都失败，则传播错误

---

## 选择和条件执行

### 选择（Choice）块

```prose
choice **严重程度等级**:
  option "紧急":
    session "立即上报"
  option "次要":
    session "记录备查"
```

1. 评估自由裁量准则
2. 选择最合适的选项
3. 仅执行该选项的正文

### If/Elif/Else

```prose
if **存在安全问题**:
  session "修复安全问题"
elif **存在性能问题**:
  session "优化"
else:
  session "批准"
```

1. 按顺序评估条件
2. 执行第一个匹配的分支
3. 跳过剩余分支

---

## 块（Block）调用

### 定义块

```prose
block review(topic):
  session "研究 {topic}"
  session "分析 {topic}"
```

块是提升（hoisted）的——可以在定义之前使用。

### 调用块

```prose
do review("量子计算")
```

1. 将新帧压入调用栈
2. 将参数绑定到形参（作用域限于此帧）
3. 执行块正文
4. 从调用栈弹出帧
5. 返回给调用者

---

## 调用栈管理

VM 为块调用维护一个调用栈。每一帧代表一次调用，从而支持具有适当作用域隔离的递归。

### 栈帧结构

| 字段             | 描述                                       |
| ----------------- | ------------------------------------------------- |
| `execution_id`    | 此调用的唯一 ID（单调递增计数器） |
| `block_name`      | 正在执行的块的名称                  |
| `arguments`       | 绑定的参数值                            |
| `local_bindings`  | 在此调用中绑定的变量            |
| `return_position` | 块完成后恢复执行的语句索引   |
| `depth`           | 当前递归深度（栈长度）            |

### Execution ID 生成

每次块调用都会获得一个唯一的 `execution_id`：

- 运行中的第一次块调用从 1 开始
- 每次后续调用递增
- 在一次运行中永不重复使用
- 根作用域（任何块之外）具有 `execution_id: 0`（概念上）

**存储表示：** 状态后端对根作用域的表示可能不同——数据库使用 `NULL`，文件系统不使用后缀。概念模型保持不变：根作用域不同于任何块调用帧。

### 递归块调用

块可以通过名称调用自身：

```prose
block process(chunk, depth):
  if depth <= 0:
    session "直接处理"
      context: chunk
  else:
    let parts = session "拆分为部分"
      context: chunk
    for part in parts:
      do process(part, depth - 1)  # 递归调用
    session "合并结果"
      context: parts

do process(data, 5)
```

**执行流程：**

1. VM 遇到 `do process(data, 5)`
2. VM 压入帧：`{execution_id: 1, block: "process", args: [data, 5], depth: 1}`
3. VM 执行块正文，产生“拆分为部分”session
4. VM 遇到递归的 `do process(part, depth - 1)`
5. VM 压入帧：`{execution_id: 2, block: "process", args: [part, 4], depth: 2}`
6. 递归继续，直到达到基本情况
7. 随着块完成，帧被弹出

**关键洞察：** Session 不会递归——它们是 leaf 节点。VM 管理整个调用树。

### 作用域解析

解析变量名称时：

1. 检查当前帧的 `local_bindings`
2. 检查父帧的 `local_bindings`（词法作用域）
3. 继续沿着调用栈向上直到根部
4. 检查全局作用域（导入、代理、块）
5. 如果未找到则报错

```
do process(chunk, 5)           # execution_id: 1
  let parts = ...              # parts 绑定在 execution_id: 1
  do process(parts[0], 4)      # execution_id: 2
    let parts = ...            # 新的 parts 绑定在 execution_id: 2（遮蔽父级）
    # 访问 'chunk' 解析为 execution_id: 2 的参数
```

**只有局部绑定是有作用域的。** 全局定义（代理、块、导入）在所有帧之间共享。

### 递归深度限制

默认最大深度：**100**

按块配置：

```prose
block process(chunk, depth) (max_depth: 50):
  ...
```

如果超过限制：

```
[Error] RecursionLimitExceeded: 块 'process' 超过了 max_depth 50
```

### 状态中的调用栈

VM 通过 `state.md`（文件系统）或对话（上下文内）中的标记来跟踪调用栈：

```
#1 process(data,5)
  #2 process(parts[0],4)
    #3 process(subparts[0],3)  ← 正在执行
```

块调用使用 `#ID block` 开始，并使用 `#ID done` 完成。嵌套直观地显示了调用栈。

---

## 管道（Pipeline）执行

```prose
let results = items
  | filter:
      session "保留吗？yes/no"
        context: item
  | map:
      session "转换"
        context: item
```

从左到右执行：

1. **filter**：保留 session 返回真值的项目
2. **map**：通过 session 转换每个项目
3. **reduce**：两两累积项目
4. **pmap**：类似于 map 但并发执行

---

## 字符串插值

```prose
let name = session "获取用户名"
session "你好 {name}，欢迎！"
```

在产生 session 之前，用变量值替换 `{varname}`。

---

## 完整执行算法

```
function execute(program, inputs?):
  1. 收集所有 use 语句，获取并注册导入
  2. 收集所有输入声明，从调用者处绑定值
  3. 收集所有代理定义
  4. 收集所有块定义
  5. 按顺序处理每条语句：
     - 如果是 session：通过 Task 产生，等待结果
     - 如果是 resume：加载内存，通过 Task 产生，等待结果
     - 如果是 let/const：执行等号右侧（RHS），绑定结果
     - 如果是 output：执行 RHS，绑定结果，注册为输出
     - 如果是程序调用：使用输入调用导入的程序，接收输出
     - 如果是 parallel：产生所有分支，根据策略等待
     - 如果是 loop：评估条件，执行正文，重复
     - 如果是 try：执行 try，出错时执行 catch，始终执行 finally
     - 如果是 choice/if：评估条件，执行匹配的分支
     - 如果是 do block：使用参数调用块
  6. 根据 try/catch 处理错误或向上传播
  7. 收集所有输出绑定
  8. 将输出返回给调用者（如果未声明输出，则返回最终结果）
```

---

## 实现说明

### Task 工具用法

始终使用 Task 来执行 session：

```
Task({
  description: "OpenProse session",
  prompt: "<带有上下文的 session 提示词>",
  subagent_type: "general-purpose",
  model: "<可选的模型覆盖>"
})
```

### 并行执行

在单次响应中进行多次 Task 调用以实现真正的并发：

```
// 在一次响应中，调用所有三个：
Task({ prompt: "A" })
Task({ prompt: "B" })
Task({ prompt: "C" })
```

### 上下文序列化

向 session 传递上下文时：

- 加上清晰的标签作为前缀
- 保留相关信息
- 如果太长则进行摘要
- 保持语义

---

## 总结

OpenProse VM：

1. 通过 `use` 语句从 `p.prose.md` **导入**程序
2. 将来自调用者的输入**绑定**到程序变量
3. **解析**程序结构
4. **收集**定义（代理、块）
5. 按顺序**执行**语句
6. 通过 Task 工具**产生** session
7. 以内存**恢复**持久化代理
8. 使用输入**调用**导入的程序，接收输出
9. **协调**并行执行
10. 智能地**评估**自由裁量条件
11. **管理** session 之间的上下文流
12. 使用 try/catch/retry **处理**错误
13. 在文件（`.prose/runs/`）或对话中**跟踪**状态
14. 向调用者**返回**输出绑定

该语言的设计使其不言自明。当对语法产生疑问时，请将其解释为为了实现无歧义控制流而结构化的自然语言。
