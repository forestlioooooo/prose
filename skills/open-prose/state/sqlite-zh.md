---
role: sqlite-state-management
status: experimental
summary: |
  OpenProse 程序的基于 SQLite 的状态管理。此方法将执行状态持久化到 SQLite 数据库，
  支持结构化查询、原子事务和灵活的模式演进。
requires: PATH 中包含 sqlite3 CLI 工具
see-also:
  - ../prose.md: VM 执行语义
  - filesystem.md: 基于文件的状态 (default，默认，更规范)
  - in-context.md: 上下文内状态 (适用于简单程序)
  - ../primitives/session.md: Session 上下文和压缩指南
---

# SQLite 状态管理 (实验性)

本文档描述 OpenProse VM 如何利用 **SQLite 数据库**来跟踪执行状态。这是基于文件状态 (`filesystem.md`) 和上下文内状态 (`in-context.md`) 的一种实验性替代方案。

## 前置条件 (Prerequisites)

**要求**：`sqlite3` 命令行工具必须在你的 PATH 中可用。

| 平台 | 安装方法 |
|----------|--------------|
| macOS | 预装 |
| Linux | `apt install sqlite3` / `dnf install sqlite3` 等 |
| Windows | `winget install SQLite.SQLite` 或从 sqlite.org 下载 |

如果 `sqlite3` 不可用，VM 将回退到文件系统状态并警告用户。

---

## 概览 (Overview)

SQLite 状态提供：

- **原子事务 (Atomic transactions)**：状态更改符合 ACID 原则
- **结构化查询 (Structured queries)**：查找特定绑定，按状态过滤，聚合结果
- **灵活模式 (Flexible schema)**：按需添加列和表
- **单文件便携性 (Single-file portability)**：整个运行状态就是一个 `.db` 文件
- **并发访问 (Concurrent access)**：SQLite 自动处理锁定

**核心原则**：数据库是一个灵活的工作空间。VM 和 subagents 将其作为一种协调机制共享，而不是刚性契约。

---

## 数据库位置 (Database Location)

数据库存储在标准的运行目录中：

```
.prose/runs/{YYYYMMDD}-{HHMMSS}-{random}/
├── state.db          # SQLite 数据库 (即此文件)
├── program.prose     # 运行程序的副本
└── attachments/      # DB 无法容纳的大型输出 (可选)
```

**运行 ID 格式**：与文件系统状态相同：`{YYYYMMDD}-{HHMMSS}-{random6}`

示例：`.prose/runs/20260116-143052-a7b3c9/state.db`

### 项目级和用户级 Agents (Project-Scoped and User-Scoped Agents)

执行级 agents (默认) 存储在每次运行的 `state.db` 中。然而，**项目级 agents** (`persist: project`) 和 **用户级 agents** (`persist: user`) 必须在多次运行中存活。

对于项目级 agents，使用单独的数据库：

```
.prose/
├── agents.db                 # 项目范围的 Agent 记忆 (在多次运行中持久化)
└── runs/
    └── {id}/
        └── state.db          # 执行范围的状态 (随运行结束而销毁)
```

对于用户级 agents，使用主目录中的数据库：

```
~/.prose/
└── agents.db                 # 用户范围的 Agent 记忆 (跨项目持久化)
```

项目级 agents 的 `agents` 和 `agent_segments` 表位于 `.prose/agents.db`，而用户级 agents 的对应表位于 `~/.prose/agents.db`。VM 在首次使用时初始化这些数据库，并向 subagents 提供正确的路径。

---

## 职责划分 (Responsibility Separation)

本节定义了**谁负责什么**。这是 VM 与 subagents 之间的契约。

### VM 职责

VM (运行 .prose 程序的编排 agent) 负责：

| 职责 | 描述 |
|----------------|-------------|
| **数据库创建** | 运行开始时创建 `state.db` 并初始化核心表 |
| **程序注册** | 存储程序源代码和元数据 |
| **执行跟踪** | 追加完成记录 (而非每条语句更新) |
| **Subagent 启动** | 通过 Task 工具启动 sessions，并提供数据库路径和指令 |
| **Parallel 协调** | 跟踪分支状态，实现合并 (join) 策略 |
| **Loop 管理** | 跟踪迭代次数，评估条件 |
| **错误聚合** | 记录失败，管理 retry 状态 |
| **完成检测** | 结束后将运行标记为已完成 |

