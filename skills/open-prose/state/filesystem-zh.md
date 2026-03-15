---
role: file-system-state-management
summary: |
  OpenProse 程序的文件系统状态管理。此方法将执行状态持久化到 `.prose/` 目录，支持检查、恢复和长时运行的工作流。
see-also:
  - ../prose.md: VM 执行语义
  - in-context.md: 上下文内状态管理 (in-context state management，替代方法)
  - sqlite.md: SQLite 状态管理 (实验性)
  - postgres.md: PostgreSQL 状态管理 (实验性)
  - ../primitives/session.md: Session 上下文和压缩指南
---

# 文件系统状态管理 (File-System State Management)

本文档描述 OpenProse VM 如何利用 **`.prose/` 目录中的文件**来跟踪执行状态。这是两种状态管理方法之一（另一种是 `in-context.md` 中的上下文内状态）。

> **备注**：文件系统模式是 OpenProse 的默认模式，适用于大多数非平凡的自动化任务。它通过将状态写入本地磁盘，确保即使会话中断也能通过 `prose run` 继续执行。

## 概览 (Overview)

基于文件的状态将所有执行产物持久化到磁盘。这支持：

- **检查 (Inspection)**：准确查看每一步发生了什么
- **恢复 (Resumption)**：接续中断的程序
- **长时运行的工作流 (Long-running workflows)**：处理超出上下文限制的程序
- **调试 (Debugging)**：追踪执行历史

**核心原则**：文件是可检查的产物。目录结构即执行状态。

---

## 目录结构 (Directory Structure)

```
# 项目级状态 (在工作目录中)
.prose/
├── .env                              # 配置 (简单的 key=value 格式)
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose             # 运行程序的副本
│       ├── state.md                  # 带有代码片段的执行状态
│       ├── bindings/
│       │   ├── {name}.md             # 根作用域绑定
│       │   └── {name}__{execution_id}.md  # 作用域绑定 (block 调用)
│       ├── imports/
│       │   └── {handle}--{slug}/     # 嵌套程序执行 (递归采用相同结构)
│       └── agents/
│           └── {name}/
│               ├── memory.md         # Agent 的当前状态
│               ├── {name}-001.md     # 历史片段 (扁平化)
│               ├── {name}-002.md
│               └── ...
└── agents/                           # 项目范围的 Agent 记忆
    └── {name}/
        ├── memory.md
        ├── {name}-001.md
        └── ...

# 用户级状态 (在主目录中)
~/.prose/
└── agents/                           # 用户范围的 Agent 记忆 (跨项目)
    └── {name}/
        ├── memory.md
        ├── {name}-001.md
        └── ...
```

### 运行 ID 格式 (Run ID Format)

格式：`{YYYYMMDD}-{HHMMSS}-{random6}`

示例：`20260115-143052-a7b3c9`

无需 "run-" 前缀——目录名已能清晰体现上下文。

### 片段编号 (Segment Numbering)

片段使用 3 位零填充数字：`captain-001.md`, `captain-002.md` 等。

如果程序超过 999 个片段，扩展至 4 位：`captain-1000.md`。

---

## 文件格式 (File Formats)

### `.prose/.env`

简单的 key=value 配置文件：

```env
OPENPROSE_POSTGRES_URL=postgresql://user:pass@localhost:5432/prose
```

---

### `state.md` —— 只增执行日志 (Append-Only Execution Log)

状态文件是执行事件的**只增日志 (append-only log)**。VM 随着执行进度追加条目，而不是在每条语句后重写整个文件。

**只有 VM 写入此文件**。Subagents 永远不会修改 `state.md`。

**核心原则**：VM 的对话历史是主要的执行状态。状态文件存在是为了恢复和调试，而不是正常执行期间的真理来源。

#### 格式 (Format)

```markdown
# run:20260115-143052-a7b3c9 feature-implementation.prose

1→ research ✓
2→ ∥start a,b,c
2a→ a ✓
2b→ b ✓
2c→ c ✓
2→ ∥done
3→ loop:1/5
3→ synthesis ✓
3→ loop:2/5 exit(**complete**)
4→ captain ✓
---end 2026-01-15T14:35:22Z
```

#### 事件标记 (Event Markers)

