---
role: experimental
summary: |
  OpenProse 的“荷马” (Homeric) 语域——一套史诗/英雄主义风格的替代关键字集。
  包含英雄、试炼、命运和荣耀。用于与功能性语域进行基准测试。
status: draft
requires: prose.md
---

# OpenProse 荷马语域 (Homeric Register)

> **这是一个皮肤层 (Skin Layer)。** 它要求先加载 `prose.md`。所有的执行语义、状态管理和虚拟机 (VM) 行为都定义在那里。本文件仅提供关键字的翻译映射。

这是 OpenProse 的一种替代语域，汲取了希腊史诗——《伊利亚特》、《奥德赛》以及英雄传统的精髓。程序变成了“远征 (Quests)”。代理 (Agent) 变成了“英雄 (Heroes)”。输出变成了“赢取的荣耀 (Glory won)”。

## 如何使用

1. 首先加载 `prose.md`（执行语义）
2. 加载本文件（关键字翻译）
3. 在解析 `.prose` 文件时，接受“荷马”关键字作为功能性关键字的别名
4. 所有执行行为保持一致——仅表面语法发生变化

> **设计约束：** 依然遵循语言原则中“结构化但显而易见”的要求——只是通过史诗般的视角来实现这种显而易见性。

---

## 完整翻译映射表

### 核心结构 (Core Constructs)

| 功能性关键字 (Functional) | 荷马关键字 (Homeric) | 参考说明 |
|------------|---------|-----------|
| `agent` | `hero` | 英雄：行动者，奋斗者 |
| `session` | `trial` | 试炼：每项任务都是一次劳作、一次测试 |
| `parallel` | `host` | 大军：万众一心，如军队般推进 |
| `block` | `book` | 卷：史诗的一个篇章 |

### 组合与绑定 (Composition & Binding)

| 功能性关键字 (Functional) | 荷马关键字 (Homeric) | 参考说明 |
|------------|---------|-----------|
| `use` | `invoke` | 祈求： “缪斯啊，请歌唱……” —— 呼唤 |
| `input` | `omen` | 征兆：神灵的启示，给定的预兆 |
| `output` | `glory` | 荣耀 (Kleos)：赢取的名望，流传后世的事迹 |
| `let` | `decree` | 宣告：宣判命运，出口成真 |
| `const` | `fate` | 命运 (Moira)：不可更改的天命 |
| `context` | `tidings` | 喜讯/消息：由使者或信使传达的情报 |

### 控制流 (Control Flow)

| 功能性关键字 (Functional) | 荷马关键字 (Homeric) | 参考说明 |
|------------|---------|-----------|
| `repeat N` | `N labors` | 劳作：赫拉克勒斯的十二大功绩 |
| `for...in` | `for each...among` | 在大军之中 |
| `loop` | `ordeal` | 苦难：反复的试炼，持续的磨砺 |
| `until` | `until` | 未改变 |
| `while` | `while` | 未改变 |
| `choice` | `crossroads` | 十字路口：命运分歧之处 |
| `option` | `path` | 路径：多条道路中的一条 |
| `if` | `should` | 史诗般的条件判断 |
| `elif` | `or should` | 连续的条件判断 |
| `else` | `otherwise` | 另一种命运 |

### 错误处理 (Error Handling)

| 功能性关键字 (Functional) | 荷马关键字 (Homeric) | 参考说明 |
|------------|---------|-----------|
| `try` | `venture` | 历险：踏上旅程 |
| `catch` | `should ruin come` | 阿特 (Até) ：神遣的毁灭、盲目的灾难 |
| `finally` | `in the end` | 终局：必然的结局 |
| `throw` | `lament` | 哀歌：英雄痛苦的呐喊 |
| `retry` | `persist` | 坚持：不屈不挠，再次尝试 |

### 会话属性 (Session Properties)

| 功能性关键字 (Functional) | 荷马关键字 (Homeric) | 参考说明 |
|------------|---------|-----------|
| `prompt` | `charge` | 使命：授予的远征任务 |
| `model` | `muse` | 缪斯：由哪位女神带来灵感 |

### 未改变部分

这些关键字已经生效，或者难以进行合理的替代：

- `**...**` 自由裁量标记 —— 已经生效
- `until`, `while` —— 已经生效
- `map`, `filter`, `reduce`, `pmap` —— 管道操作符
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
# 荷马语域 (Homeric)
invoke "@alice/research" as research
omen topic: "调查主题"

hero helper:
  muse: sonnet

decree findings = trial: helper
  charge: "研究 {topic}"

