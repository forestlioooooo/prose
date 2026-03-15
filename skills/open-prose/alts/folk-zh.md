---
role: experimental
summary: |
  OpenProse 的“民间故事” (Folk) 语域——一套文学化/民俗风格的替代关键字集。
  包含奇想、戏剧感，植根于童话与神话。用于与功能性语域进行基准测试。
status: draft
requires: prose.md
---

# OpenProse 民间故事语域 (Folk Register)

> **这是一个皮肤层 (Skin Layer)。** 它要求先加载 `prose.md`。所有的执行语义、状态管理和虚拟机 (VM) 行为都定义在那里。本文件仅提供关键字的翻译映射。

这是 OpenProse 的一种替代语域，更倾向于使用文学、戏剧和民俗术语。功能性语域优先考虑实用性与清晰度；民间故事语域则优先考虑趣味性 (Whimsy) 和叙事流畅性。

## 如何使用

1. 首先加载 `prose.md`（执行语义）
2. 加载本文件（关键字翻译）
3. 在解析 `.prose` 文件时，接受“民间故事”关键字作为功能性关键字的别名
4. 所有执行行为保持一致——仅表面语法发生变化

> **设计约束：** 依然遵循语言原则中“结构化但显而易见”的要求——只是对不同感官偏好的用户而言，显而易见的逻辑不同。

---

## 完整翻译映射表

### 核心结构 (Core Constructs)

| 功能性关键字 (Functional) | 民间故事关键字 (Folk) | 来源说明 | 隐含意义 (Connotation) |
|------------|------|--------|-------------|
| `agent` | `sprite` | 民俗 (Folklore) | 小妖精/精灵：机灵、轻盈、转瞬即逝的灵魂助手 |
| `session` | `scene` | 戏剧 (Theatre) | 场景：行动的时刻，戏剧化的框架 |
| `parallel` | `ensemble` | 戏剧 (Theatre) | 全体：全员同时登台表演 |
| `block` | `act` | 戏剧 (Theatre) | 幕：戏剧行动的可重用单元 |

### 组合与绑定 (Composition & Binding)

| 功能性关键字 (Functional) | 民间故事关键字 (Folk) | 来源说明 | 隐含意义 (Connotation) |
|------------|------|--------|-------------|
| `use` | `summon` | 民俗 (Folklore) | 召唤：从他处唤出 |
| `input` | `given` | 童话 (Fairy tale) | 给定：“给定一把魔法剑……” |
| `output` | `yield` | 农业/魔法 | 产出：法术产生的结果 |
| `let` | `name` | 民俗 (Folklore) | 命名：命名即拥有力量（真实姓名） |
| `const` | `seal` | 中世纪 (Medieval) | 封印：不可更改的、法令上的蜡封 |
| `context` | `bearing` | 纹章学 (Heraldry) | 佩戴：信使所携带的东西 |

### 控制流 (Control Flow)

| 功能性关键字 (Functional) | 民间故事关键字 (Folk) | 来源说明 | 隐含意义 (Connotation) |
|------------|------|--------|-------------|
| `repeat N` | `N times` | 童话 (Fairy tale) | 次：“她呼唤了三次……” |
| `for...in` | `for each...among` | 叙事性 (Narrative) | 略带讲故事的口吻 |
| `loop` | `loop` | — | 已具备诗意，保持不变 |
| `until` | `until` | — | 已经生效，保持不变 |
| `while` | `while` | — | 已经生效，保持不变 |
| `choice` | `crossroads` | 民俗 (Folklore) | 十字路口：在分岔路做出重大抉择 |
| `option` | `path` | 旅程 (Journey) | 路径：选择哪条路走下去 |
| `if` | `when` | 叙事性 (Narrative) | 当：“当月亮升起时……” |
| `elif` | `or when` | 叙事性 (Narrative) | 连续的条件判断 |
| `else` | `otherwise` | 讲故事 (Storytelling) | 自然的叙事转折 |

### 错误处理 (Error Handling)

| 功能性关键字 (Functional) | 民间故事关键字 (Folk) | 来源说明 | 隐含意义 (Connotation) |
|------------|------|--------|-------------|
| `try` | `venture` | 历险 (Adventure) | 尝试带有不确定性的事物 |
| `catch` | `should it fail` | 叙事性 (Narrative) | 条件化的失败处理 |
| `finally` | `ever after` | 童话 (Fairy tale) | 终章：“从此以后……” |
| `throw` | `cry` | 戏剧 (Drama) | 呼救：发出警报或高声呼喊 |
| `retry` | `persist` | 任务 (Quest) | 坚持：面对阻碍不断尝试 |

### 会话属性 (Session Properties)

| 功能性关键字 (Functional) | 民间故事关键字 (Folk) | 来源说明 | 隐含意义 (Connotation) |
|------------|------|--------|-------------|
| `prompt` | `charge` | 骑士精神 (Chivalry) | 职责：授予任务或责任 |
| `model` | `voice` | 戏剧 (Theatre) | 声部：由哪个声音发言 |

### 未改变部分

这些关键字已经具备诗意，或因过于偏向功能性而难以被合理替代：

