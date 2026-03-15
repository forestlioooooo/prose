---
role: postgres-state-management
status: experimental
summary: |
  OpenProse 程序的基于 PostgreSQL 的状态管理。此方法将执行状态持久化到 PostgreSQL 数据库，
  支持真正的并发写入、网络访问、团队协作和高吞吐量负载。
requires: PATH 中包含 psql CLI 工具，运行中的 PostgreSQL 服务器
see-also:
  - ../prose.md: VM 执行语义
  - filesystem.md: 基于文件的状态 (default，默认，更简单)
  - sqlite.md: SQLite 状态 (可查询，单文件)
  - in-context.md: 上下文内状态 (适用于简单程序)
  - ../primitives/session.md: Session 上下文和压缩指南
---

# PostgreSQL 状态管理 (实验性)

本文档描述 OpenProse VM 如何利用 **PostgreSQL 数据库**来跟踪执行状态。这是基于文件状态 (`filesystem.md`)、SQLite 状态 (`sqlite.md`) 和上下文内状态 (`in-context.md`) 的一种实验性替代方案。

## 前置条件 (Prerequisites)

**要求**：
1. `psql` 命令行工具必须在你的 PATH 中可用
2. 一个运行中的 PostgreSQL 服务器 (本地、Docker 或云端)

### 安装 psql (Installing psql)

| 平台 | 安装命令 | 备注 |
|----------|---------|-------|
| macOS (Homebrew) | `brew install libpq && brew link --force libpq` | 仅客户端；无服务器 |
| macOS (Postgres.app) | 从 https://postgresapp.com 下载 | 完整的 GUI 安装 |
| Debian/Ubuntu | `apt install postgresql-client` | 仅客户端 |
| Fedora/RHEL | `dnf install postgresql` | 仅客户端 |
| Arch Linux | `pacman -S postgresql-libs` | 仅客户端 |
| Windows | `winget install PostgreSQL.PostgreSQL` | 完整安装程序 |

安装后，进行验证：

```bash
psql --version    # 应输出：psql (PostgreSQL) 16.x
```

如果 `psql` 不可用，VM 将提供回退到 SQLite 状态的选项。

---

## 概览 (Overview)

PostgreSQL 状态提供：

- **真正的并发写入 (True concurrent writes)**：行级锁定 (row-level locking) 允许 parallel 分支同时进行写入
- **网络访问**：可从任何机器、外部工具或仪表板查询状态
- **团队协作**：多名开发人员可共享运行状态
- **丰富的 SQL**：支持 JSONB 查询、窗口函数、CTE，用于复杂的执行状态分析
- **高吞吐量**：可处理每分钟 1000+ 次写入及多 GB 级的输出
- **耐用性 (Durability)**：基于 WAL 的恢复、时点恢复 (PITR)

**核心原则**：数据库是一个灵活的、共享的工作空间。VM 和 subagents 通过它进行协调，外部工具可实时观察和查询执行状态。

---

## 安全警告 (Security Warning)

**⚠️ 凭据对 subagents 可见。** `OPENPROSE_POSTGRES_URL` 连接字符串会传递给启动的 sessions，以便它们写入输出。这意味着：

- 数据库凭据会出现在 subagent 的上下文中，并可能被记录
- 请将这些凭据视为**非敏感信息**
- 为 OpenProse 使用**专用数据库**，而非生产系统
- 创建一个**有限权限用户**，仅允许访问 `openprose` 模式 (schema)

**建议配置**：
```sql
-- 创建专用用户，赋予最小权限
CREATE USER openprose_agent WITH PASSWORD 'changeme';
CREATE SCHEMA openprose AUTHORIZATION openprose_agent;
GRANT ALL ON SCHEMA openprose TO openprose_agent;
-- 该用户仅能访问 openprose 模式，无法触及其他部分
```

---

## 何时使用 PostgreSQL 状态 (When to Use PostgreSQL State)

PostgreSQL 状态适用于具有特定规模或协作需求的**高级用户**：

| 需求 | PostgreSQL 的优势 |
|------|------------------|
| >5 个并发 parallel 分支同时写入 | SQLite 会锁定表；PostgreSQL 允许行级并发 |
| 外部仪表板查询状态 | PostgreSQL 专为并发读取者设计 |
| 长时工作流的团队协作 | 共享网络访问；无需同步文件 |
| 输出超过 1GB | 批量摄取；无单文件瓶颈 |
| 任务关键型工作流 (数小时/数天) | 强大的耐用性；时点恢复 |

