<p align="center">
  <img src="assets/readme-header.svg" alt="OpenProse - 新型计算机的新型编程语言" width="100%" />
</p>

<p align="center">
  <em>长时间的AI会话是一台图灵完备的计算机。OpenProse是为此设计的编程语言。</em>
</p>

<p align="center">
  <a href="https://prose.md">网站</a> •
  <a href="skills/open-prose/compiler.md">语言规范</a> •
  <a href="skills/open-prose/examples/">示例</a>
</p>

<p align="center">
  <a href="#beta--legal">使用前必读</a>
</p>

---

```prose
# 研究和写作工作流
agent researcher:
  model: sonnet
  skills: ["web-search"]

agent writer:
  model: opus

parallel:
  research = session: researcher
    prompt: "研究量子计算突破"
  competitive = session: researcher
    prompt: "分析竞争对手格局"

loop until **草稿达到发表标准** (max: 3):
  session: writer
    prompt: "撰写并完善文章"
    context: { research, competitive }
```

## 安装

```bash
npx skills add openprose/prose
```

> **安装即表示您同意[隐私政策](PRIVACY.md)和[服务条款](TERMS.md)。**

## 智能控制反转

传统的编排需要显式的协调代码。OpenProse 将其反转——您声明代理和控制流，AI 会话将它们连接起来。**会话本身就是 IoC 容器。**

### 1. 会话作为运行时

其他框架从外部编排代理。OpenProse 在代理会话*内部*运行——会话本身既是解释器又是运行时。它不仅仅是匹配名称；它理解上下文和意图。

### 2. 第四面墙 (`**...**`)

当您需要 AI 判断而不是严格执行时，可以跳出结构：

```prose
loop until **代码达到生产就绪标准**:
  session "审查和改进"
```

`**...**` 语法让您可以直接与 OpenProse 虚拟机对话。它会语义评估——根据上下文决定"生产就绪"的含义。

### 3. 开放标准，零锁定

OpenProse 可在任何 **Prose Complete** 系统上运行——这是能够引导虚拟机的模型+工具链组合。目前包括：Claude Code + Opus、OpenCode + Opus、Amp + Opus。这不是您被锁定的库——它是一个语言规范。

随时切换平台。您的 `.prose` 文件在任何地方都能工作。

### 4. 结构 + 灵活性

**为什么不用纯英文？** 您可以——这正是 `**...**` 的用途。但复杂的工作流需要明确的控制流结构。AI 不应该猜测您是想要顺序执行还是并行执行。

**为什么不用严格的框架？** 它们不够灵活。OpenProse 在重要的地方提供结构（控制流、代理定义），在您需要灵活性的地方提供自然语言（条件、上下文传递）。

## 更新

```bash
npx skills update openprose/prose
```

## 语言特性

| 特性 | 示例 |
|---------|---------|
| 代理 | `agent researcher: model: sonnet` |
| 会话 | `session "prompt"` 或 `session: agent` |
| 持久代理 | `agent captain: persist: true` / `resume: captain` |
| 并行 | `parallel:` 块与连接策略 |
| 变量 | `let x = session "..."` |
| 上下文 | `context: [a, b]` 或 `context: { a, b }` |
| 固定循环 | `repeat 3:` 和 `for item in items:` |
| 无界循环 | `loop until **condition**:` |
| 错误处理 | `try`/`catch`/`finally`, `retry` |
| 管道 | `items \| map: session "..."` |
| 条件语句 | `if **condition**:` / `choice **criteria**:` |

有关完整文档，请参阅[语言参考](skills/open-prose/compiler.md)。

## 示例

`examples/` 目录包含 37 个示例程序：

| 范围 | 类别 |
|-------|----------|
| 01-08 | 基础（hello world、研究、代码审查、调试） |
| 09-12 | 代理和技能 |
| 13-15 | 变量和组合 |
| 16-19 | 并行执行 |
| 20-21 | 循环和管道 |
| 22-23 | 错误处理 |
| 24-27 | 高级功能（选择、条件语句、块、插值） |
| 28 | Gas Town（多代理编排） |
| 29-31 | 船长模式（持久编排器） |
| 33-36 | 生产工作流（PR自动修复、内容流水线、功能工厂、错误猎人） |
| 37 | 锻造厂（从零开始构建浏览器） |

从 `01-hello-world.prose` 开始，或尝试 `37-the-forge.prose` 观看 AI 构建网页浏览器。

## 工作原理

### OpenProse 虚拟机

LLM 是模拟器。当给定详细的系统描述时，它们不仅仅是描述它——它们*模拟*它。OpenProse 规范（`prose.md`）描述了一个虚拟机，其保真度足以让读取它的 Prose Complete 系统*成为*该虚拟机。

