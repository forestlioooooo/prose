# 更新日志

OpenProse 的所有重要变更将记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)，
此项目遵循 [语义化版本控制](https://semver.org/spec/v2.0.0.html)。

## [0.8.1] - 2025-01-23

### 变更

- **令牌效率改进**：状态跟踪现在更加紧凑，减少了长时间运行程序期间的上下文使用。仅追加日志替换了冗长的状态文件，紧凑标记替换了冗长的叙述。

## [0.8.0] - 2025-01-23

### 破坏性变更

- **注册表语法简化**：注册表引用不再需要 `@` 前缀。
  - **迁移**：更新您的导入和运行命令：
    - `prose run @irl-danb/habit-miner` 变为 `prose run irl-danb/habit-miner`
    - `use "@alice/research"` 变为 `use "alice/research"`
  - 解析规则：URL直接获取，带有 `/` 的路径解析为 p.prose.md，否则为本地文件

### 新增

- **内存程序**（推荐 sqlite+ 后端）：
  - `user-memory.prose`：跨项目持久个人内存，具有教导/查询/反思模式
  - `project-memory.prose`：项目范围的机构内存，具有摄取/查询/更新/总结模式

- **分析程序**：
  - `cost-analyzer.prose`：令牌使用和成本模式分析，具有单个/比较/趋势范围
  - `calibrator.prose`：验证轻量评估与深度评估的可靠性
  - `error-forensics.prose`：失败运行的根本原因分析

- **改进循环程序**：
  - `vm-improver.prose`：分析检查报告并提出改进 OpenProse 虚拟机的 PR
  - `program-improver.prose`：分析检查报告并提出改进 .prose 源代码的 PR

- **技能安全扫描器 v2**：增强功能包括渐进披露、模型分层（Sonnet 用于清单，Opus 用于深度分析）、具有优雅降级的并行扫描器，以及持久的扫描历史

- **交互式示例**：展示输入原语的新示例
- **系统提示**：添加了系统提示配置

### 移除

- **遥测系统**：移除了所有与遥测相关的代码、配置和文档，包括 USER_ID/SESSION_ID 跟踪和分析端点调用

### 变更

- 用户范围的持久代理现在存储在 `~/.prose/agents/` 中
- 更新了注册表语法变更的文档