glory summary = trial "总结"
  tidings: findings
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
# 荷马语域 (Homeric)
host:
  security = trial "检查安全性"
  perf = trial "检查性能"
  style = trial "检查风格"

trial "合成审查结论"
  tidings: { security, perf, style }
```

### 带条件的循环

```prose
# 功能性语域 (Functional)
loop until **代码无 bug** (max: 5):
  session "寻找并修复 bug"
```

```prose
# 荷马语域 (Homeric)
ordeal until **代码无 bug** (max: 5):
  trial "寻找并修复 bug"
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
# 荷马语域 (Homeric)
venture:
  trial "高风险操作"
should ruin come as err:
  trial "处理错误"
    tidings: err
in the end:
  trial "清理"
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
# 荷马语域 (Homeric)
crossroads **严重级别**:
  path "紧急":
    trial "立即上报"
  path "次要":
    trial "记录待办"
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
# 荷马语域 (Homeric)
should **存在安全问题**:
  trial "修复安全"
or should **存在性能问题**:
  trial "优化"
otherwise:
  trial "通过"
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
# 荷马语域 (Homeric)
book review(topic):
  trial "研究 {topic}"
  trial "分析 {topic}"

do review("量子计算")
```

### 固定迭代

```prose
# 功能性语域 (Functional)
repeat 12:
  session "完成任务"
```

```prose
# 荷马语域 (Homeric)
12 labors:
  trial "完成任务"
```

### 不可变绑定 (Immutable Binding)

```prose
# 功能性语域 (Functional)
const config = { model: "opus", retries: 3 }
```

```prose
# 荷马语域 (Homeric)
fate config = { muse: "opus", persist: 3 }
```

---

## 采用“荷马”语域的理由

1. **普适的认可。** 希腊史诗是西方文学的基石。
2. **英雄化的框架。** 将平淡的任务转化为光辉的试炼。
3. **天然的契合感。** 英雄面对试炼，接收消息，赢得荣耀——完美映射 `agent`/`session`/`output`。
4. **庄重感。** 当你希望程序显得史诗般宏大且意义深远时。
5. **命运与宣告。** 将 `const` 视为“命运” (fate，不可更改)，将 `let` 视为“宣告” (decree，可变但有权威性)，非常直观。

## 反对“荷马”语域的理由

1. **宏大感失调。** 用“十二大劳作”来形容简单的循环可能显得过于夸张。
2. **西方中心主义。** 希腊史诗传统具有特定的文化属性。
3. **词汇受限。** 相比博尔赫斯或民间故事语域，其标志性术语较少。
4. **可能显得滑稽。** 对日常琐事使用英雄化的语言可能产生一种意外的滑稽感 (Bathos)。

---

## 荷马核心概念

| 术语 | 含义 | 用于替代 |
|------|---------|----------|
| 荣耀 (Kleos) | 名望，流传后世的荣耀 | `output` → `glory` |
| 命运 (Moira) | 命定的部分 | `const` → `fate` |
| 阿特 (Até) | 神遣的毁灭，众神降下的盲目 | `catch` → `should ruin come` |
| 回归 (Nostos) | 归途 | (暂未使用，但可用于 `finally`) |
| 宾主之情 (Xenia) | 客人与主人之间的情谊 | (暂未使用) |
| 缪斯 (Muse) | 神圣的灵感 | `model` → `muse` |

---

## 曾考虑的替代方案

### 关于 `hero` (agent)

| 关键字 | 被否决原因 |
|---------|------------------|
| `champion` | 更有中世纪色彩，而非荷马风格 |
| `warrior` | 过于武化，并非所有任务都是战斗 |
| `wanderer` | 过于被动 |

### 关于 `trial` (session)

| 关键字 | 被否决原因 |
|---------|------------------|
| `labor` | 不错，但已保留给 `repeat N labors` |
| `quest` | 更有中世纪/RPG 风格 |
| `task` | 过于平淡 |

### 关于 `host` (parallel)

| 关键字 | 被否决原因 |
|---------|------------------|
| `army` | 军事色彩过于浓厚 |
| `fleet` | 仅适用于航海隐喻 |
| `phalanx` | 过于技术化 |

---

## 结论 (Verdict)

本文件被保留用于基准测试。荷马语域提供了庄重感和英雄化的框架。最适合：

- 感觉像史诗般庞大的程序
- 喜欢古典参考的用户
- 将“荣耀”视为产出的语境

应用于日常琐事时可能会导致不经意的滑稽效果。