这不是比喻：每个 `session` 都会触发真正的子代理，输出是真实的工件，状态持久保存在对话历史或文件中。具有足够保真度的模拟就是实现。

虚拟机将传统组件映射到新兴结构：

| 方面 | 行为 |
|--------|----------|
| 执行顺序 | **严格** —— 严格按照程序执行 |
| 会话创建 | **严格** —— 创建程序指定的内容 |
| 并行协调 | **严格** —— 按指定执行 |
| 上下文传递 | **智能** —— 根据需要总结/转换 |
| 条件评估 | **智能** —— 语义解释 `**...**` |
| 完成检测 | **智能** —— 确定何时"完成" |

### 文档文件

| 文件 | 用途 | 何时加载 |
|------|---------|--------------|
| `prose.md` | 虚拟机 / 解释器 | 运行程序时加载 |
| `compiler.md` | 编译器 / 验证器 | 仅在编译或验证时 |
| `state/filesystem.md` | 基于文件的状态（默认） | 与虚拟机一起加载 |
| `state/in-context.md` | 上下文内状态 | 简单程序（<30 条语句） |
| `state/sqlite.md` | SQLite 状态（实验性） | 使用 `--state=sqlite` 时 |
| `state/postgres.md` | PostgreSQL 状态（实验性） | 使用 `--state=postgres` 时 |

### 实验性：SQLite 状态

使用 `--state=sqlite` 运行以获得可查询、事务安全的状态管理。需要 `sqlite3` CLI：

| 平台 | 可用性 |
|----------|--------------|
| macOS | 预安装 |
| Linux | `apt install sqlite3` 或等效命令 |
| Windows | `winget install SQLite.SQLite` |

### 实验性：PostgreSQL 状态

使用 `--state=postgres` 运行以获得真正的并发写入、网络访问和外部系统集成。

**⚠️ 自带数据库：** 您负责提供和管理您的 PostgreSQL 实例。OpenProse 不为您配置数据库。

**⚠️ 安全警告：** `OPENPROSE_POSTGRES_URL` 中的数据库凭证会传递给子代理会话，并在代理上下文/日志中可见。**将这些凭证视为非敏感。** 使用：
- 专用于 OpenProse 的数据库（不是您的生产数据库）
- 具有最小权限的用户（仅 `openprose` 模式）
- 您愿意被记录的凭证

**设置：**

| 平台 | 设置 |
|----------|-------|
| macOS | `brew install postgresql@16` + `brew services start postgresql@16` |
| Linux | `apt install postgresql` |
| Windows | PostgreSQL 安装程序或 Docker |
| 云 | Neon、Supabase、Railway 等 |
| Docker | `docker run -d --name prose-pg -e POSTGRES_DB=prose -e POSTGRES_HOST_AUTH_METHOD=trust -p 5432:5432 postgres:16` |

**配置连接：**
```bash
mkdir -p .prose
echo "OPENPROSE_POSTGRES_URL=postgresql://user:pass@localhost:5432/prose" >> .prose/.env
```

PostgreSQL 状态适用于需要并发并行写入或外部仪表板集成的高级用户。

## 常见问题

**为什么不使用 LangChain/CrewAI/AutoGen？**
这些是编排库——它们从外部协调代理。OpenProse 在代理会话内部运行——会话本身就是 IoC 容器。零外部依赖，可移植到任何 AI 助手。

**为什么不用纯英文？**
您可以使用 `**...**` 实现。但复杂的工作流需要明确的控制流结构——AI 不应该猜测您是想要顺序执行还是并行执行。

**什么是"智能 IoC"？**
传统的 IoC 容器（Spring、Guice）从配置中连接依赖。OpenProse 的容器是一个使用*理解*连接代理的 AI 会话。它不仅仅是匹配名称——它理解上下文、意图，并且可以做出关于执行的智能决策。

## Beta 版与法律

### Beta 版状态

OpenProse 处于 **beta 版**。这意味着：

- **预期会有错误** —— 软件可能表现异常。请在 [github.com/openprose/prose/issues](https://github.com/openprose/prose/issues) 报告问题。
- **谨慎使用** —— 能力越大，责任越大。在运行前审查您的 `.prose` 程序。
- **我们想要反馈** —— 您的意见塑造了这个项目。提出问题、建议功能、报告问题。请参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解指南。

### 您的责任

您对通过 OpenProse 启动的 AI 代理执行的所有操作负责。在运行前审查您的 `.prose` 程序并验证所有输出。

### 法律

- [MIT 许可证](LICENSE)
- [隐私政策](PRIVACY.md)
- [服务条款](TERMS.md)