**如果以上需求均不涉及，请使用文件系统或 SQLite 状态。** 它们更简单，且能满足 99% 程序的需要。

### 决策树 (Decision Tree)

```
你的程序是否少于 30 条语句且没有 parallel 块？
  是 -> 使用上下文内状态 (in-context state，零摩擦)
  否 -> 继续...

外部工具 (仪表板、监控、分析) 是否需要查询状态？
  是 -> 使用 PostgreSQL (需要网络访问)
  否 -> 继续...

是否有多台机器或团队成员需要共享访问同一个运行实例？
  是 -> 使用 PostgreSQL (协作需求)
  否 -> 继续...

是否有超过 5 个并发的 parallel 分支需要同时写入？
  是 -> 使用 PostgreSQL (并发需求)
  否 -> 继续...

输出是否会超过 1GB，或每分钟写入次数是否超过 100 次？
  是 -> 使用 PostgreSQL (规模需求)
  否 -> 使用文件系统 (默认) 或 SQLite (如果你需要 SQL 查询)
```

### 并发案例 (The Concurrency Case)

使用 PostgreSQL 的主要动机是 **parallel 执行中的并发写入**：

- SQLite 使用表级锁：parallel 分支必须排队 (序列化)
- PostgreSQL 使用行级锁：parallel 分支可同时写入

如果你的程序有 10 个 parallel 分支同时完成，PostgreSQL 在写入阶段的速度将比 SQLite 快 5-10 倍。

---

## 数据库设置 (Database Setup)

### 选项 1：Docker (推荐)

获取运行中的 PostgreSQL 实例的最快路径：

```bash
docker run -d \
  --name prose-pg \
  -e POSTGRES_DB=prose \
  -e POSTGRES_HOST_AUTH_METHOD=trust \
  -p 5432:5432 \
  postgres:16
```

然后配置连接：

```bash
mkdir -p .prose
echo "OPENPROSE_POSTGRES_URL=postgresql://postgres@localhost:5432/prose" > .prose/.env
```

管理命令：

```bash
docker ps | grep prose-pg    # 检查是否在运行
docker logs prose-pg         # 查看日志
docker stop prose-pg         # 停止
docker start prose-pg        # 再次启动
docker rm -f prose-pg        # 完全移除
```

### 选项 2：本地安装 PostgreSQL

对于偏好原生安装的用户：

**macOS (Homebrew):**

```bash
brew install postgresql@16
brew services start postgresql@16
createdb myproject
echo "OPENPROSE_POSTGRES_URL=postgresql://localhost/myproject" >> .prose/.env
```

**Linux (Debian/Ubuntu):**

```bash
sudo apt install postgresql
sudo systemctl start postgresql
sudo -u postgres createdb myproject
echo "OPENPROSE_POSTGRES_URL=postgresql:///myproject" >> .prose/.env
```

### 选项 3：云端 PostgreSQL

适用于团队协作或生产环境：

| 服务商 | 免费额度 | 冷启动 | 适用场景 |
|----------|-----------|------------|----------|
| **Neon** | 0.5GB, 自动挂起 | 1-3s | 开发、测试 |
| **Supabase** | 500MB, 不自动挂起 | 无 | 需要认证/存储的项目 |
| **Railway** | 每月 $5 额度 | 无 | 简单的生产部署 |

```bash
# 示例：Neon
echo "OPENPROSE_POSTGRES_URL=postgresql://user:pass@ep-name.us-east-2.aws.neon.tech/neondb?sslmode=require" >> .prose/.env
```

---

## 数据库位置 (Database Location)

连接字符串存储在 `.prose/.env` 中：

```
your-project/
├── .prose/
│   ├── .env                    # OPENPROSE_POSTGRES_URL=...
│   └── runs/                   # 执行元数据和附件
│       └── {YYYYMMDD}-{HHMMSS}-{random}/
│           ├── program.prose   # 运行程序的副本
│           └── attachments/    # 大型输出 (可选)
├── .gitignore                  # 应排除 .prose/.env
└── your-program.prose
```

**运行 ID 格式**：`{YYYYMMDD}-{HHMMSS}-{random6}`

示例：`20260116-143052-a7b3c9`

