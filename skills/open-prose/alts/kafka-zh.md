---
role: experimental
summary: |
  OpenProse 的“卡夫卡” (Kafka) 语域——一套官僚式/荒诞主义的替代关键字集。
  包含办事员、诉讼程序、请愿和法令。用于与功能性语域进行基准测试。
status: draft
requires: prose.md
---

# OpenProse 卡夫卡语域 (Kafka Register)

> **这是一个皮肤层 (Skin Layer)。** 它要求先加载 `prose.md`。所有的执行语义、状态管理和虚拟机 (VM) 行为都定义在那里。本文件仅提供关键字的翻译映射。

这是 OpenProse 的一种替代语域，汲取了弗兰茨·卡夫卡 (Franz Kafka) 的作品精髓——《审判》、《城堡》以及《在流放地》。程序变成了“诉讼程序 (Proceedings)”。代理 (Agent) 变成了“办事员 (Clerks)”。一切皆是流程，且无人确切知晓规则。

## 如何使用

1. 首先加载 `prose.md`（执行语义）
2. 加载本文件（关键字翻译）
3. 在解析 `.prose` 文件时，接受“卡夫卡”关键字作为功能性关键字的别名
4. 所有执行行为保持一致——仅表面语法发生变化

> **设计约束：** 依然遵循语言原则中“结构化但显而易见”的要求——只是通过官僚主义的视角来实现这种显而易见性。（这种讽刺意味是刻意为之。）

---

## 完整翻译映射表

### 核心结构 (Core Constructs)

| 功能性关键字 (Functional) | 卡夫卡关键字 (Kafka) | 参考说明 |
|------------|-------|-----------|
| `agent` | `clerk` | 办事员：官僚机构中的一名小吏 |
| `session` | `proceeding` | 诉讼程序：采取的官方行动 |
| `parallel` | `departments` | 部门：多个官僚机构同时行动 |
| `block` | `regulation` | 条例：成文化的规程 |

### 组合与绑定 (Composition & Binding)

| 功能性关键字 (Functional) | 卡夫卡关键字 (Kafka) | 参考说明 |
|------------|-------|-----------|
| `use` | `requisition` | 征用：从档案库中请求 |
| `input` | `petition` | 请愿：提交审议的事项 |
| `output` | `verdict` | 裁决：官僚机构返回的结果 |
| `let` | `file` | 归档：在系统中记录 |
| `const` | `statute` | 法令：不可更改的法律 |
| `context` | `dossier` | 档案：关于某个案例的累积卷宗 |

### 控制流 (Control Flow)

| 功能性关键字 (Functional) | 卡夫卡关键字 (Kafka) | 参考说明 |
|------------|-------|-----------|
| `repeat N` | `N hearings` | 聆讯：反复在法庭出面 |
| `for...in` | `for each...in the matter of` | 官僚化的迭代表达 |
| `loop` | `appeal` | 上诉：永无止境的再次请愿，流程继续 |
| `until` | `until` | 未改变 |
| `while` | `while` | 未改变 |
| `choice` | `tribunal` | 审判庭：作出判决的地方 |
| `option` | `ruling` | 裁定：一种可能的判定结果 |
| `if` | `in the event that` | 官僚化的条件判断 |
| `elif` | `or in the event that` | 连续的条件判断 |
| `else` | `otherwise` | 默认裁定 |

### 错误处理 (Error Handling)

| 功能性关键字 (Functional) | 卡夫卡关键字 (Kafka) | 参考说明 |
|------------|-------|-----------|
| `try` | `submit` | 呈报：提交处理 |
| `catch` | `should it be denied` | 倘若被拒绝：官僚机构驳回 |
| `finally` | `regardless` | 无论如何：无论结果如何都会发生的事 |
| `throw` | `reject` | 驳回：系统拒绝执行 |
| `retry` | `resubmit` | 再次呈报：重新尝试该流程 |

### 会话属性 (Session Properties)