| 标记 | 含义 | 示例 |
|--------|---------|---------|
| `N→ name ✓` | 语句 N 已完成，绑定已写入 | `1→ research ✓` |
| `N→ ✓` | 匿名 session 已完成 | `5→ ✓` |
| `N→ ∥start a,b,c` | parallel 块启动，包含分支 | `2→ ∥start a,b,c` |
| `Na→ name ✓` | parallel 分支已完成 | `2a→ a ✓` |
| `N→ ∥done` | parallel 块已合并 (joined) | `2→ ∥done` |
| `N→ loop:I/M` | loop 迭代 I，最大 M 次 | `3→ loop:2/5` |
| `N→ loop:I/M exit(reason)` | loop 已退出 | `3→ loop:3/5 exit(**done**)` |
| `N→ block:name#ID` | block 调用开始 | `4→ block:process#43` |
| `N→ #ID done` | block 调用完成 | `4→ #43 done` |
| `N→ ✗ error` | 语句执行失败 | `5→ ✗ timeout` |
| `N→ retry:A/M` | retry 尝试 A，最大 M 次 | `5→ retry:2/3` |
| `---end TIMESTAMP` | 程序已完成 | `---end 2026-01-15T14:35:22Z` |
| `---error TIMESTAMP msg` | 程序执行失败 | `---error 2026-01-15T14:35:22Z timeout` |

#### VM 何时写入 (When the VM Writes)

VM 向 `state.md` 追加内容：

| 事件 | 动作 |
|-------|--------|
| 语句完成 | 追加完成标记 |
| Parallel 启动/合并 | 追加 parallel 标记 |
| Loop 迭代/退出 | 追加 loop 标记 |
| Block 调用/完成 | 追加 block 标记 |
| 发生错误 | 追加 error 标记 |
| 程序结束 | 追加 end 标记 |

**注意**：VM 不会重写整个文件。每次写入仅追加一行，保持令牌 (token) 生成量最小。

#### 恢复 (Resumption)

要恢复中断的运行，VM 会：

1. 读取 `state.md` 找到最后完成的语句
2. 扫描 `bindings/` 目录以查找现有输出
3. 从下一条语句继续

只增格式使此过程非常直接——找到最后一行，确定位置即可。

---

## `bindings/{name}.md`

所有命名值（input, output, let, const）都存储为绑定文件。

```markdown
# research

kind: let

source:
```prose
let research = session: researcher
  prompt: "Research AI safety"
```

---

AI safety research covers several key areas including alignment,
robustness, and interpretability. The field has grown significantly
since 2020 with major contributions from...
```

**结构**：
- 包含绑定名称的页眉
- `kind:` 字段指明类型 (input, output, let, const)
- `source:` 代码片段显示来源
- `---` 分隔符
- 下方为实际值

**`kind` 字段区分以下类型**：

| Kind | 含义 |
|------|---------|
| `input` | 从调用者接收的值 |
| `output` | 返回给调用者的值 |
| `let` | 可变变量 |
| `const` | 不可变变量 |

### 匿名 Session 绑定 (Anonymous Session Bindings)

没有显式捕获输出的 session 仍会产生结果：

```prose
session "Analyze the codebase"   # 无 `let x = ...` 捕获
```

这些会自动生成带 `anon_` 前缀的名称：

- `bindings/anon_001.md`
- `bindings/anon_002.md`
- 等。

这确保了所有 session 输出都被持久化且可检查。

---

### 作用域绑定 (Scoped Bindings / Block Invocations)

当在 block 调用内部创建绑定时，它被限定在对应的执行帧 (execution frame) 内，以防止递归调用间的冲突。

**命名约定**：`{name}__{execution_id}.md`

示例：
- `bindings/result__43.md` —— 执行 ID 43 中的绑定 `result`
- `bindings/parts__44.md` —— 执行 ID 44 中的绑定 `parts`

**带有执行作用域的文件格式**：

```markdown
# result

kind: let
execution_id: 43

source:
```prose
let result = session "Process chunk"
```

---

Processed chunk into 3 sub-parts...
```

**作用域解析 (Scope resolution)**：VM 通过检查以下位置解析变量引用：
1. `{name}__{current_execution_id}.md`
2. `{name}__{parent_execution_id}.md`
3. 继续沿调用栈向上
4. `{name}.md` (根作用域)

最先匹配到的获胜。

**递归调用的示例目录**：

```
bindings/
├── data.md              # 根作用域输入
├── result__1.md         # 第一次 process() 调用
├── parts__1.md          # 第一次调用的 parts
├── result__2.md         # 递归调用 (深度 2)
├── parts__2.md          # 深度 2 的 parts
├── result__3.md         # 递归调用 (深度 3)
└── ...
```

---

### Agent 记忆文件 (Agent Memory Files)

#### `agents/{name}/memory.md`

Agent 当前累积的状态：

```markdown
# Agent Memory: captain

## 当前理解 (Current Understanding)

项目正在实现用于用户管理的 REST API。
架构使用 Express + PostgreSQL。测试覆盖率目标为 80%。

## 已做出的决策 (Decisions Made)

- 2026-01-15: 批准使用 JWT 而非 session tokens (更简单的无状态认证)
- 2026-01-15: 设置 80% 覆盖率阈值 (平衡质量与速度)

## 待处理问题 (Open Concerns)

- 登录端点尚未实现速率限制
- 需要验证 OAuth 流程是否兼容新的 token 格式
```