### 环境变量优先级 (Environment Variable Precedence)

VM 按以下顺序检查：

1. `.prose/.env` 中的 `OPENPROSE_POSTGRES_URL`
2. Shell 环境变量中的 `OPENPROSE_POSTGRES_URL`
3. Shell 环境变量中的 `DATABASE_URL` (常见的后备选项)

### 安全性：添加到 .gitignore

```gitignore
# OpenProse 敏感文件
.prose/.env
.prose/runs/
```

---

## 职责划分 (Responsibility Separation)

本节定义了**谁负责什么**。这是 VM 与 subagents 之间的契约。

### VM 职责

VM (运行 .prose 程序的编排 agent) 负责：

| 职责 | 描述 |
|----------------|-------------|
| **模式初始化** | 运行开始时创建 `openprose` 模式和表 |
| **运行注册** | 存储程序源代码和元数据 |
| **执行跟踪** | 随着语句执行更新位置、状态和计时 |
| **Subagent 启动** | 通过 Task 工具启动 sessions，并提供数据库指令 |
| **Parallel 协调** | 跟踪分支状态，实现合并策略 |
| **Loop 管理** | 跟踪迭代次数，评估条件 |
| **错误聚合** | 记录失败，管理 retry 状态 |
| **上下文保存** | 在主线程中保持充分的叙述 |
| **完成检测** | 结束后将运行标记为已完成 |

**关键点**：VM 必须在自己的对话中保留足够的上下文，以便在不重新查询整个数据库的情况下理解执行状态。数据库用于协调和持久化，而不是工作记忆的替代品。

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
Location: openprose.bindings (满足 name='research' AND run_id='20260116-143052-a7b3c9' AND execution_id IS NULL)
Summary: AI safety research covering alignment, robustness, and interpretability with 15 citations.
```

**block 调用内部**：
```
Binding written: result
Location: openprose.bindings (满足 name='result' AND run_id='20260116-143052-a7b3c9' AND execution_id=43)
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
| 清理 | VM (在运行结束时，可选择删除旧数据) |

---

## 核心模式 (Core Schema)

VM 使用 `openprose` 模式初始化这些表。这是一个**最小可行模式**——可自由扩展。

```sql
-- 为 OpenProse 状态创建专用模式
CREATE SCHEMA IF NOT EXISTS openprose;

-- 运行元数据
CREATE TABLE IF NOT EXISTS openprose.run (
    id TEXT PRIMARY KEY,
    program_path TEXT,
    program_source TEXT,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status TEXT NOT NULL DEFAULT 'running'
        CHECK (status IN ('running', 'completed', 'failed', 'interrupted')),
    state_mode TEXT NOT NULL DEFAULT 'postgres',
    metadata JSONB DEFAULT '{}'::jsonb
);

-- 执行位置和历史
CREATE TABLE IF NOT EXISTS openprose.execution (
    id SERIAL PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES openprose.run(id) ON DELETE CASCADE,
    statement_index INTEGER NOT NULL,
    statement_text TEXT,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'executing', 'completed', 'failed', 'skipped')),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error_message TEXT,
    parent_id INTEGER REFERENCES openprose.execution(id) ON DELETE CASCADE,
    metadata JSONB DEFAULT '{}'::jsonb
);

-- 所有命名值 (input, output, let, const)
CREATE TABLE IF NOT EXISTS openprose.bindings (
    name TEXT NOT NULL,
    run_id TEXT NOT NULL REFERENCES openprose.run(id) ON DELETE CASCADE,
    execution_id INTEGER,  -- 根作用域为 NULL, block 调用为非空值
    kind TEXT NOT NULL CHECK (kind IN ('input', 'output', 'let', 'const')),
    value TEXT,
    source_statement TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    attachment_path TEXT,
    metadata JSONB DEFAULT '{}'::jsonb,
    PRIMARY KEY (name, run_id, COALESCE(execution_id, -1))  -- 带作用域的复合主键
);

-- 持久化 Agent 记忆
CREATE TABLE IF NOT EXISTS openprose.agents (
    name TEXT NOT NULL,
    run_id TEXT,  -- 项目级和用户级 agents 此项为 NULL
    scope TEXT NOT NULL CHECK (scope IN ('execution', 'project', 'user', 'custom')),
    memory TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata JSONB DEFAULT '{}'::jsonb,
    PRIMARY KEY (name, COALESCE(run_id, '__project__'))
);

-- Agent 调用历史
CREATE TABLE IF NOT EXISTS openprose.agent_segments (
    id SERIAL PRIMARY KEY,
    agent_name TEXT NOT NULL,
    run_id TEXT,  -- 项目级 agents 此项为 NULL
    segment_number INTEGER NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    prompt TEXT,
    summary TEXT,
    metadata JSONB DEFAULT '{}'::jsonb,
    UNIQUE (agent_name, COALESCE(run_id, '__project__'), segment_number)
);

-- 导入注册表
CREATE TABLE IF NOT EXISTS openprose.imports (
    alias TEXT NOT NULL,
    run_id TEXT NOT NULL REFERENCES openprose.run(id) ON DELETE CASCADE,
    source_url TEXT NOT NULL,
    fetched_at TIMESTAMPTZ,
    inputs_schema JSONB,
    outputs_schema JSONB,
    content_hash TEXT,
    metadata JSONB DEFAULT '{}'::jsonb,
    PRIMARY KEY (alias, run_id)
);

-- 常用查询索引
CREATE INDEX IF NOT EXISTS idx_execution_run_id ON openprose.execution(run_id);
CREATE INDEX IF NOT EXISTS idx_execution_status ON openprose.execution(status);
CREATE INDEX IF NOT EXISTS idx_execution_parent_id ON openprose.execution(parent_id) WHERE parent_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_execution_metadata_gin ON openprose.execution USING GIN (metadata jsonb_path_ops);
CREATE INDEX IF NOT EXISTS idx_bindings_run_id ON openprose.bindings(run_id);
CREATE INDEX IF NOT EXISTS idx_bindings_execution_id ON openprose.bindings(execution_id) WHERE execution_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_agents_run_id ON openprose.agents(run_id) WHERE run_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_agents_project_scoped ON openprose.agents(name) WHERE run_id IS NULL;
CREATE INDEX IF NOT EXISTS idx_agent_segments_lookup ON openprose.agent_segments(agent_name, run_id);
```