**关键点**：VM 的对话历史是主要的执行状态。数据库的存在是为了持久化和协调，而不是正常执行期间的真理来源。VM 在完成事件时追加记录——它**不**在每条语句执行后更新数据库。

### Subagent 职责

Subagents (由 VM 启动的 sessions) 负责：

| 职责 | 描述 |
|----------------|-------------|
| **写入自己的输出** | 在 `bindings` 表中插入/更新其绑定 |
| **记忆管理** | 针对持久化 agents：读取并更新其记忆记录 |
| **片段记录** | 针对持久化 agents：追加片段历史 |
| **附件处理** | 将大型输出写入 `attachments/` 目录，并在 DB 中存储路径 |
| **原子写入** | 更新多个关联记录时使用事务 |

**关键点**：Subagents 仅向 `bindings`, `agents`, 和 `agent_segments` 表执行写入。VM 完全拥有 `execution` 表。完成信号通过底层系统 (Task 工具返回) 发出，而非数据库更新。

**关键点**：Subagents 必须将输出直接写入数据库。VM 不写入 subagent 的输出——它仅在 subagent 完成后读取。

**Subagents 返回给 VM 的内容**：一个包含绑定位置的确认消息——而不是完整内容：

**根作用域 (Root scope)**：
```
Binding written: research
Location: .prose/runs/20260116-143052-a7b3c9/state.db (bindings 表, name='research', execution_id=NULL)
Summary: AI safety research covering alignment, robustness, and interpretability with 15 citations.
```

**block 调用内部**：
```
Binding written: result
Location: .prose/runs/20260116-143052-a7b3c9/state.db (bindings 表, name='result', execution_id=43)
Execution ID: 43
Summary: Processed chunk into 3 sub-parts for recursive processing.
```

VM 跟踪位置而非值。这使 VM 的上下文保持精简，并支持任意大小的中间值。

### 共同关注点 (Shared Concerns)

| 关注点 | 处理者 |
|---------|-------------|
| 模式演进 (Schema evolution) | 任意一方 (按需使用 `CREATE TABLE IF NOT EXISTS`, `ALTER TABLE`) |
| 自定义表 | 任意一方 (扩展请使用 `x_` 前缀) |
| 索引 | 任意一方 (为频繁查询的列添加索引) |
| 清理 | VM (在运行结束时，可选择执行 vacuum) |

---

## 核心模式 (Core Schema)

VM 初始化这些表。这是一个**最小可行模式 (minimum viable schema)**——可自由扩展。

```sql
-- 运行元数据
CREATE TABLE IF NOT EXISTS run (
    id TEXT PRIMARY KEY,
    program_path TEXT,
    program_source TEXT,
    started_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now')),
    status TEXT DEFAULT 'running',  -- running, completed, failed, interrupted
    state_mode TEXT DEFAULT 'sqlite'
);

-- 执行位置和历史
CREATE TABLE IF NOT EXISTS execution (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    statement_index INTEGER,
    statement_text TEXT,
    status TEXT,  -- pending, executing, completed, failed, skipped
    started_at TEXT,
    completed_at TEXT,
    error_message TEXT,
    parent_id INTEGER REFERENCES execution(id),  -- 用于嵌套块
    metadata TEXT  -- 用于结构特定数据的 JSON (loop 迭代, parallel 分支等)
);

-- 所有命名值 (input, output, let, const)
CREATE TABLE IF NOT EXISTS bindings (
    name TEXT,
    execution_id INTEGER,  -- 根作用域为 NULL, block 调用为非空值
    kind TEXT,  -- input, output, let, const
    value TEXT,
    source_statement TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now')),
    attachment_path TEXT,  -- 如果值过大，存储文件路径
    PRIMARY KEY (name, IFNULL(execution_id, -1))  -- IFNULL 处理根作用域的 NULL
);

-- 持久化 Agent 记忆
CREATE TABLE IF NOT EXISTS agents (
    name TEXT PRIMARY KEY,
    scope TEXT,  -- execution, project, user, custom
    memory TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
);

-- Agent 调用历史
CREATE TABLE IF NOT EXISTS agent_segments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_name TEXT REFERENCES agents(name),
    segment_number INTEGER,
    timestamp TEXT DEFAULT (datetime('now')),
    prompt TEXT,
    summary TEXT,
    UNIQUE(agent_name, segment_number)
);

-- 导入注册表
CREATE TABLE IF NOT EXISTS imports (
    alias TEXT PRIMARY KEY,
    source_url TEXT,
    fetched_at TEXT,
    inputs_schema TEXT,  -- JSON
    outputs_schema TEXT  -- JSON
);
```

