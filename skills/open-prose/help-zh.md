# OpenProse 帮助文档

当用户调用 `prose help` 或询问有关 OpenProse 的问题时，加载此文件。

---

## 欢迎使用

OpenProse 是一种用于 AI 会话的编程语言。你可以编写结构化的程序来编排 AI 代理 (agents)，而虚拟机 (VM)（即当前会话）通过生成真实的子代理来执行这些程序。

**一个长时间运行的 AI 会话就是一个图灵完备的计算机。OpenProse 是专门为其设计的编程语言。**

---

## 你想自动化什么？

当用户调用 `prose help` 时，引导他们定义想要构建的内容。使用 AskUserQuestion 工具：

```
问题: "你想用 OpenProse 自动化处理什么任务？"
标题: "目标"
选项:
  1. "运行工作流" - "我有一个现成的 .prose 文件需要执行"
  2. "构建新项目" - "帮我为特定任务创建一个程序"
  3. "学习语法" - "查看示例并了解其工作原理"
  4. "探索可能性" - "OpenProse 能做什么？"
```

**在用户回答后：**

- **运行工作流**：询问文件路径，然后加载 `prose.md` 并执行。
- **构建新项目**：请他们描述任务，然后协助编写 .prose 程序（加载 `guidance/patterns.md`）。
- **学习语法**：展示 `examples/` 目录下的示例，解释虚拟机模型。
- **探索可能性**：演示关键示例，如 `37-the-forge.prose` 或 `28-gas-town.prose`。

---

## 可用命令

| 命令 | 功能 |
|---------|--------------|
| `prose help` | 此帮助文档 - 引导你找到所需内容 |
| `prose run <file>` | 执行 .prose 程序 |
| `prose compile <file>` | 仅验证语法而不运行 |
| `prose update` | 迁移旧版工作区文件 |
| `prose examples` | 浏览并运行示例程序 |

---

## 快速入门

**运行一个示例：**
```
prose run examples/01-hello-world.prose
```

**创建你的第一个程序：**
```
prose help
→ 选择 "构建新项目"
→ 描述你想自动化的任务
```

---

## 常见问题 (FAQs)

### 支持哪些 AI 助手？

Claude Code, OpenCode 和 Amp。任何运行足够智能的模型并支持“子代理 (subagents)”等原语的系统都被视为 "Prose Complete"（具备运行 Prose 的完整能力）。

### 这怎么能算是一个虚拟机 (VM)？

大语言模型 (LLM) 是模拟器——当给定详细的系统描述时，它们不仅是描述，而是在模拟。`prose.md` 规范以足够的保真度描述了一个 VM，阅读它就能诱导模拟行为。而具有足够保真度的模拟就是实现：每个会话生成一个真实的子代理，输出是真实的产物，状态持久化在对话历史或文件中。模拟即是执行。

### 什么是“智能控制反转 (Intelligent IoC)”？

传统的 IoC 容器（如 Spring, Guice）通过配置文件装配依赖关系。OpenProse 的容器是一个通过“理解”来装配代理的 AI 会话。它不只是匹配名称——它理解上下文、意图，并能对执行做出智能决策。

### 这看起来很像 Python。

语法是有意设计成类似 Python 的——Python 基于缩进的结构非常易读且自明。但其语义完全不同。OpenProse 没有函数，没有类，也没有通用的计算能力。它只有代理 (agents)、会话 (sessions) 和控制流。设计原则是：结构化但自明，无需过多文档即可实现无歧义的解释。

### 为什么不直接用英语？

英语本身就是一个代理框架——我们不是要取代它，而是要结构化它。纯英语无法区分顺序执行和并行执行，无法指定重试次数，也无法限定变量作用域。OpenProse 在需要模糊性作为特性时使用英语（在 `**...**` 内部），在其他地方使用结构化语法。这种“第四面墙”语法让你能在需要时精确地依赖 AI 的判断。

### 为什么不用 YAML？

我们最初尝试过 YAML。问题在于：循环、条件判断和变量声明在 YAML 中并不直观——而当你试图让它们变得直观时，代码会变得冗长且丑陋。更根本的是，YAML 是为机器解析而优化的，而 OpenProse 是为机器的“智能阅读/领悟”而优化的。它不需要被传统意义上解析——它需要被理解。这是完全不同的设计目标。

### 为什么不用 LangChain/CrewAI/AutoGen？

那些是编排库——它们从外部协调代理。OpenProse 在代理会话内部运行——会话本身就是 IoC 容器。这意味着零外部依赖，并且可以移植到任何 AI 助手。从 Claude Code 切换到其他平台？你的 .prose 文件依然有效。

---

## 语法一览

```prose
session "prompt"              # 生成子代理
agent name:                   # 定义代理模板
let x = session "..."         # 捕获结果
parallel:                     # 并发执行
repeat N:                     # 固定次数循环
for x in items:               # 迭代循环
loop until **condition**:     # AI 评估的循环
try: ... catch: ...           # 错误处理
if **condition**: ...         # 条件判断
choice **criteria**: option   # AI 选择的分支
block name(params):           # 可重用块
do blockname(args)            # 调用块
items | map: ...              # 管道操作
```

有关完整语法和验证规则，请参见 `compiler.md`。

---

## 示例 (Examples)

`examples/` 目录包含 37 个示例程序：

| 范围 | 类别 |
|-------|----------|
| 01-08 | 基础（hello world、研究、代码审查、调试） |
| 09-12 | 代理 (agents) 和技能 (skills) |
| 13-15 | 变量和组合 |
| 16-19 | 并行执行 |
| 20-21 | 循环和管道 |
| 22-23 | 错误处理 |
| 24-27 | 高级（choice, conditionals, blocks, interpolation） |
| 28 | Gas Town（多代理编排） |
| 29-31 | 船长椅模式 (Captain's chair pattern，持久编排器) |
| 33-36 | 生产工作流（PR 自动修复、内容管道、功能工厂、错误猎人） |
| 37 | The Forge（从零构建浏览器） |

**推荐起点：**
- `01-hello-world.prose` - 最简单的程序
- `16-parallel-reviews.prose` - 体验并行执行
- `37-the-forge.prose` - 观察 AI 从头构建网页浏览器