### 模式约定 (Schema Conventions)

- **时间戳**：使用 `TIMESTAMPTZ` 及 `NOW()` (带时区)
- **JSON 字段**：在 `metadata` 列中使用 `JSONB` 存储结构化数据 (可查询、可索引)
- **大型值**：如果绑定值超过约 100KB，写入 `attachments/{name}.md` 并存储路径
- **扩展表**：使用 `x_` 前缀 (例如 `x_metrics`, `x_audit_log`)
- **匿名绑定**：没有显式捕获的 session 使用自动生成的名称：`anon_001`, `anon_002` 等
- **导入绑定**：使用导入别名作为前缀限定作用域：`research.findings`, `research.sources`
- **限定作用域的绑定**：使用 `execution_id` 列——根作用域为 NULL，block 调用为非空值

### 作用域解析查询 (Scope Resolution Query)

对于递归块，绑定被限定在各自的执行帧中。通过向上遍历调用栈来解析变量：

```sql
-- 在运行 '20260116-143052-a7b3c9' 中，从 execution_id 43 开始查找绑定 'result'
WITH RECURSIVE scope_chain AS (
  -- 从当前执行开始
  SELECT id, parent_id FROM openprose.execution WHERE id = 43
  UNION ALL
  -- 向上遍历至父级
  SELECT e.id, e.parent_id
  FROM openprose.execution e
  JOIN scope_chain s ON e.id = s.parent_id
)
SELECT b.* FROM openprose.bindings b
WHERE b.name = 'result'
  AND b.run_id = '20260116-143052-a7b3c9'
  AND (b.execution_id IN (SELECT id FROM scope_chain) OR b.execution_id IS NULL)
ORDER BY
  CASE WHEN b.execution_id IS NULL THEN 1 ELSE 0 END,  -- 优先选择限定作用域的而非根作用域
  b.execution_id DESC NULLS LAST  -- 优先选择更深 (更局部) 的作用域
LIMIT 1;
```

**如果你已知作用域链，可使用更简单的版本**：

```sql
-- 直接查找：检查当前作用域 (43)，然后检查父级 (42)，最后检查根作用域 (NULL)
SELECT * FROM openprose.bindings
WHERE name = 'result'
  AND run_id = '20260116-143052-a7b3c9'
  AND (execution_id = 43 OR execution_id = 42 OR execution_id IS NULL)
ORDER BY execution_id DESC NULLS LAST
LIMIT 1;
```

