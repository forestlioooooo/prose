---
role: experimental
summary: |
  OpenProse 的“天方夜谭” (Arabian Nights) 语域——一套叙事性/嵌套式的替代关键字集。
  包含神灯精灵 (Djinns)、故事中的故事、愿望和誓言。用于与功能性语域进行基准测试。
status: draft
requires: prose.md
---

# OpenProse 天方夜谭语域 (Arabian Nights Register)

> **这是一个皮肤层 (Skin Layer)。** 它要求先加载 `prose.md`。所有的执行语义、状态管理和虚拟机 (VM) 行为都定义在那里。本文件仅提供关键字的翻译映射。

这是 OpenProse 的一种替代语域，汲取了《一千零一夜》 (One Thousand and One Nights) 的元素。程序变成了山鲁佐德 (Scheherazade) 讲述的故事。递归变成了“故事中的故事”。代理 (Agent) 变成了受命效力的精灵 (Djinns)。

## 如何使用

1. 首先加载 `prose.md`（执行语义）
2. 加载本文件（关键字翻译）
3. 在解析 `.prose` 文件时，接受“天方夜谭”关键字作为功能性关键字的别名
4. 所有执行行为保持一致——仅表面语法发生变化

> **设计约束：** 依然遵循语言原则中“结构化但显而易见”的要求——只是通过讲故事的视角来实现这种显而易见性。

---

## 完整翻译映射表

### 核心结构 (Core Constructs)

| 功能性关键字 (Functional) | 天方夜谭关键字 (Nights) | 参考说明 |
|------------|--------|-----------|
| `agent` | `djinn` | 受命效力的神灵，实现愿望 (Grants wishes) |
| `session` | `tale` | 讲述的一段故事，叙事单元 |
| `parallel` | `bazaar` | 巴扎（集市）：众多声音，众多摊位，同时并行 |
| `block` | `frame` | 框架故事：包含其他故事的故事 |

### 组合与绑定 (Composition & Binding)

| 功能性关键字 (Functional) | 天方夜谭关键字 (Nights) | 参考说明 |
|------------|--------|-----------|
| `use` | `conjure` | 召唤：从他处唤来 |
| `input` | `wish` | 愿望：向精灵提出的要求 |
| `output` | `gift` | 礼物：作为回报授予的事物 |
| `let` | `name` | 命名：命名即拥有力量（与民间故事语域相同） |
| `const` | `oath` | 誓言：不可违背、已封存的诺言 |
| `context` | `scroll` | 卷轴：书写并传递的信息 |

### 控制流 (Control Flow)

| 功能性关键字 (Functional) | 天方夜谭关键字 (Nights) | 参考说明 |
|------------|--------|-----------|
| `repeat N` | `N nights` | “连续一千零一夜……” |
| `for...in` | `for each...among` | 在商人之中，在故事之中 |
| `loop` | `telling` | 讲述：讲述在继续 |
| `until` | `until` | 未改变 |
| `while` | `while` | 未改变 |
| `choice` | `crossroads` | 十字路口：故事分叉的地方 |
| `option` | `path` | 路径：故事可能走向的一种方式 |
| `if` | `should` | 叙事性条件判断 |
| `elif` | `or should` | 连续的条件判断 |
| `else` | `otherwise` | 另一种讲述方式 |

### 错误处理 (Error Handling)

| 功能性关键字 (Functional) | 天方夜谭关键字 (Nights) | 参考说明 |
|------------|--------|-----------|
| `try` | `venture` | 历险：踏上旅程 |
| `catch` | `should misfortune strike` | 倘若不幸降临：情节转暗 |
| `finally` | `and so it was` | 终局：必然的结局 |
| `throw` | `curse` | 诅咒：宣判厄运 |
| `retry` | `persist` | 坚持：英雄再次尝试 |

### 会话属性 (Session Properties)

| 功能性关键字 (Functional) | 天方夜谭关键字 (Nights) | 参考说明 |
|------------|--------|-----------|
| `prompt` | `command` | 命令：对精灵下达的指令 |
| `model` | `spirit` | 灵体：由哪位神灵回应 |

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
# 天方夜谭语域 (Nights)
conjure "@alice/research" as research
wish topic: "调查主题"

djinn helper:
  spirit: sonnet

name findings = tale: helper
  command: "研究 {topic}"

gift summary = tale "总结"
  scroll: findings
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
# 天方夜谭语域 (Nights)
bazaar:
  security = tale "检查安全性"
  perf = tale "检查性能"
  style = tale "检查风格"

tale "合成审查结论"
  scroll: { security, perf, style }
```

### 带条件的循环

```prose
# 功能性语域 (Functional)
loop until **代码无 bug** (max: 5):
  session "寻找并修复 bug"
