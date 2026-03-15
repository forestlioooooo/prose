---
role: experimental
summary: |
  OpenProse 的“博尔赫斯” (Borges) 语域——一套学术性/形而上学的替代关键字集。
  包含迷宫 (Labyrinths)、做梦者 (Dreamers)、分岔路口 (Forking paths) 和无限图书馆 (Infinite libraries)。用于与功能性语域进行基准测试。
status: draft
requires: prose.md
---

# OpenProse 博尔赫斯语域 (Borges Register)

> **这是一个皮肤层 (Skin Layer)。** 它要求先加载 `prose.md`。所有的执行语义、状态管理和虚拟机 (VM) 行为都定义在那里。本文件仅提供关键字的翻译映射。

这是 OpenProse 的一种替代语域，汲取了豪尔赫·路易斯·博尔赫斯 (Jorge Luis Borges) 的作品精髓。如果说功能性语域是实用主义的，民间故事语域是异想天开的，那么博尔赫斯语域则是学术性且形而上学的——一切读起来都像是引用自某本虚构的百科全书。

## 如何使用

1. 首先加载 `prose.md`（执行语义）
2. 加载本文件（关键字翻译）
3. 在解析 `.prose` 文件时，接受“博尔赫斯”关键字作为功能性关键字的别名
4. 所有执行行为保持一致——仅表面语法发生变化

> **设计约束：** 依然遵循语言原则中“结构化但显而易见”的要求——只是通过博尔赫斯式的视角来实现这种显而易见性。

---

## 完整翻译映射表

### 核心结构 (Core Constructs)

| 功能性关键字 (Functional) | 博尔赫斯关键字 (Borges) | 参考说明 |
|------------|--------|-----------|
| `agent` | `dreamer` | 选自《环形废墟》(The Circular Ruins) —— 那些通过梦境将世界带入现实的做梦者 |
| `session` | `dream` | 每次执行都是做梦者脑中的一场梦 |
| `parallel` | `forking` | 选自《小径分岔的花园》(The Garden of Forking Paths) —— 分支的时间线 |
| `block` | `chapter` | 书中之书，自引用结构 |

### 组合与绑定 (Composition & Binding)

| 功能性关键字 (Functional) | 博尔赫斯关键字 (Borges) | 参考说明 |
|------------|--------|-----------|
| `use` | `retrieve` | 选自《巴别图书馆》(The Library of Babel) —— 从无限的书中检阅/取回 |
| `input` | `axiom` | 公理：给定的前提（体现了博尔赫斯学术化/数学化的语调） |
| `output` | `theorem` | 定理：从公理推导出的结果 |
| `let` | `inscribe` | 刻下：书写即存在 |
| `const` | `zahir` | 选自《扎伊尔》(The Zahir) —— 无法忘却、不可更改、在思想中固定的事物 |
| `context` | `memory` | 选自《博闻强记的富内斯》(Funes the Memorious) —— 完美且全量的回忆 |

### 控制流 (Control Flow)

| 功能性关键字 (Functional) | 博尔赫斯关键字 (Borges) | 参考说明 |
|------------|--------|-----------|
| `repeat N` | `N mirrors` | 镜像：面对面摆放的无限倒影 |
| `for...in` | `for each...within` | 略具博尔赫斯风格的介词 |
| `loop` | `labyrinth` | 迷宫：不断折回自身的迷径 |
| `until` | `until` | 未改变 |
| `while` | `while` | 未改变 |
| `choice` | `bifurcation` | 分叉：路径的分裂 |
| `option` | `branch` | 分支：分道扬镳的时间分支 |
| `if` | `should` | 学术性条件判断 |
| `elif` | `or should` | 连续的条件判断 |
| `else` | `otherwise` | 自然的替代选项 |

### 错误处理 (Error Handling)

| 功能性关键字 (Functional) | 博尔赫斯关键字 (Borges) | 参考说明 |
|------------|--------|-----------|
| `try` | `venture` | 历险：进入迷宫 |
| `catch` | `lest` | “以免：以免它失败……”（古雅、学术的表达方式） |
| `finally` | `ultimately` | 终极：不可避免的结论 |
| `throw` | `shatter` | 破碎：打破镜子，终结梦境 |
| `retry` | `recur` | 递归：无限倒退，再次尝试 |

### 会话属性 (Session Properties)

| 功能性关键字 (Functional) | 博尔赫斯关键字 (Borges) | 参考说明 |
|------------|--------|-----------|
| `prompt` | `query` | 检索：向图书馆提出查询 |
| `model` | `author` | 作者：哪位作家书写了这场梦 |

### 未改变部分

这些关键字已经生效，或者难以进行合理的替代：

- `**...**` 自由裁量标记 —— 已经体现了“打破第四面墙”的特质
- `until`, `while` —— 已经生效
- `map`, `filter`, `reduce`, `pmap` —— 管道操作符
- `max` —— 约束修饰符
- `as` —— 别名
- 模型名称：`sonnet`, `opus`, `haiku` —— 本身已具备文学色彩

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
# 博尔赫斯语域 (Borges)
retrieve "@alice/research" as research
axiom topic: "调查主题"

dreamer helper:
  author: sonnet

inscribe findings = dream: helper
  query: "研究 {topic}"

theorem summary = dream "总结"
  memory: findings
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
# 博尔赫斯语域 (Borges)
forking:
  security = dream "检查安全性"
  perf = dream "检查性能"
  style = dream "检查风格"

dream "合成审查结论"
  memory: { security, perf, style }