---

## 数据库交互 (Database Interaction)

VM 和 subagents 都通过 `psql` CLI 进行交互。

### 从 VM 角度

```bash
# 初始化模式
psql "$OPENPROSE_POSTGRES_URL" -f schema.sql

# 注册新运行实例
psql "$OPENPROSE_POSTGRES_URL" -c "
  INSERT INTO openprose.run (id, program_path, program_source, status)
  VALUES ('20260116-143052-a7b3c9', '/path/to/program.prose', 'program source...', 'running')
"

# 更新执行位置
psql "$OPENPROSE_POSTGRES_URL" -c "
  INSERT INTO openprose.execution (run_id, statement_index, statement_text, status, started_at)
  VALUES ('20260116-143052-a7b3c9', 3, 'session \"Research AI safety\"', 'executing', NOW())
"

# 读取绑定
psql "$OPENPROSE_POSTGRES_URL" -t -A -c "
  SELECT value FROM openprose.bindings WHERE name = 'research' AND run_id = '20260116-143052-a7b3c9'
"

# 检查 parallel 分支状态
psql "$OPENPROSE_POSTGRES_URL" -c "
  SELECT metadata->>'branch' AS branch, status FROM openprose.execution
  WHERE run_id = '20260116-143052-a7b3c9' AND metadata->>'parallel_id' = 'p1'
"
```

### 从 Subagents 角度

VM 在启动 subagent 时提供数据库连接和指令：

**根作用域 (Root scope)**：

```
你的输出将存入 PostgreSQL 状态。

| 属性 | 值 |
|----------|-------|
| 连接串 | `postgresql://user:***@host:5432/db` |
| 模式 (Schema) | `openprose` |
| 运行 ID | `20260116-143052-a7b3c9` |
| 绑定名称 | `research` |
| 执行 ID | (根作用域) |

完成后，写入你的输出：

psql "$OPENPROSE_POSTGRES_URL" -c "
  INSERT INTO openprose.bindings (name, run_id, execution_id, kind, value, source_statement)
  VALUES (
    'research',
    '20260116-143052-a7b3c9',
    NULL,  -- 根作用域
    'let',
    E'AI safety research covers alignment, robustness...',
    'let research = session: researcher'
  )
  ON CONFLICT (name, run_id, COALESCE(execution_id, -1)) DO UPDATE
  SET value = EXCLUDED.value, updated_at = NOW()
"
```

**block 调用内部 (包含 execution_id)**：

```
你的输出将存入 PostgreSQL 状态。

| 属性 | 值 |
|----------|-------|
| 连接串 | `postgresql://user:***@host:5432/db` |
| 模式 (Schema) | `openprose` |
| 运行 ID | `20260116-143052-a7b3c9` |
| 绑定名称 | `result` |
| 执行 ID | `43` |
| Block 名称 | `process` |
| 深度 | `3` |

完成后，写入你的输出：

psql "$OPENPROSE_POSTGRES_URL" -c "
  INSERT INTO openprose.bindings (name, run_id, execution_id, kind, value, source_statement)
  VALUES (
    'result',
    '20260116-143052-a7b3c9',
    43,  -- 限定在此执行作用域
    'let',
    E'Processed chunk into 3 sub-parts...',
    'let result = session \"Process chunk\"'
  )
  ON CONFLICT (name, run_id, COALESCE(execution_id, -1)) DO UPDATE
  SET value = EXCLUDED.value, updated_at = NOW()
"
```

对于持久化 agents (执行范围)：

```
你的记忆存储在数据库中：

读取当前状态：
  psql "$OPENPROSE_POSTGRES_URL" -t -A -c "SELECT memory FROM openprose.agents WHERE name = 'captain' AND run_id = '20260116-143052-a7b3c9'"

完成后更新：
  psql "$OPENPROSE_POSTGRES_URL" -c "UPDATE openprose.agents SET memory = '...', updated_at = NOW() WHERE name = 'captain' AND run_id = '20260116-143052-a7b3c9'"

记录此片段：
  psql "$OPENPROSE_POSTGRES_URL" -c "INSERT INTO openprose.agent_segments (agent_name, run_id, segment_number, prompt, summary) VALUES ('captain', '20260116-143052-a7b3c9', 3, '...', '...')"