```

```prose
# 天方夜谭语域 (Nights)
telling until **代码无 bug** (max: 5):
  tale "寻找并修复 bug"
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
# 天方夜谭语域 (Nights)
venture:
  tale "高风险操作"
should misfortune strike as err:
  tale "处理错误"
    scroll: err
and so it was:
  tale "清理"
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
# 天方夜谭语域 (Nights)
crossroads **严重级别**:
  path "紧急":
    tale "立即上报"
  path "次要":
    tale "记录待办"
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
# 天方夜谭语域 (Nights)
should **存在安全问题**:
  tale "修复安全"
or should **存在性能问题**:
  tale "优化"
otherwise:
  tale "通过"
```

### 可重用块（框架故事 / Frame Stories）

```prose
# 功能性语域 (Functional)
block review(topic):
  session "研究 {topic}"
  session "分析 {topic}"

do review("量子计算")
```

```prose
# 天方夜谭语域 (Nights)
frame review(topic):
  tale "研究 {topic}"
  tale "分析 {topic}"

tell review("量子计算")
```

### 固定迭代

```prose
# 功能性语域 (Functional)
repeat 1001:
  session "讲一个故事"
```

```prose
# 天方夜谭语域 (Nights)
1001 nights:
  tale "讲一个故事"
```

### 不可变绑定 (Immutable Binding)

```prose
# 功能性语域 (Functional)
const config = { model: "opus", retries: 3 }
```

```prose
# 天方夜谭语域 (Nights)
oath config = { spirit: "opus", persist: 3 }
```

---

## 采用“天方夜谭”语域的理由

1. **框架叙事即递归。** “故事中的故事”完美映射了嵌套的程序调用。
2. **精灵/愿望/礼物。** `agent`/`input`/`output` 的映射极其自然。
3. **文化底蕴深厚。** 《一千零一夜》举世闻名。
4. **巴扎 (Bazaar) 象征并行。** 众多商人，众多摊位，同时活跃——意象生动。
5. **誓言 (Oath) 象征常量。** 不可违背的诺言是不可变性 (Immutability) 的完美隐喻。
6. **“1001 nights”** 作为循环计数令人愉悦。

## 反对“天方夜谭”语域的理由

1. **文化敏感性。** 必须以尊重的态度处理，避免使用东方主义 (Orientalist) 的陈词滥调。
2. **“Djinn” 的发音。** 不熟悉的读者可能会感到困惑（读成 jinn? djinn? 还是 genie?）。
3. **某些映射略显牵强。** 用“巴扎”表示并行虽然生动，但不直接。
4. **“should misfortune strike”** 作为 `catch` 显得过长。

---

## 天方夜谭核心概念

| 术语 | 含义 | 用于替代 |
|------|---------|----------|
| 山鲁佐德 (Scheherazade) | 为了生存而讲述故事的叙述者 | （程序作者） |
| 精灵 (Djinn) | 受命效力的超自然灵体 | `agent` → `djinn` |
| 框架故事 (Frame story) | 包含其他故事的故事 | `block` → `frame` |
| 愿望 (Wish) | 向精灵提出的要求 | `input` → `wish` |
| 誓言 (Oath) | 不可违背的诺言 | `const` → `oath` |
| 巴扎 (Bazaar) | 集市，拥有众多商贩 | `parallel` → `bazaar` |

---

## 曾考虑的替代方案

### 关于 `djinn` (agent)

| 关键字 | 被否决原因 |
|---------|------------------|
| `genie` | 带有迪士尼色彩，文学性较弱 |
| `spirit` | 已用于 `model` |
| `ifrit` | 过于具体（精灵的一种类型） |
| `narrator` | 过于元描述 (Meta)，山鲁佐德才是用户 |

### 关于 `tale` (session)

| 关键字 | 被否决原因 |
|---------|------------------|
| `story` | 不错，但 `tale` 更具文学感 |
| `night` | 已保留给 `repeat N nights` |
| `chapter` | 更偏向西方/小说风格 |

### 关于 `bazaar` (parallel)

| 关键字 | 被否决原因 |
|---------|------------------|
| `caravan` | 带有顺序含义（驼队一个接一个） |
| `chorus` | 希腊传统，并非本语域背景 |
| `souk` | 知名度较低 |

### 关于 `scroll` (context)

| 关键字 | 被否决原因 |
|---------|------------------|
| `letter` | 规模太小/太私人化 |
| `tome` | 规模太大 |
| `message` | 过于平白 |

---

## 结论 (Verdict)

本文件被保留用于基准测试。天方夜谭语域提供了一个讲故事的框架，能自然地映射到递归、嵌套的程序中。精灵/愿望/礼物的三元组尤为优雅。

最适合：

- 具有深度嵌套的程序（故事中的故事）
- 感觉像是在实现愿望的工作流
- 喜欢叙事框架的用户

用于可重用块的 `frame` 关键字特别贴切——山鲁佐德的框架故事包含了一千个传说。
