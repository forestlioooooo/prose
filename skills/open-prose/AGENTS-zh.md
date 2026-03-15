# OpenProse 技能知识库

**位置:** `skills/open-prose/`  
**关联文件:** 与 SKILL.md 位于同一目录

## 概述
OpenProse 技能是项目的核心，包含语言虚拟机规范、编译器、示例程序和共享工具。此目录定义了整个 OpenProse 运行时行为。

## 关键文件
| 文件 | 用途 | 何时加载 |
|------|------|----------|
| `SKILL.md` | 技能激活和路由规则 | 用户使用任何 `prose` 命令时 |
| `prose.md` | 虚拟机/解释器规范 | 运行程序时加载（成为 VM） |
| `compiler.md` | 编译器/验证器规范 | 仅当编译或验证时加载 |
| `SOUL.md` | 内存模板 | 用于个人 SOUL.md 记忆 |
| `help.md` | 帮助和常见问题 | `prose help` 时加载 |

## 目录结构
```
open-prose/
├── examples/           # 51个示例程序（01-50 + roadmap）
├── lib/               # 标准库（检查器、分析器、内存工具）
├── common/            # 公共程序（注册表、观察者、发布者等）
├── state/             # 状态管理后端
│   ├── filesystem.md  # 文件系统状态（默认）
│   ├── in-context.md  # 上下文内状态
│   ├── sqlite.md      # SQLite 状态（实验性）
│   └── postgres.md    # PostgreSQL 状态（实验性）
├── guidance/          # 编写指导
│   ├── patterns.md    # 最佳实践
│   └── antipatterns.md # 反模式
├── primitives/        # 原语定义
├── alts/              # 替代方案文档
└── roadmap/           # 路线图示例
```

## 状态管理模式
OpenProse 支持四种状态管理方式：

| 模式 | 适用场景 | 状态位置 |
|------|----------|----------|
| **文件系统** (默认) | 复杂程序、需要恢复、调试 | `.prose/runs/{id}/` 文件 |
| **上下文内** | 简单程序 (<30 条语句)、无需持久化 | 对话历史 |
| **SQLite** (实验性) | 可查询状态、原子事务 | `.prose/runs/{id}/state.db` |
| **PostgreSQL** (实验性) | 真正并发写入、外部集成 | PostgreSQL 数据库 |

**重要安全提示**: PostgreSQL 凭证在 `OPENPROSE_POSTGRES_URL` 中对子代理可见。请使用专用数据库和有限权限用户。

## 示例程序类别
- **01-08**: 基础（hello world、研究、代码审查、调试）
- **09-12**: 代理和技能
- **13-15**: 变量和组合
- **16-19**: 并行执行
- **20-21**: 循环和管道
- **22-23**: 错误处理
- **24-27**: 高级（选择、条件、块、插值）
- **28**: Gas Town（多代理编排）
- **29-31**: 船长椅模式（持久编排器）
- **33-36**: 生产工作流（PR 自动修复、内容管道、功能工厂、错误猎人）
- **37**: The Forge（从零构建浏览器）

## 共享工具
### 标准库 (`lib/`)
- `inspector.prose` - 运行后分析
- `vm-improver.prose` - VM 改进建议
- `program-improver.prose` - 程序改进建议
- `cost-analyzer.prose` - 令牌/成本分析
- `error-forensics.prose` - 错误根本原因分析
- `profiler.prose` - 性能分析
- `user-memory.prose` - 用户范围内存
- `project-memory.prose` - 项目范围内存

### 公共程序 (`common/`) - 星座共享
- `registry.prose` - 程序注册表
- `seeker.prose` - 搜索执行/程序
- `observatory.prose` - 被动观察/合成
- `publisher.prose` - 发布到注册表
- `sentinel.prose` - 安全/监控程序
- `swarm.prose` - 扇出编排
- 以及其他 12+ 个公共程序

## 执行流程
1. **激活技能**: 用户使用 `prose` 命令 → 加载 `SKILL.md`
2. **加载 VM**: 运行程序时加载 `prose.md` + 状态后端
3. **解析程序**: 根据 `compiler.md` 验证语法
4. **执行会话**: 每个 `session` 语句触发 Task 工具调用
5. **状态跟踪**: 使用叙述协议跟踪执行（[位置]、[绑定]、[成功]等）
6. **智能评估**: `**...**` 标记需要 AI 判断

## 重要约定
- **单一技能**: 只有 `open-prose` 一个技能，没有单独的 `prose-run`、`prose-compile` 等技能
- **文档位置**: OpenProse 文档文件与此 SKILL.md 位于同一目录，不在用户工作区搜索
- **远程程序**: 支持从 URL 或注册表引用运行 `.prose` 程序
- **更新迁移**: `prose update` 检查遗留文件结构并迁移到当前格式

## 注意事项
- **编译时**: `compiler.md` 文件很大，仅当用户明确请求编译或验证时加载
- **状态切换**: 默认加载 `state/filesystem.md`；用户可通过 `--in-context`、`--state=sqlite` 或 `--state=postgres` 切换
- **PostgreSQL 要求**: 需要 `psql` CLI 和运行中的 PostgreSQL 服务器，配置在 `.prose/.env` 中