```

### 带条件的循环

```prose
# 功能性语域 (Functional)
loop until **代码无 bug** (max: 5):
  session "寻找并修复 bug"
```

```prose
# 博尔赫斯语域 (Borges)
labyrinth until **代码无 bug** (max: 5):
  dream "寻找并修复 bug"
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
# 博尔赫斯语域 (Borges)
venture:
  dream "高风险操作"
lest as err:
  dream "处理错误"
    memory: err
ultimately:
  dream "清理"
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
# 博尔赫斯语域 (Borges)
bifurcation **严重级别**:
  branch "紧急":
    dream "立即上报"
  branch "次要":
    dream "记录待办"
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
# 博尔赫斯语域 (Borges)
should **存在安全问题**:
  dream "修复安全"
or should **存在性能问题**:
  dream "优化"
otherwise:
  dream "通过"
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
# 博尔赫斯语域 (Borges)
chapter review(topic):
  dream "研究 {topic}"
  dream "分析 {topic}"

do review("量子计算")
```

### 固定迭代

```prose
# 功能性语域 (Functional)
repeat 3:
  session "生成想法"
```

```prose
# 博尔赫斯语域 (Borges)
3 mirrors:
  dream "生成想法"
```

### 不可变绑定 (Immutable Binding)

```prose
# 功能性语域 (Functional)
const config = { model: "opus", retries: 3 }
```

```prose
# 博尔赫斯语域 (Borges)
zahir config = { author: "opus", recur: 3 }
```

---

## 采用“博尔赫斯”语域的理由

1. **形而上学的共鸣。** AI 会话通过梦境创造子代理，这与《环形废墟》的情节如出一辙。
2. **学术化语气。** `axiom`（公理）和 `theorem`（定理）将程序构建为逻辑推演。
3. **令人难忘的隐喻。** 不可更改的“扎伊尔” (Zahir)。无法逃脱的“迷宫” (Labyrinth)。从中检阅检索的“图书馆” (Library)。
4. **主题的一致性。** 博尔赫斯笔下的无限、递归和分岔的时间，都是计算的核心概念。
5. **文学威望。** 博尔赫斯受众广泛，这些引用能引起许多用户的共鸣。

## 反对“博尔赫斯”语域的理由

1. **需要一定的门槛。** 对于没读过博尔赫斯的读者来说，“扎伊尔” (Zahir) 和“富内斯” (Funes) 略显晦涩。
2. **可能显得自命不凡。** 有时感觉更像是在炫技，而不是在沟通。
3. **翻译/转换开销。** 用户必须在头脑中将 `labyrinth`（迷宫）映射到 `loop`（循环）。
4. **文化特定性。** 相比民间故事/童话原型，其普及性略逊一筹。

---

## 博尔赫斯核心参考

为不熟悉相关素材的读者准备：

| 作品名称 | 使用的概念 | 简介 |
|------|--------------|---------|
| 《环形废墟》(The Circular Ruins) | `dreamer`, `dream` | 一个男人通过梦境创造了另一个男人，却发现自己也是别人的梦中产物 |
| 《小径分岔的花园》(The Garden of Forking Paths) | `forking`, `bifurcation`, `branch` | 一座身为一本书的迷宫；时间在不断分岔，走向不同的未来 |
| 《巴别图书馆》(The Library of Babel) | `retrieve` | 一座包含所有可能书籍的无限图书馆 |
| 《博闻强记的富内斯》(Funes the Memorious) | `memory` | 一个拥有完美记忆、无法忘记任何细节的男人 |
| 《扎伊尔》(The Zahir) | `zahir` | 一个一旦见过就无法被遗忘或忽视的物体 |
| 《阿莱夫》(The Aleph) | (暂未使用) | 空间中的一个点，包含所有其他的点 |
| 《特隆、乌克巴尔、奥比斯·特蒂乌斯》(Tlön, Uqbar, Orbis Tertius) | (暂未使用) | 一个虚构的世界逐渐侵蚀并取代现实世界 |

---

## 曾考虑的替代方案

### 关于 `dreamer` (agent)

| 关键字 | 被否决原因 |
|---------|------------------|
| `author` | 已转而用于 `model` |
| `scribe` | 过于被动，仅负责记录 |
| `librarian` | 角色更偏向馆员而非创造者 |

### 关于 `labyrinth` (loop)

| 关键字 | 被否决原因 |
|---------|------------------|
| `recursion` | 过于技术化 |
| `eternal return` | 过长 |
| `ouroboros` | 属于其他神话体系 |

### 关于 `zahir` (const)

| 关键字 | 被否决原因 |
|---------|------------------|
| `aleph` | 阿莱夫关于全能性，而非不可变性 |
| `fixed` | 过于平白 |
| `eternal` | 过度使用 |

### 关于 `memory` (context)

| 关键字 | 被否决原因 |
|---------|------------------|
| `funes` | 作为独立关键字过于晦涩 |
| `recall` | 听起来像函数调用 |
| `archive` | 更偏向巴别图书馆而非富内斯 |

---

## 结论 (Verdict)

本文件被保留用于与功能性及民间故事语域进行基准测试。博尔赫斯语域提供了一种独特的智性/形而上学风味，可能会引起欣赏文学化计算的用户的共鸣。

潜在的基准测试问题：

1. **易学性** —— 将 `labyrinth` 直观地理解为循环是否自然？
2. **记忆点** —— `zahir` 是否比 `const` 更容易记住？
3. **理解度** —— 用户是否能立即理解 `dreamer`/`dream`？
4. **偏好度** —— 用户最喜欢哪种语域？
5. **错误率** —— 隐喻映射是否会导致逻辑错误？