#### `agents/{name}/{name}-NNN.md` (片段 / Segments)

每次调用的历史记录，在同一目录下扁平化存储：

```markdown
# Segment 001

timestamp: 2026-01-15T14:32:15Z
prompt: "Review the research findings"

## 摘要 (Summary)

- 已评审：来自 parallel research session 的文档
- 发现：核心概念覆盖良好，缺失边缘情况
- 决策：继续执行实现，记录差距待以后处理
- 下一步：根据确定的差距评审实现
```

---

## 谁写入什么 (Who Writes What)

| 文件 | 写入者 |
|------|------------|
| `state.md` | 仅 VM |
| `bindings/{name}.md` | Subagent |
| `agents/{name}/memory.md` | 持久化 Agent |
| `agents/{name}/{name}-NNN.md` | 持久化 Agent |

VM 负责编排；subagents 直接将自己的输出写入文件系统。**VM 永远不持有完整的绑定值——它只跟踪文件路径。**

---

## Subagent 输出写入 (Subagent Output Writing)

当 VM 启动一个 session 时，它会告知 subagent 将输出写入何处。

### 对于普通 Sessions (Regular Sessions)

```
完成此任务后，将输出写入：
  .prose/runs/20260115-143052-a7b3c9/bindings/research.md

格式：
# research

kind: let

source:
```prose
let research = session: researcher
  prompt: "Research AI safety"
```

---

[此处为你的输出]
```

### 对于持久化 Agents (Persistent Agents / resume:)

```
你的记忆位于：
  .prose/runs/20260115-143052-a7b3c9/agents/captain/memory.md

请先阅读它以了解之前的上下文。完成后，根据 primitives/session.md 中的指南，
使用你压缩后的状态更新它。

同时将你的片段记录写入：
  .prose/runs/20260115-143052-a7b3c9/agents/captain/captain-003.md
```

### Subagents 返回给 VM 的内容 (What Subagents Return to the VM)

写入输出后，subagent 返回一个**确认消息**——而不是完整内容：

**根作用域 (block 调用外部)**：
```
Binding written: research
Location: .prose/runs/20260115-143052-a7b3c9/bindings/research.md
Summary: AI safety research covering alignment, robustness, and interpretability with 15 citations.
```

**block 调用内部 (包含 execution_id)**：
```
Binding written: result
Location: .prose/runs/20260115-143052-a7b3c9/bindings/result__43.md
Execution ID: 43
Summary: Processed chunk into 3 sub-parts for recursive processing.
```

VM 记录位置并继续。它**不**读取文件——它只是将引用传递给后续需要该上下文的 sessions。

---

## 导入的递归结构 (Imports Recursive Structure)

导入的程序递归地使用**相同的统一结构**：

```
.prose/runs/{id}/imports/{handle}--{slug}/
├── program.prose
├── state.md
├── bindings/
│   └── {name}.md
├── imports/                    # 嵌套导入在此处
│   └── {handle2}--{slug2}/
│       └── ...
└── agents/
    └── {name}/
```

这允许无限的嵌套深度，同时在每个级别保持一致的结构。

---

## 持久化 Agent 的记忆作用域 (Memory Scoping for Persistent Agents)

| 作用域 (Scope) | 声明方式 | 路径 | 生命周期 |
|-------|-------------|------|----------|
| 执行 (默认) | `persist: true` | `.prose/runs/{id}/agents/{name}/` | 随运行结束而销毁 |
| 项目 | `persist: project` | `.prose/agents/{name}/` | 在项目内的多次运行中存活 |
| 用户 | `persist: user` | `~/.prose/agents/{name}/` | 跨项目存活 |
| 自定义 | `persist: "path"` | 指定路径 | 用户控制 |

---

## VM 更新协议 (VM Update Protocol)

每条语句完成后，VM 会：

1. **确认** subagent 已写入其输出文件
2. 向 `state.md` **追加**单行标记
3. **继续**下一条语句

VM 每个事件追加一行——它永远不会重写完整的状态文件。这使执行期间的令牌生成量降至最低。

---

## 恢复执行 (Resuming Execution)

如果执行中断，通过以下步骤恢复：

1. 读取 `.prose/runs/{id}/state.md` —— 找到最后完成的标记
2. 扫描 `bindings/` 目录以确认现有输出
3. 从下一条语句继续执行

只增日志格式使恢复变得简单：最后一行指示了执行停止的位置。