### 模式约定 (Schema Conventions)

- **时间戳**：使用 ISO 8601 格式 (`datetime('now')`)
- **JSON 字段**：在 `metadata`, `*_schema` 列中将结构化数据存储为 JSON 文本
- **大型值**：如果绑定值超过约 100KB，写入 `attachments/{name}.md` 并存储路径
- **扩展表**：使用 `x_` 前缀 (例如 `x_metrics`, `x_audit_log`)
- **匿名绑定**：没有显式捕获的 session (没有 `let x =` 的 `session "..."`) 使用自动生成的名称：`anon_001`, `anon_002` 等
- **导入绑定**：使用导入别名作为前缀限定作用域：`research.findings`, `research.sources`
- **限定作用域的绑定**：使用 `execution_id` 列——根作用域为 NULL，block 调用为非空值

### 作用域解析查询 (Scope Resolution Query)

对于递归块，绑定被限定在各自的执行帧中。通过向上遍历调用栈来解析变量：

```sql
-- 从 execution_id 43 开始查找绑定 'result'
WITH RECURSIVE scope_chain AS (
  -- 从当前执行开始
  SELECT id, parent_id FROM execution WHERE id = 43
  UNION ALL
  -- 向上遍历至父级
  SELECT e.id, e.parent_id
  FROM execution e
  JOIN scope_chain s ON e.id = s.parent_id
)
SELECT b.* FROM bindings b
LEFT JOIN scope_chain s ON b.execution_id = s.id
WHERE b.name = 'result'
  AND (b.execution_id IN (SELECT id FROM scope_chain) OR b.execution_id IS NULL)
ORDER BY
  CASE WHEN b.execution_id IS NULL THEN 1 ELSE 0 END,  -- 优先选择限定作用域的而非根作用域
  s.id DESC NULLS LAST  -- 优先选择更深 (更局部) 的作用域
LIMIT 1;
```

**如果你已知作用域链，可使用更简单的版本**：

```sql
-- 直接查找：检查当前作用域，然后检查父级，最后检查根作用域
SELECT * FROM bindings
WHERE name = 'result'
  AND (execution_id = 43 OR execution_id = 42 OR execution_id IS NULL)
ORDER BY execution_id DESC NULLS LAST
LIMIT 1;
```

---

## 数据库交互 (Database Interaction)

VM 和 subagents 都通过 `sqlite3` CLI 进行交互。

### 从 VM 角度

```bash
# 初始化数据库
sqlite3 .prose/runs/20260116-143052-a7b3c9/state.db "CREATE TABLE IF NOT EXISTS..."

# 更新执行位置
sqlite3 .prose/runs/20260116-143052-a7b3c9/state.db "
  INSERT INTO execution (statement_index, statement_text, status, started_at)
  VALUES (3, 'session \"Research AI safety\"', 'executing', datetime('now'))
"

# 读取绑定
sqlite3 -json .prose/runs/20260116-143052-a7b3c9/state.db "
  SELECT value FROM bindings WHERE name = 'research'
"

# 检查 parallel 分支状态
sqlite3 .prose/runs/20260116-143052-a7b3c9/state.db "
  SELECT statement_text, status FROM execution
  WHERE json_extract(metadata, '$.parallel_id') = 'p1'
"
```

### 从 Subagents 角度

VM 在启动 subagent 时提供数据库路径和指令：

**根作用域 (block 调用外部)**：

```
你的输出数据库为：
  .prose/runs/20260116-143052-a7b3c9/state.db

完成后，写入你的输出：

sqlite3 .prose/runs/20260116-143052-a7b3c9/state.db "
  INSERT OR REPLACE INTO bindings (name, execution_id, kind, value, source_statement, updated_at)
  VALUES (
    'research',
    NULL,  -- 根作用域
    'let',
    'AI safety research covers alignment, robustness...',
    'let research = session: researcher',
    datetime('now')
  )
"
```