| 功能性关键字 (Functional) | 卡夫卡关键字 (Kafka) | 参考说明 |
|------------|-------|-----------|
| `prompt` | `directive` | 指示：官方指令 |
| `model` | `authority` | 权威：处于科层制中的哪个级别 |

### 未改变部分

这些关键字已经生效，或者难以进行合理的替代：

- `**...**` 自由裁量标记 —— 体现了官僚机构高深莫测的判断力
- `until`, `while` —— 已经生效
- `map`, `filter`, `reduce`, `pmap` —— 管道操作符
- `max` —— 约束修饰符
- `as` —— 别名
- 模型名称：`sonnet`, `opus`, `haiku` —— 保留（或参见上文的“权威”）

---

## 侧向对比 (Side-by-Side Comparison)

### 简单程序

```prose
# 功能性语域 (Functional)
use "@alice/research" as research
input topic: "调查主题"

agent helper:
  model: sonnet

let findings = session: helper
  prompt: "研究 {topic}"

output summary = session "总结"
  context: findings
```

```prose
# 卡夫卡语域 (Kafka)
requisition "@alice/research" as research
petition topic: "调查主题"

clerk helper:
  authority: sonnet

file findings = proceeding: helper
  directive: "研究 {topic}"

verdict summary = proceeding "总结"
  dossier: findings
```

### 并行执行 (Parallel Execution)

```prose
# 功能性语域 (Functional)
parallel:
  security = session "检查安全性"
  perf = session "检查性能"
  style = session "检查风格"

session "合成审查结论"
  context: { security, perf, style }
```

```prose
# 卡夫卡语域 (Kafka)
departments:
  security = proceeding "检查安全性"
  perf = proceeding "检查性能"
  style = proceeding "检查风格"

proceeding "合成审查结论"
  dossier: { security, perf, style }
```

### 带条件的循环

```prose
# 功能性语域 (Functional)
loop until **代码无 bug** (max: 5):
  session "寻找并修复 bug"
```

```prose
# 卡夫卡语域 (Kafka)
appeal until **代码无 bug** (max: 5):
  proceeding "寻找并修复 bug"
```

### 错误处理

```prose
# 功能性语域 (Functional)
try:
  session "高风险操作"
catch as err:
  session "处理错误"
    context: err
finally:
  session "清理"
```

```prose
# 卡夫卡语域 (Kafka)
submit:
  proceeding "高风险操作"
should it be denied as err:
  proceeding "处理错误"
    dossier: err
regardless:
  proceeding "清理"
```

### 选择块 (Choice Block)

```prose
# 功能性语域 (Functional)
choice **严重级别**:
  option "紧急":
    session "立即上报"
  option "次要":
    session "记录待办"
```

```prose
# 卡夫卡语域 (Kafka)
tribunal **严重级别**:
  ruling "紧急":
    proceeding "立即上报"
  ruling "次要":
    proceeding "记录待办"
```

### 条件判断 (Conditionals)

```prose
# 功能性语域 (Functional)
if **存在安全问题**:
  session "修复安全"
elif **存在性能问题**:
  session "优化"
else:
  session "通过"
```

```prose
# 卡夫卡语域 (Kafka)
in the event that **存在安全问题**:
  proceeding "修复安全"
or in the event that **存在性能问题**:
  proceeding "优化"
otherwise:
  proceeding "通过"
```

### 可重用块 (Reusable Blocks)

```prose
# 功能性语域 (Functional)
block review(topic):
  session "研究 {topic}"
  session "分析 {topic}"

do review("量子计算")
```

```prose
# 卡夫卡语域 (Kafka)
regulation review(topic):
  proceeding "研究 {topic}"
  proceeding "分析 {topic}"

invoke review("量子计算")
```

### 固定迭代

```prose
# 功能性语域 (Functional)
repeat 3:
  session "尝试连接"
```

```prose
# 卡夫卡语域 (Kafka)
3 hearings:
  proceeding "尝试连接"
```

### 不可变绑定 (Immutable Binding)