```

对于项目级 agents，在查询中使用 `run_id IS NULL`：

```sql
-- 读取项目级 Agent 记忆
SELECT memory FROM openprose.agents WHERE name = 'advisor' AND run_id IS NULL;

-- 更新项目级 Agent 记忆
UPDATE openprose.agents SET memory = '...' WHERE name = 'advisor' AND run_id IS NULL;
```

---

## 主线程中的上下文保存 (Context Preservation in Main Thread)

**这一点至关重要**。数据库用于持久化和协调，但 VM 仍必须维持对话上下文。

### VM 必须叙述的内容

即使使用 PostgreSQL 状态，VM 也应在对话中叙述关键事件：

```
[位置] 语句 3: let research = session: researcher
   正在启动 session，输出将写入状态数据库
   [Task 工具调用]
[成功] Session 已完成，绑定已写入数据库
[绑定] research = <已存储至 openprose.bindings>
```

### 为什么两者都需要？

| 目的 | 机制 |
|---------|-----------|
| **工作记忆** | 对话叙述 (VM 在不重新查询的情况下“记得”的内容) |
| **持久状态** | PostgreSQL 数据库 (克服上下文限制，支持恢复执行) |
| **Subagent 协调** | PostgreSQL 数据库 (共享访问点) |
| **调试/检查** | PostgreSQL 数据库 (可查询的历史记录) |

叙述是 VM 执行的“心理模型”。数据库是用于恢复和检查的“真理来源”。

---

## 并行执行 (Parallel Execution)

对于 parallel 块，VM 使用 `metadata` JSONB 字段跟踪分支。**只有 VM 向 `execution` 表执行写入。**

```sql
-- VM 标记 parallel 启动
INSERT INTO openprose.execution (run_id, statement_index, statement_text, status, started_at, metadata)
VALUES ('20260116-143052-a7b3c9', 5, 'parallel:', 'executing', NOW(),
  '{"parallel_id": "p1", "strategy": "all", "branches": ["a", "b", "c"]}'::jsonb)
RETURNING id;  -- 保存为 parent_id (例如 42)

-- VM 为每个分支创建执行记录
INSERT INTO openprose.execution (run_id, statement_index, statement_text, status, started_at, parent_id, metadata)
VALUES
  ('20260116-143052-a7b3c9', 6, 'a = session "Task A"', 'executing', NOW(), 42, '{"parallel_id": "p1", "branch": "a"}'::jsonb),
  ('20260116-143052-a7b3c9', 7, 'b = session "Task B"', 'executing', NOW(), 42, '{"parallel_id": "p1", "branch": "b"}'::jsonb),
  ('20260116-143052-a7b3c9', 8, 'c = session "Task C"', 'executing', NOW(), 42, '{"parallel_id": "p1", "branch": "c"}'::jsonb);

-- Subagents 将各自输出写入 bindings 表 (参见“从 Subagents 角度”章节)
-- Task 工具通知 VM 任务已完成

-- VM 在 Task 返回后标记分支完成
UPDATE openprose.execution SET status = 'completed', completed_at = NOW()
WHERE run_id = '20260116-143052-a7b3c9' AND metadata->>'parallel_id' = 'p1' AND metadata->>'branch' = 'a';

-- VM 检查是否所有分支均已完成
SELECT COUNT(*) AS pending FROM openprose.execution
WHERE run_id = '20260116-143052-a7b3c9'
  AND metadata->>'parallel_id' = 'p1'
  AND parent_id IS NOT NULL
  AND status NOT IN ('completed', 'failed', 'skipped');
```

### 并发优势 (The Concurrency Advantage)

每个 subagent 都向 `openprose.bindings` 中的不同行执行写入。PostgreSQL 的行级锁意味着**无阻塞**：

```
SQLite (表级锁):
  分支 1 写入 -------|
                       分支 2 等待 ------|
                                            分支 3 等待 -----|
  总耗时：3 * 写入耗时 (序列化执行)

PostgreSQL (行级锁):
  分支 1 写入  --|
  分支 2 写入  --|  (并发执行)
  分支 3 写入  --|
  总耗时：约 1 * 写入耗时 (并行执行)