**block 调用内部 (包含 execution_id)**：

```
执行作用域：
  execution_id: 43
  block: process
  depth: 3

你的输出数据库为：
  .prose/runs/20260116-143052-a7b3c9/state.db

完成后，写入你的输出：

sqlite3 .prose/runs/20260116-143052-a7b3c9/state.db "
  INSERT OR REPLACE INTO bindings (name, execution_id, kind, value, source_statement, updated_at)
  VALUES (
    'result',
    43,  -- 限定在此执行作用域
    'let',
    'Processed chunk into 3 sub-parts...',
    'let result = session \"Process chunk\"',
    datetime('now')
  )
"
```

对于持久化 agents (执行范围)：

```
你的记忆存储在数据库中：
  .prose/runs/20260116-143052-a7b3c9/state.db

读取当前状态：
  sqlite3 -json .prose/runs/20260116-143052-a7b3c9/state.db "SELECT memory FROM agents WHERE name = 'captain'"

完成后更新：
  sqlite3 .prose/runs/20260116-143052-a7b3c9/state.db "UPDATE agents SET memory = '...', updated_at = datetime('now') WHERE name = 'captain'"

记录此片段：
  sqlite3 .prose/runs/20260116-143052-a7b3c9/state.db "INSERT INTO agent_segments (agent_name, segment_number, prompt, summary) VALUES ('captain', 3, '...', '...')"
```

对于项目级 agents，使用 `.prose/agents.db`。对于用户级 agents，使用 `~/.prose/agents.db`。

---

## 主线程中的上下文保存 (Context Preservation in Main Thread)

VM 的对话历史是主要的执行状态。数据库存在是为了持久化和调试。

### 紧凑叙述 (Compact Narration)

在对话中使用最小化的标记 (与文件系统/上下文内状态相同)：

```
1→ research ✓
∥ [a b c] done
loop:2/5 exit
```

Task 工具调用和结果都在对话中——无需冗长地叙述。

### 为什么同时使用对话和数据库？

| 目的 | 机制 |
|---------|-----------|
| **工作记忆 (Working memory)** | 对话 (VM 在不查询的情况下“记得”的内容) |
| **持久状态 (Durable state)** | 数据库 (克服上下文限制，支持恢复执行) |
| **调试/检查** | 数据库 (可查询的历史记录) |

对话是主要的；数据库用于持久化和检查。

---

## 并行执行 (Parallel Execution)

对于 parallel 块，请**追加完成记录**而非更新状态。只有 VM 向 `execution` 表执行写入。

```sql
-- VM 追加 parallel 启动记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (5, 'parallel:', 'started', '{"parallel_id": "p1", "branches": ["a", "b", "c"]}');

-- Subagents 写入 bindings 表，Task 工具发出完成信号
-- VM 在每个分支返回时追加其完成记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (5, 'parallel:a', 'completed', '{"parallel_id": "p1", "branch": "a"}');

-- 当所有分支完成，VM 追加合并 (join) 记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (5, 'parallel:', 'joined', '{"parallel_id": "p1"}');
```

**只增模式 (Append-only)**：不对现有行执行 UPDATE。每个事件都是一个新的 INSERT。

---

## Loop 跟踪

```sql
-- VM 追加 loop 启动记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (10, 'loop', 'started', '{"loop_id": "l1", "max": 5, "condition": "**complete**"}');

-- VM 追加每次迭代完成记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (10, 'loop', 'iteration', '{"loop_id": "l1", "iteration": 1}');

INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (10, 'loop', 'iteration', '{"loop_id": "l1", "iteration": 2}');

-- VM 追加退出记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (10, 'loop', 'exited', '{"loop_id": "l1", "iteration": 2, "reason": "condition_satisfied"}');
```

**只增模式**：迭代记录是追加的，而非更新。通过查询 `MAX(iteration)` 获取当前状态。

---

## 错误处理 (Error Handling)