- `**...**` 自由裁量标记 —— 已体现了“打破第四面墙”的特质
- `loop`, `until`, `while` —— 本身已具备叙事质感
- `map`, `filter`, `reduce`, `pmap` —— 管道操作符，功能性表达即可
- `max` —— 约束修饰符
- `as` —— 别名
- 模型名称：`sonnet`, `opus`, `haiku` —— 本身已具备诗意

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
# 民间故事语域 (Folk)
summon "@alice/research" as research
given topic: "调查主题"

sprite helper:
  voice: sonnet

name findings = scene: helper
  charge: "研究 {topic}"

yield summary = scene "总结"
  bearing: findings
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
# 民间故事语域 (Folk)
ensemble:
  security = scene "检查安全性"
  perf = scene "检查性能"
  style = scene "检查风格"

scene "合成审查结论"
  bearing: { security, perf, style }
```

### 带条件的循环

```prose
# 功能性语域 (Functional)
loop until **代码无 bug** (max: 5):
  session "寻找并修复 bug"
```

```prose
# 民间故事语域 (Folk)
loop until **代码无 bug** (max: 5):
  scene "寻找并修复 bug"
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
# 民间故事语域 (Folk)
venture:
  scene "高风险操作"
should it fail as err:
  scene "处理错误"
    bearing: err
ever after:
  scene "清理"
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
# 民间故事语域 (Folk)
crossroads **严重级别**:
  path "紧急":
    scene "立即上报"
  path "次要":
    scene "记录待办"
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
# 民间故事语域 (Folk)
when **存在安全问题**:
  scene "修复安全"
or when **存在性能问题**:
  scene "优化"
otherwise:
  scene "通过"
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
# 民间故事语域 (Folk)
act review(topic):
  scene "研究 {topic}"
  scene "分析 {topic}"

perform review("量子计算")
```

---

## 采用“民间故事”语域的理由

1. **“OpenProse” 本身极具文学感。** Prose（散文/白话）是一种文学形式——为何不进一步深入呢？
2. **“第四面墙”带有戏剧色彩。** `**...**` 已经在使用戏剧术语了。
3. **传递不同的信号。** 文学化的术语在暗示：“这可不是你通常见到的那种 DSL（领域特定语言）。”
4. **内在一致性。** 一切都汲取自民俗/戏剧/叙事传统。
5. **记忆点深刻。** `sprite`（小精灵）、`scene`（场景）、`crossroads`（十字路口）能让人过目不忘。
6. **模型名称本身就契合。** `sonnet`（十四行诗）、`opus`（作品）、`haiku`（俳句）都是诗歌形式。

## 反对“民间故事”语域的理由

1. **需要一定的文化底蕴。** 并不是每个人都熟悉民俗典故。
2. **更难进行搜索。** “OpenProse summon” 肯定不如 “OpenProse import” 好搜。
3. **可能显得过于考究。** 有些用户只想要纯粹实用的工具。
4. **翻译/转换开销。** 用户必须在头脑中将其映射回熟悉的编程概念。

---

## 曾考虑的替代方案

### 关于 `sprite` (ephemeral agent)

| 关键字 | 来源 | 被否决原因 |
|---------|--------|------------------|
| `spark` | 英语 | 尚可，但民俗感不足 |
| `wisp` | 英语 | 过于虚无缥缈 |
| `herald` | 英语 | 更像信使而非实操者 |
| `courier` | 法语 | 不错的功能性方案，但不够文学 |
| `envoy` | 法语 | 过于正式，带外交色彩 |

### 关于 `shade` (持久性代理，若实现)

| 关键字 | 来源 | 被否决原因 |
|---------|--------|------------------|
| `daemon` | 希腊语/Unix | 带有 Unix “始终运行”的含义 |
| `oracle` | 希腊语 | 读起来过于偏向“只读”感 |
| `spirit` | 拉丁语 | 与 `sprite` 太接近 |
| `specter` | 拉丁语 | 带有负面/惊悚色彩 |
| `genius` | 罗马 | 含义已被过度占用（指天才） |

### 关于 `ensemble` (parallel)

| 关键字 | 来源 | 被否决原因 |
|---------|--------|------------------|
| `chorus` | 希腊语 | 指合唱，即全员异口同声，而不是各司其职 |
| `troupe` | 法语 | 不错的替代方案，但清晰度稍低 |
| `company` | 戏剧 | 含义已被过度占用（指公司） |

### 关于 `crossroads` (choice)

| 关键字 | 来源 | 被否决原因 |
|---------|--------|------------------|
| `fork` | 路径 | 过于技术化（Git fork） |
| `branch` | 树 | 也过于技术化 |
| `divergence` | 拉丁语 | 过于抽象 |

---

## 结论 (Verdict)

本文件被保留用于与功能性语域进行基准测试。虽然功能性语域仍是主流，但“民间故事”语域在以下方面提供了有趣的参考：

1. **易学性** —— 哪种方案对新手更友好？
2. **记忆点** —— 哪种方案更容易记住？
3. **错误率** —— 哪种方案导致的错误更少？
4. **偏好度** —— 用户到底更喜欢哪一个？

未来的实验可以同时提供这两种语域并衡量其实际效果。