```

---

## Loop 跟踪

```sql
-- Loop 元数据用于跟踪迭代状态
INSERT INTO openprose.execution (run_id, statement_index, statement_text, status, started_at, metadata)
VALUES ('20260116-143052-a7b3c9', 10, 'loop until **analysis complete** (max: 5):', 'executing', NOW(),
  '{"loop_id": "l1", "max_iterations": 5, "current_iteration": 0, "condition": "**analysis complete**"}'::jsonb);

-- 更新迭代进度
UPDATE openprose.execution
SET metadata = jsonb_set(metadata, '{current_iteration}', '2')
WHERE run_id = '20260116-143052-a7b3c9' AND metadata->>'loop_id' = 'l1' AND parent_id IS NULL;
```

---

## 错误处理 (Error Handling)

```sql
-- 记录失败
UPDATE openprose.execution
SET status = 'failed',
    error_message = 'Connection timeout after 30s',
    completed_at = NOW()
WHERE id = 15;

-- 在元数据中跟踪 retry 尝试
UPDATE openprose.execution
SET metadata = jsonb_set(jsonb_set(metadata, '{retry_attempt}', '2'), '{max_retries}', '3')
WHERE id = 15;

-- 将运行实例标记为失败
UPDATE openprose.run SET status = 'failed' WHERE id = '20260116-143052-a7b3c9';
```

---

## 项目级和用户级 Agents (Project-Scoped and User-Scoped Agents)

执行级 agents (默认) 使用特定的 `run_id`。**项目级 agents** (`persist: project`) 和 **用户级 agents** (`persist: user`) 使用 `run_id IS NULL` 并在多次运行中持久化。

对于用户级 agents，VM 维持一个单独的连接，或使用命名约定将其与项目级 agents 区分开。一种方法是在同一个数据库中为用户级 agent 名称添加 `__user__` 前缀，或者使用通过 `OPENPROSE_POSTGRES_USER_URL` 配置的独立用户级数据库。

### run_id 方法

主键中的 `COALESCE` 技巧允许在同一个表中存储两种作用域：

```sql
PRIMARY KEY (name, COALESCE(run_id, '__project__'))
```

这意味着：
- `name='advisor', run_id=NULL` 的主键为 `('advisor', '__project__')`
- `name='advisor', run_id='20260116-143052-a7b3c9'` 的主键为 `('advisor', '20260116-143052-a7b3c9')`

同一个 agent 名称可以同时作为项目级和执行级存在，而不会发生冲突。

### 查询模式 (Query Patterns)

| 作用域 | 查询语句 |
|-------|-------|
| 执行级 (Execution-scoped) | `WHERE name = 'captain' AND run_id = '{RUN_ID}'` |
| 项目级 (Project-scoped) | `WHERE name = 'advisor' AND run_id IS NULL` |

### 项目级记忆指南 (Project-Scoped Memory Guidelines)

项目级 agents 应存储可泛化的、累积性的知识：

**建议存储**：用户偏好、项目背景、学到的模式、决策理由
**不建议存储**：运行特定的细节、时间敏感信息、海量数据

### Agent 清理 (Agent Cleanup)

- **执行级**：可在运行完成或保留期结束后删除
- **项目级**：仅在用户明确要求时删除

```sql
-- 删除已完成运行实例的执行级 agents
DELETE FROM openprose.agents WHERE run_id = '20260116-143052-a7b3c9';

-- 删除特定的项目级 agent (用户发起)
DELETE FROM openprose.agents WHERE name = 'old_advisor' AND run_id IS NULL;
```

---

## 大型输出 (Large Outputs)

当绑定值过大，不适合在数据库中存储时 (>100KB)：

1. 将内容写入 `attachments/{binding_name}.md`
2. 在 `attachment_path` 列存储路径
3. 在 `value` 中保留摘要

```sql
INSERT INTO openprose.bindings (name, run_id, kind, value, attachment_path, source_statement)
VALUES (
  'full_report',
  '20260116-143052-a7b3c9',
  'let',
  'Full analysis report (847KB) - see attachment',
  'attachments/full_report.md',
  'let full_report = session "Generate comprehensive report"'
)
ON CONFLICT (name, run_id) DO UPDATE
SET value = EXCLUDED.value, attachment_path = EXCLUDED.attachment_path, updated_at = NOW();
```

---

## 恢复执行 (Resuming Execution)

要恢复中断的运行：

```sql
-- 查找当前位置
SELECT statement_index, statement_text, status
FROM openprose.execution
WHERE run_id = '20260116-143052-a7b3c9' AND status = 'executing'
ORDER BY id DESC LIMIT 1;

