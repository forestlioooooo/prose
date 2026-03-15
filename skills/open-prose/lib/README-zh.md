# OpenProse 标准库 (OpenProse Standard Library)

OpenProse 自带的本地分析与改进程序。这些是生产级别、经过充分测试的程序，用于评估运行结果、改进代码以及管理内存。

对于参与分布式“星座” (Constellation) 系统的程序，请参见 `common/README.md`。

## 程序列表 (Programs)

### 评估与改进 (Evaluation & Improvement)

| 程序 | 描述 |
|---------|-------------|
| `inspector.prose` | 运行后的分析，用于评估运行时的保真度和任务有效性 |
| `vm-improver.prose` | 分析检查结果并提交 PR 以改进虚拟机 (VM) |
| `program-improver.prose` | 分析检查结果并提交 PR 以改进 .prose 源码 |
| `cost-analyzer.prose` | Token 使用情况和成本模式分析 |
| `calibrator.prose` | 针对深度评估验证轻量级评估的准确性 |
| `error-forensics.prose` | 针对运行失败的任务进行根因分析 |

### 内存 (Memory)

| 程序 | 描述 |
|---------|-------------|
| `user-memory.prose` | 跨项目的持久化个人内存 |
| `project-memory.prose` | 项目范围的组织内存 |

## 改进循环 (The Improvement Loop)

评估程序构成了一个递归的改进循环：

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   运行程序  ──►  检查器(Inspector)  ──►  VM 改进器  ──► PR    │
│        ▲                │                                   │
│        │                ▼                                   │
│        │         程序改进器 ──► PR                            │
│        │                │                                   │
│        └────────────────┘                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

辅助分析：
- **cost-analyzer (成本分析器)** —— 钱花在哪了？寻找优化机会。
- **calibrator (校准器)** —— 廉价的评估是否可以作为昂贵评估的可靠替代？
- **error-forensics (错误取证)** —— 为什么运行失败？进行根因分析。

## 使用方法 (Usage)

```bash
# 检查一个已完成的运行
prose run lib/inspector.prose
# 输入: run_path, depth (light|deep), target (vm|task|all)

# 提出 VM 改进建议
prose run lib/vm-improver.prose
# 输入: inspection_path, prose_repo

# 提出程序改进建议
prose run lib/program-improver.prose
# 输入: inspection_path, run_path

# 分析成本
prose run lib/cost-analyzer.prose
# 输入: run_path, scope (single|compare|trend)

# 验证轻量级 vs 深度评估
prose run lib/calibrator.prose
# 输入: run_paths, sample_size

# 调查失败原因
prose run lib/error-forensics.prose
# 输入: run_path, focus (vm|program|context|external)

# 内存程序（推荐使用 sqlite+ 后端）
prose run lib/user-memory.prose --backend sqlite+
# 输入: mode (teach|query|reflect), content

prose run lib/project-memory.prose --backend sqlite+
# 输入: mode (ingest|query|update|summarize), content
```

## 内存程序 (Memory Programs)

内存程序使用持久化代理 (persistent agents) 来积累知识：

**user-memory (用户内存)** (`persist: user`)
- 学习你在所有项目中的偏好、决策和模式
- 记录错误和吸取的教训
- 根据积累的知识回答问题

**project-memory (项目内存)** (`persist: project`)
- 理解本项目架构和决策
- 跟踪事物为何以现状存在的原因
- 使用项目特定上下文回答问题

两者都推荐使用 `--backend sqlite+` 以获得可靠的持久化。

## 设计原则 (Design Principles)

1. **生产就绪** —— 经过测试，文档齐全，能处理边缘情况。
2. **可组合** —— 可以通过 `use` 在其他程序中导入。
3. **用户范围状态** —— 跨项目工具使用 `persist: user`。
4. **最小依赖** —— 无需外部服务。
5. **清晰契约** —— 输入和输出定义明确。
6. **增量价值** —— 在简单模式下有用，在深度模式下更强大。