```sql
-- 追加失败记录
INSERT INTO execution (statement_index, statement_text, status, error_message, metadata)
VALUES (15, 'session "Risky"', 'failed', 'Connection timeout after 30s', '{}');

-- 追加 retry 尝试记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (15, 'session "Risky"', 'retry', '{"attempt": 2, "max": 3}');

-- 追加最终成功或最终失败记录
INSERT INTO execution (statement_index, statement_text, status, metadata)
VALUES (15, 'session "Risky"', 'completed', '{"attempt": 2}');
```

**只增模式**：每次重试都是新记录。按 statement_index 查询最新状态。

---

## 大型输出 (Large Outputs)

当绑定值过大，不适合在数据库中存储时 (>100KB)：

1. 将内容写入 `attachments/{binding_name}.md`
2. 在 `attachment_path` 列存储路径
3. 在 `value` 中保留摘要或设为 null

```sql
INSERT INTO bindings (name, kind, value, attachment_path, source_statement)
VALUES (
  'full_report',
  'let',
  'Full analysis report (847KB) - see attachment',
  'attachments/full_report.md',
  'let full_report = session "Generate comprehensive report"'
);
```

---

## 恢复执行 (Resuming Execution)

要恢复中断的运行：

```sql
-- 查找当前位置
SELECT statement_index, statement_text, status
FROM execution
WHERE status = 'executing'
ORDER BY id DESC LIMIT 1;

-- 获取所有已完成的绑定
SELECT name, kind, value, attachment_path FROM bindings;

-- 获取 Agent 记忆状态
SELECT name, memory FROM agents;

-- 检查 parallel 块状态
SELECT json_extract(metadata, '$.branch') as branch, status
FROM execution
WHERE json_extract(metadata, '$.parallel_id') IS NOT NULL
  AND parent_id = (SELECT id FROM execution WHERE status = 'executing' AND statement_text LIKE 'parallel:%');
```

---

## 灵活性鼓励 (Flexibility Encouragement)

与文件系统状态不同，SQLite 状态有意降低了**规范性**。核心模式只是起点。鼓励你：

- 按需向现有表**添加列**
- **创建扩展表** (使用 `x_` 前缀)
- **存储自定义指标** (计时、令牌数、模型信息)
- 为你的查询模式**构建索引**
- 对半结构化数据使用 **JSON 函数**

扩展示例：

```sql
-- 自定义指标表
CREATE TABLE x_metrics (
    execution_id INTEGER REFERENCES execution(id),
    metric_name TEXT,
    metric_value REAL,
    recorded_at TEXT DEFAULT (datetime('now'))
);

-- 添加自定义列
ALTER TABLE bindings ADD COLUMN token_count INTEGER;

-- 为常用查询创建索引
CREATE INDEX idx_execution_status ON execution(status);
```

数据库就是你的工作空间。尽情使用它。

---

## 与其他模式的比较 (Comparison with Other Modes)

| 特性 | filesystem.md | in-context.md | sqlite.md |
|--------|---------------|---------------|-----------|
| **状态位置** | `.prose/runs/{id}/` 文件 | 对话历史 | `.prose/runs/{id}/state.db` |
| **可查询性** | 通过读取文件 | 否 | 是 (SQL) |
| **原子更新** | 否 | 不适用 | 是 (事务) |
| **模式灵活性** | 固定文件结构 | 不适用 | 灵活 (可添加表/列) |
| **恢复执行** | 读取 state.md | 重新阅读对话 | 查询数据库 |
| **复杂度上限** | 高 | 低 (<30 条语句) | 高 |
| **依赖项** | 无 | 无 | sqlite3 CLI |
| **状态** | 稳定 | 稳定 | **实验性** |

---

## 总结 (Summary)

SQLite 状态管理：

1. 每次运行使用**单个数据库文件**
2. 使用**只增写入**以实现最小令牌开销
3. 在 VM 和 subagents 之间提供**清晰的职责划分**
4. 支持通过**结构化查询**检查状态
5. 允许按需进行**灵活的模式演进**
6. 要求安装 **sqlite3 CLI** 工具
7. 目前处于**实验阶段**——预期会有变动

核心契约：VM 追加执行事件 (而非更新)；subagents 将自己的输出直接写入数据库。对话是主要状态；数据库用于持久化和检查。