-- 获取所有已完成的绑定
SELECT name, kind, value, attachment_path FROM openprose.bindings
WHERE run_id = '20260116-143052-a7b3c9';

-- 获取 Agent 记忆状态
SELECT name, scope, memory FROM openprose.agents
WHERE run_id = '20260116-143052-a7b3c9' OR run_id IS NULL;

-- 检查 parallel 块状态
SELECT metadata->>'branch' AS branch, status
FROM openprose.execution
WHERE run_id = '20260116-143052-a7b3c9'
  AND metadata->>'parallel_id' IS NOT NULL
  AND parent_id IS NOT NULL;
```

---

## 灵活性鼓励 (Flexibility Encouragement)

PostgreSQL 状态在设计上保持了**高度灵活性**。核心模式只是起点。鼓励你：

- 按需向现有表**添加列**
- **创建扩展表** (使用 `x_` 前缀)
- **存储自定义指标** (计时、令牌数、模型信息)
- 为你的查询模式**构建索引**
- 对半结构化数据查询使用 **JSONB 操作符**

扩展示例：

```sql
-- 自定义指标表
CREATE TABLE IF NOT EXISTS openprose.x_metrics (
    id SERIAL PRIMARY KEY,
    run_id TEXT REFERENCES openprose.run(id) ON DELETE CASCADE,
    execution_id INTEGER REFERENCES openprose.execution(id) ON DELETE CASCADE,
    metric_name TEXT NOT NULL,
    metric_value NUMERIC,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata JSONB DEFAULT '{}'::jsonb
);

-- 添加自定义列
ALTER TABLE openprose.bindings ADD COLUMN IF NOT EXISTS token_count INTEGER;

-- 为常用查询创建并发索引
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_bindings_created ON openprose.bindings(created_at);
```

数据库就是你的工作空间。尽情使用它。

---

## 与其他模式的比较 (Comparison with Other Modes)

| 特性 | filesystem.md | in-context.md | sqlite.md | postgres.md |
|--------|---------------|---------------|-----------|-------------|
| **状态位置** | `.prose/runs/{id}/` 文件 | 对话历史 | `.prose/runs/{id}/state.db` | PostgreSQL 数据库 |
| **可查询性** | 通过读取文件 | 否 | 是 (SQL) | **是 (SQL)** |
| **原子更新** | 否 | 不适用 | 是 (事务) | **是 (ACID)** |
| **并发写入** | 是 (不同文件) | 不适用 | **否 (表级锁)** | **是 (行级锁)** |
| **网络访问** | 否 | 否 | 否 | **是** |
| **团队协作** | 通过文件同步 | 否 | 通过文件同步 | **是** |
| **模式灵活性** | 固定文件结构 | 不适用 | 灵活 | **极度灵活 (JSONB)** |
| **恢复执行** | 读取 state.md | 重新阅读对话 | 查询数据库 | 查询数据库 |
| **复杂度上限** | 高 | 低 (<30 条语句) | 高 | **极高** |
| **依赖项** | 无 | 无 | sqlite3 CLI | psql CLI + PostgreSQL |
| **设置摩擦** | 零 | 零 | 低 | 中-高 |
| **状态** | 稳定 | 稳定 | 实验性 | **实验性** |

---

## 总结 (Summary)

PostgreSQL 状态管理：

1. 所有运行实例使用**共享的 PostgreSQL 数据库**
2. 通过行级锁提供**真正的并发写入**
3. 为外部工具和仪表板提供**网络访问**支持
4. 支持在共享运行状态上的**团队协作**
5. 允许利用 JSONB 和自定义表进行**灵活的模式演进**
6. 要求安装 **psql CLI** 并运行 PostgreSQL 服务器
7. 目前处于**实验阶段**——预期会有变动

核心契约：VM 管理执行流并启动 subagents；subagents 将自己的输出直接写入数据库。完成信号通过 Task 工具返回发出，而非数据库更新。外部工具可实时查询执行状态。

**PostgreSQL 状态适用于高级用户。** 如果你不需要并发写入、网络访问或团队协作，文件系统或 SQLite 状态会更简单且足够。