```prose
# 功能性语域 (Functional)
const config = { model: "opus", retries: 3 }
```

```prose
# 卡夫卡语域 (Kafka)
statute config = { authority: "opus", resubmit: 3 }
```

---

## 采用“卡夫卡”语域的理由

1. **黑色幽默。** 将程序视为官僚机构既有趣又容易引起共鸣。
2. **出奇地贴切。** 软件系统往往就像一个高深莫测的官僚机构。
3. **映射清晰。** 请愿/裁决、归档/卷宗、办事员/诉讼程序等都映射良好。
4. **上诉即循环。** 无休止的上诉过程是重试逻辑 (Retry logic) 的完美隐喻。
5. **文化共鸣。** “卡夫卡式” (Kafkaesque) 是一个被广泛理解的形容词。
6. **自我觉察。** 为编程语言使用卡夫卡语域是在承认其中的荒诞性。

## 反对“卡夫卡”语域的理由

1. **基调阴郁。** 并不是每个人都想让自己的程序感觉像《审判》。
2. **关键字冗长。** “in the event that” 和 “should it be denied” 比较长。
3. **引起焦虑。** 觉得官僚机构令人压抑的用户可能不喜欢。
4. **讽刺意味可能失效。** 某些用户可能会字面上理解并感到反感。

---

## 卡夫卡核心概念

| 术语 | 含义 | 用于替代 |
|------|---------|----------|
| 官僚机构 (The apparatus) | 高深莫测的系统 | 虚拟机 (VM) 本身 |
| K. | 主角，从未被完整命名 | 用户 |
| 审判 (The Trial) | 没有明确规则的过程 | 程序执行 |
| 城堡 (The Castle) | 无法企及的权威 | 更高层的系统 |
| 办事员 (Clerk) | 处理事务的小官吏 | `agent` → `clerk` |
| 诉讼程序 (Proceeding) | 官方行动 | `session` → `proceeding` |
| 卷宗 (Dossier) | 累积的档案 | `context` → `dossier` |

---

## 曾考虑的替代方案

### 关于 `clerk` (agent)

| 关键字 | 被否决原因 |
|---------|------------------|
| `official` | 过于泛泛 |
| `functionary` | 拼写较难 |
| `bureaucrat` | 带有过多贬义 |
| `advocate` | 角色过于积极/有帮助 |

### 关于 `proceeding` (session)

| 关键字 | 被否决原因 |
|---------|------------------|
| `case` | 含义已被过度占用 (Switch case) |
| `hearing` | 已保留给 `repeat N hearings` |
| `trial` | 已用于荷马语域 |
| `process` | 过于技术化 |

### 关于 `departments` (parallel)

| 关键字 | 被否决原因 |
|---------|------------------|
| `bureaus` | 不错的替代方案，但清晰度稍低 |
| `offices` | 过于平庸 |
| `ministries` | 相比卡夫卡式更具奥威尔色彩 |

### 关于 `appeal` (loop)

| 关键字 | 被否决原因 |
|---------|------------------|
| `recourse` | 法律技术色彩过重 |
| `petition` | 已用于 `input` |
| `process` | 过于泛泛 |

---

## 结论 (Verdict)

本文件被保留用于基准测试。卡夫卡语域提供了一种带点黑色幽默、带点自我觉察的框架，它承认了软件系统官僚化的一面。这种讽刺正是其精髓所在。

最适合：

- 对软件复杂性持有幽默感的用户
- 读起来确实像是在处理官僚事务的程序
- 欢迎承认荒诞性的语境

不推荐用于：

- 觉得官僚主义隐喻令人压抑的用户
- 需要诚挚、积极氛围的语境
- 需要让人感到亲切易读的文档

---

## 结语

> “一定是有人诬陷了约瑟夫·K，因为尽管他并没犯什么错，但一天清晨，他还是被捕了。”
> —— 《审判》

在卡夫卡语域中，你的程序就是约瑟夫·K。官僚机构会处理它。无论成败，无人能给出定论。但诉讼程序终将继续。
