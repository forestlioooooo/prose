---
role: in-context-state-management
summary: |
  使用带有文本标记的叙述协议进行上下文内状态管理 (in-context state management)。
  此方法在对话历史本身内部跟踪执行状态。
  OpenProse VM 通过“大声思考”来持久化状态——你说的话就成了你记得的事。
see-also:
  - ../prose.md: VM 执行语义
  - filesystem.md: 文件系统状态管理 (alternative approach，替代方法)
  - sqlite.md: SQLite 状态管理 (实验性)
  - postgres.md: PostgreSQL 状态管理 (实验性)
  - ../primitives/session.md: Session 上下文和压缩指南
---

# 上下文内状态管理 (In-Context State Management)

本文档描述 OpenProse VM 如何通过对话历史中的**结构化叙述**来跟踪执行状态。这是两种状态管理方法之一（另一种是 `filesystem.md` 中的基于文件的状态）。

## 概览 (Overview)

上下文内状态使用带前缀的文本标记将状态持久化在对话中。VM 对执行过程进行“大声思考”——你说的话就成了你记得的事。

**核心原则**：你的对话历史就是 VM 的工作记忆。

---

## 何时使用上下文内状态 (When to Use In-Context State)

上下文内状态适用于：

| 因素 | 上下文内 (In-Context) | 建议改为基于文件的状态 (File-Based) |
|--------|------------|------------------------|
| 语句数量 | < 30 条 | >= 30 条 |
| Parallel 分支 | < 5 个并发 | >= 5 个并发 |
| 导入的程序 | 0-2 个导入 | >= 3 个导入 |
| 嵌套深度 | <= 2 层 | > 2 层 |
| 预期持续时间 | < 5 分钟 | >= 5 分钟 |

在程序开始时宣布你的状态模式：

```
OpenProse Program Start
   State mode: in-context (程序较小，适合放入上下文)
```

---

## 叙述协议 (The Narration Protocol)

使用**紧凑标记 (compact markers)** 以最小的令牌 (token) 开销跟踪状态。VM 的对话历史是主要状态——标记的存在是为了清晰和潜在的恢复，而不是详细的日志。

### 核心标记 (Core Markers)

| 标记 | 含义 | 示例 |
|--------|---------|---------|
| `N→ name ✓` | 语句 N 已完成，已绑定到名称 | `1→ research ✓` |
| `N→ ✓` | 匿名 session 已完成 | `3→ ✓` |
| `N→ ✗ error` | 语句执行失败 | `2→ ✗ timeout` |
| `∥ [a b c]` | Parallel 已启动 | `∥ [security perf style]` |
| `∥ [a✓ b✓ c→]` | Parallel 进度 | `∥ [security✓ perf✓ style→]` |
| `∥ done` | Parallel 已合并 (joined) | `∥ done` |
| `loop:I/M` | Loop 迭代 | `loop:2/5` |
| `loop exit` | Loop 条件已满足 | `loop:3/5 exit` |
| `#ID name` | Block 调用 | `#43 process` |
| `#ID done` | Block 调用完成 | `#43 done` |
| `try→` | 进入 try | `try→` |
| `catch→` | 进入 catch | `catch→ err` |
| `finally→` | 进入 finally | `finally→` |

---

## 各结构的叙述模式 (Narration Patterns by Construct)

### Session 语句

```
1→ research ✓
```

仅需一行。Task 工具调用和结果都在对话中——无需再次叙述。

### Parallel 块

```
∥ [a b c]
  [a, b, c 的 Task 调用]
∥ [a✓ b✓ c✓] done
```

### Loop 块

```
loop:1/5
  3→ synthesis ✓
loop:2/5
  3→ synthesis ✓
loop:3/5 exit(**complete**)
```

### 错误处理 (Error Handling)

```
try→
  2→ ✗ timeout
catch→ err
  3→ recovery ✓
finally→
  4→ cleanup ✓
```

### Block 调用 (Block Invocation)

```
#1 process(data,5)
  5→ parts ✓
  #2 process(parts[0],4)
    6→ subparts ✓
  #2 done
  7→ combined ✓
#1 done
```

Block 调用在视觉上嵌套。`#ID` 唯一标识每次调用，用于作用域绑定。

### 作用域绑定 (Scoped Bindings)

在 block 内部，绑定隐式地限定在当前的 `#ID` 作用域内：

```
#43 process
  5→ result ✓   (作用域限定在 #43)
```

### 程序导入 (Program Imports)

```
use alice/research → research
research(topic:"quantum") → result ✓
```

---

## 上下文序列化 (Context Serialization)

**上下文内状态传递的是值，而非引用**。VM 直接在对话历史中持有绑定值。

将上下文传递给 sessions 时，请进行适当格式化：

| 上下文大小 | 策略 |
|--------------|----------|
| < 2000 字符 | 原样传递 (verbatim) |
| 2000-8000 字符 | 总结关键点 |
| > 8000 字符 | 仅提取精要 |

**局限性**：上下文内状态不支持带有任意大型绑定的 RLM 风格模式。对于大型中间值，请使用文件系统或 PostgreSQL 状态。

---

## 完整执行追踪示例 (Complete Execution Trace Example)

```prose
agent researcher:
  model: sonnet

let research = session: researcher
  prompt: "Research AI safety"

parallel:
  a = session "Analyze risk A"
  b = session "Analyze risk B"

loop until **analysis complete** (max: 3):
  session "Synthesize"
    context: { a, b, research }
```

**紧凑叙述 (Compact narration)**：
```
1→ research ✓
∥ [a b]
∥ [a✓ b✓] done
loop:1/3
  3→ synthesis ✓
loop:2/3 exit(**complete**)
---end
```

整个执行追踪仅需 7 行，而不是 40 多行。Task 工具调用及其结果都在对话历史中——标记仅用于跟踪位置和完成情况。

---

## VM 隐式跟踪的内容 (What the VM Tracks Implicitly)

VM 的对话自然包含以下内容：

| 信息 | 存储位置 |
|-------------|----------------|
| Agent/block 定义 | 程序开始时读取，位于早期上下文中 |
| 绑定值 (Binding values) | 对话中的 Task 工具结果 |
| 当前位置 | VM 知道它刚刚执行了什么 |
| Loop 迭代 | VM 正在计数 |
| Parallel 状态 | VM 启动了任务，能看到返回结果 |
| 调用栈 (Call stack) | VM 调用了 block |

紧凑标记的存在是为了**清晰和恢复**，而不是作为主要的存储媒介。对话历史本身就是状态。

---

## 与文件系统状态的独立性 (Independence from File-Based State)

上下文内状态和文件系统状态 (`filesystem.md`) 是**相互独立的方法**。你可以根据程序的复杂程度二选一。

- **上下文内 (In-context)**：状态存在于对话历史中
- **基于文件 (File-based)**：状态存在于 `.prose/runs/{id}/` 中

它们的设计并非互补——在程序开始时选择合适的模式即可。

---

## 总结 (Summary)

上下文内状态管理：

1. 使用**紧凑标记** (`1→ research ✓`) 而非冗长的叙述
2. 依赖**对话历史**作为主要状态
3. 适用于**较小、较简单的程序** (<30 条语句)
4. 每条语句生成**最小令牌数**
5. 通过读取之前的标记支持执行恢复

对话就是状态。标记在不增加令牌负担的情况下提供结构和可恢复性。
