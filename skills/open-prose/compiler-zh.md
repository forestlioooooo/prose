---
role: language-specification
summary: |
  OpenProse 的完整语法规则、验证规则和编译语义。
  在编译、验证或解决歧义语法时阅读此文件。假设
  prose.md 已在上下文中用于执行语义。
see-also:
  - SKILL.md: 激活触发器、入门指南
  - prose.md: 执行语义、如何运行程序
  - state/filesystem.md: 文件系统状态管理（默认）
  - state/in-context.md: 上下文内状态管理（根据请求）
---

# OpenProse 语言参考手册 (OpenProse Language Reference)

OpenProse 是一种用于 AI 会话的编程语言。一个 AI 会话就是一个图灵完备的计算机；本文档提供了关于该语言语法、语义和执行模型的完整文档。

---

## 文档目的：编译器 + 验证器 (Document Purpose: Compiler + Validator)

本文档承担双重角色：

### 作为编译器 (As Compiler)

当被要求“编译” (compile) 一个 `.prose` 文件时，使用此规范来：

1. **解析** (Parse)：根据语法规则解析程序
2. **验证** (Validate)：验证程序是否格式良好且语义有效
3. **转换** (Transform)：将程序转换为“最佳实践”的标准形式：
   - 在适当的情况下展开语法糖 (syntax sugar)
   - 规范化格式和结构
   - 应用优化（例如：提升块定义 [hoisting block definitions]）

### 作为验证器 (As Validator)

验证标准：**一个仅拥有 `prose.md` 的空白代理能否理所当然地理解此程序？**

验证时，检查：

- 语法正确性 (Syntax correctness)：所有构造均符合语法规则
- 语义有效性 (Semantic validity)：引用已解析，类型匹配
- 显而易见性 (Self-evidence)：即使没有此完整规范，程序也是清晰的

如果某个构造存在歧义或不明显，应将其标记或转换为更清晰的形式。

### 何时阅读此文档 (When to Read This Document)

- **请求编译**：完整阅读以应用所有规则
- **请求验证**：完整阅读以检查所有约束
- **遇到歧义语法**：参考特定章节
- **仅进行解释执行**：改用 `prose.md`（更小、更快）

---

## 目录 (Table of Contents)

1. [概述](#overview)
2. [文件格式](#file-format)
3. [注释](#comments)
4. [字符串字面量](#string-literals)
5. [Use 语句（程序组合）](#use-statements-program-composition)
6. [Input 声明](#input-declarations)
7. [Output 绑定](#output-bindings)
8. [程序调用](#program-invocation)
9. [Agent 定义](#agent-definitions)
10. [Session 语句](#session-statement)
11. [Resume 语句](#resume-statement)
12. [变量与上下文](#variables--context)
13. [组合块（Composition Blocks）](#composition-blocks)
14. [并行块（Parallel Blocks）](#parallel-blocks)
15. [固定循环](#fixed-loops)
16. [无界循环](#unbounded-loops)
17. [管道操作](#pipeline-operations)
18. [错误处理](#error-handling)
19. [Choice 块](#choice-blocks)
20. [条件语句](#conditional-statements)
21. [执行模型](#execution-model)
22. [验证规则](#validation-rules)
23. [示例](#examples)
24. [未来特性](#future-features)

---

## 概述 (Overview)

OpenProse 为定义多代理工作流 (multi-agent workflows) 提供了一种声明式语法。程序由顺序执行的语句组成，每个 `session` 语句都会生成一个子代理 (subagent) 来完成任务。

### 设计原则 (Design Principles)

- **模式胜过框架** (Pattern over framework)：最简单的解决方案几乎什么都不是——只是英语的结构
- **显而易见** (Self-evident)：程序应在最少文档的情况下即可理解
- **OpenProse 虚拟机是智能的** (The OpenProse VM is intelligent)：为理解而设计，而非为解析而设计
- **框架无关** (Framework-agnostic)：适用于 Claude Code、OpenCode 和任何未来的代理框架
- **文件即工件** (Files are artifacts)：`.prose` 是可移植的工作单元

### 当前实现状态 (Current Implementation Status)

以下特性已实现：

| 特性 (Feature) | 状态 (Status) | 描述 (Description) |
| :--- | :--- | :--- |
| Comments | 已实现 | `# comment` 语法 |
| Single-line strings | 已实现 | 带转义的 `"string"` |
| Simple session | 已实现 | `session "prompt"` |
| Agent definitions | 已实现 | 带有 model/prompt 属性的 `agent name:` |
| Session with agent | 已实现 | 带有属性覆盖的 `session: agent` |
| Use statements | 已实现 | `use "@handle/slug" as name` |
| Agent skills | 已实现 | `skills: ["skill1", "skill2"]` |
| Agent permissions | 已实现 | 带有规则的 `permissions:` 块 |
| Let binding | 已实现 | `let name = session "..."` |
| Const binding | 已实现 | `const name = session "..."` |
| Variable reassignment | 已实现 | `name = session "..."`（仅限 let） |
| Context property | 已实现 | `context: var` 或 `context: [a, b, c]` |
| do: blocks | 已实现 | 显式的顺序执行块 |
| Inline sequence | 已实现 | `session "A" -> session "B"` |
| Named blocks | 已实现 | 带有 `do name` 调用的 `block name:` |
| Parallel blocks | 已实现 | 用于并发执行的 `parallel:` |
| Named parallel results | 已实现 | `parallel` 内部的 `x = session "..."` |
| Object context | 已实现 | `context: { a, b, c }` 简写 |
| Join strategies | 已实现 | `parallel ("first"):` 或 `parallel ("any"):
  session "尝试 1"
  session "尝试 2"

# 指定数量：等待 2 个成功
parallel ("any", count: 2):
  session "尝试 1"
  session "尝试 2"
  session "尝试 3"
```

#### All（全部 - 默认） (All - Default)

等待所有分支完成：

```prose
# 隐式 - 这是默认行为
parallel:
  session "任务 A"
  session "任务 B"

# 显式
parallel ("all"):
  session "任务 A"
  session "任务 B"
```

### 失败策略 (Failure Policies)

控制并行块如何处理分支失败：

#### Fail-Fast（快速失败 - 默认） (Fail-Fast - Default)

如果任何分支失败，立即失败并取消其他分支：

```prose
parallel:  # 隐式快速失败
  session "关键任务 1"
  session "关键任务 2"

# 显式
parallel (on-fail: "fail-fast"):
  session "关键任务 1"
  session "关键任务 2"
```

#### Continue（继续） (Continue)

让所有分支完成，然后报告所有失败：

```prose
parallel (on-fail: "continue"):
  session "任务 1"
  session "任务 2"
  session "任务 3"

# 无论哪些分支失败都会继续运行
session "处理结果，包括失败项"
```

#### Ignore（忽略） (Ignore)

忽略所有失败，总是视为成功：

```prose
parallel (on-fail: "ignore"):
  session "可选增强任务 1"
  session "可选增强任务 2"

# 即使所有分支都失败，这也会运行
session "无论如何都继续"
```

### 组合修饰符 (Combining Modifiers)

等待策略和失败策略可以结合使用：

```prose
# 兼顾速度与韧性
parallel ("first", on-fail: "continue"):
  session "快速但不稳定"
  session "慢速但稳定"

# 获取任意 2 个结果，忽略失败
parallel ("any", count: 2, on-fail: "ignore"):
  session "方案 1"
  session "方案 2"
  session "方案 3"
  session "方案 4"
```

### 执行语义 (Execution Semantics)

当 OpenProse 虚拟机遇到 `parallel:` 块时：

1. **分叉 (Fork)**：同时启动所有分支
2. **执行 (Execute)**：每个分支独立运行
3. **汇合 (Join)**：根据等待策略等待：
   - `"all"`（默认）：等待所有分支
   - `"first"`：只要第一个完成就返回
   - `"any"`：只要第一个成功（或使用 `count` 指定的 N 个成功）就返回
4. **处理失败 (Handle failures)**：根据失败策略处理：
   - `"fail-fast"`（默认）：取消剩余分支并立即失败
   - `"continue"`：等待所有分支，然后报告失败
   - `"ignore"`：将失败视为成功
5. **继续 (Continue)**：带着可用结果继续执行下一条语句

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 等待策略无效 | 错误 | 必须是 "all", "first", 或 "any" |
| 失败策略无效 | 错误 | 必须是 "fail-fast", "continue", 或 "ignore" |
| 使用 count 但未指定 "any" | 错误 | count 仅在 "any" 策略下有效 |
| count 小于 1 | 错误 | count 必须至少为 1 |
| count 超过分支数 | 警告 | count 超过了并行分支的数量 |
| 并行块中变量重复 | 错误 | 变量已定义 |
| 变量名与代理名冲突 | 错误 | 变量名与代理名冲突 |
| 对象上下文中变量未定义 | 错误 | 上下文中的变量未定义 |

---

## 固定循环 (Fixed Loops)

固定循环提供对预定次数或集合的受限迭代。

### Repeat 块 (Repeat Block)

`repeat` 块执行其主体固定的次数。

#### 基本语法 (Basic Syntax)

```prose
repeat 3:
  session "生成一个创意点子"
```

#### 使用索引变量 (With Index Variable)

使用 `as` 访问当前迭代索引：

```prose
repeat 5 as i:
  session "处理项目"
    context: i
```

索引变量 `i` 的作用域限于循环主体，且从 0 开始。

### For-Each 块 (For-Each Block)

`for` 块遍历一个集合。

#### 基本语法 (Basic Syntax)

```prose
let fruits = ["苹果", "香蕉", "樱桃"]
for fruit in fruits:
  session "描述这种水果"
    context: fruit
```

#### 使用行内数组 (With Inline Array)

```prose
for topic in ["AI", "气候", "太空"]:
  session "研究这个主题"
    context: topic
```

#### 使用索引变量 (With Index Variable)

同时访问项目及其索引：

```prose
let items = ["a", "b", "c"]
for item, i in items:
  session "处理带索引的项目"
    context: [item, i]
```

### 并行 For-Each (Parallel For-Each)

`parallel for` 块并发运行所有迭代（扇出模式）：

```prose
let topics = ["AI", "气候", "太空"]
parallel for topic in topics:
  session "研究这个主题"
    context: topic

session "合并所有研究成果"
```

这等同于：

```prose
parallel:
  session "研究 AI" context: "AI"
  session "研究气候" context: "气候"
  session "研究太空" context: "太空"
```

但更简洁且动态。

### 变量作用域 (Variable Scoping)

循环变量的作用域限于循环主体：

- 它们在每次迭代中隐式地为 `const`（不可变）
- 它们会遮蔽同名的外部变量（会发出警告）
- 它们在循环外部不可访问

```prose
let item = session "outer"
for item in ["a", "b"]:
  # 这里的 'item' 是循环变量
  session "process loop item"
    context: item
# 这里的 'item' 再次引用外部变量
session "use outer item"
  context: item
```

### 嵌套 (Nesting)

循环可以嵌套：

```prose
repeat 2:
  repeat 3:
    session "内部任务"
```

不同类型的循环可以组合：

```prose
let items = ["a", "b"]
repeat 2:
  for item in items:
    session "处理项目"
      context: item
```

### 完整示例 (Complete Example)

```prose
# 生成点子的多个变体
repeat 3:
  session "生成一个创意的初创公司点子"

session "从上述选项中选择最佳点子"

# 从多个角度研究选定的点子
let angles = ["市场", "技术", "竞争"]
parallel for angle in angles:
  session "从这个角度研究初创公司点子"
    context: angle

session "将所有研究综合成一份商业计划书"
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 重复次数必须为正数 | 错误 | 重复次数必须为正数 |
| 重复次数必须为整数 | 错误 | 重复次数必须为整数 |
| 集合变量未定义 | 错误 | 集合变量未定义 |
| 循环变量遮蔽外部变量 | 警告 | 循环变量遮蔽了外部变量 |

---

## 无界循环 (Unbounded Loops)

无界循环提供带有 AI 评估终止条件的迭代。与固定循环不同，迭代次数事先未知——OpenProse 虚拟机在运行时利用其智能评估条件，以决定何时停止。

### 自由裁量标记 (Discretion Markers)

无界循环使用**自由裁量标记** (`**...**`) 来包裹 AI 评估的条件。这些标记发出信号，表示封闭的文本应由 OpenProse 虚拟机在运行时进行智能解读，而不是作为字面上的布尔表达式。

```prose
# **...** 内部的文本由 AI 进行评估
loop until **这首诗意象生动且行文流畅**:
  session "审查并改进这首诗"
```

对于多行条件，请使用三个星号：

```prose
loop until ***
  文档已完成
  所有部分均已审阅
  且格式一致
***:
  session "继续编写文档"
```

### 基本循环 (Basic Loop)
` |
| Failure policies | 已实现 | `parallel (on-fail: "continue"):` |
| Repeat blocks | 已实现 | `repeat N:` 固定次数迭代 |
| Repeat with index | 已实现 | 带有索引变量的 `repeat N as i:` |
| For-each blocks | 已实现 | `for item in items:` 迭代 |
| For-each with index | 已实现 | 带有索引的 `for item, i in items:` |
| Parallel for-each | 已实现 | `parallel for item in items:` 扇出 (fan-out) |
| Unbounded loop | 已实现 | 带有可选最大迭代次数的 `loop:` |
| Loop until | 已实现 | `loop until **condition**:` 由 AI 评估 |
| Loop while | 已实现 | `loop while **condition**:` 由 AI 评估 |
| Loop with index | 已实现 | `loop as i:` 或 `loop until ... as i:` |
| Map pipeline | 已实现 | `items | map:` 转换每个项目 |
| Filter pipeline | 已实现 | `items | filter:` 保留匹配的项目 |
| Reduce pipeline | 已实现 | `items | reduce(acc, item):` 累加 |
| Parallel map | 已实现 | `items | pmap:` 并发转换 |
| Pipeline chaining | 已实现 | `| filter: ... | map: ...` |
| Try/catch blocks | 已实现 | 带有 `catch:` 的 `try:` 用于错误处理 |
| Try/catch/finally | 已实现 | 用于清理的 `finally:` |
| Error variable | 已实现 | `catch as err:` 访问错误上下文 |
| Throw statement | 已实现 | `throw` 或 `throw "message"` |
| Retry property | 已实现 | `retry: 3` 失败时自动重试 |
| Backoff strategy | 已实现 | `backoff: exponential` 重试间的延迟 |
| Input declarations | 已实现 | `input name: "description"` |
| Output bindings | 已实现 | `output name = expression` |
| Program invocation | 已实现 | `name(input: value)` 调用导入的程序 |
| Multi-line strings | 已实现 | `"""..."""` 保留空格 |
| String interpolation | 已实现 | `"Hello {name}"` 变量替换 |
| Block parameters | 已实现 | 带有参数的 `block name(param):` |
| Block invocation args | 已实现 | `do name(arg)` 传递参数 |
| Choice blocks | 已实现 | `choice **criteria**: option "label":` |
| If/elif/else | 已实现 | `if **condition**:` 条件分支 |
| Persistent agents | 已实现 | `persist: true` 或 `persist: project` |
| Resume statement | 已实现 | `resume: agent` 带着记忆继续 |

---

## 文件格式 (File Format)

| 属性 (Property) | 值 (Value) |
| :--- | :--- |
| 扩展名 | `.prose` |
| 编码 | UTF-8 |
| 大小写敏感性 | 区分大小写 |
| 缩进 | 空格（类似 Python） |
| 换行符 | LF 或 CRLF |

---

## 注释 (Comments)

注释在程序中提供文档说明，在执行期间会被忽略。

### 语法 (Syntax)

```prose
# 这是一个独立注释

session "Hello"  # 这是一个行内注释
```

### 规则 (Rules)

1. 注释以 `#` 开头并延伸至行尾
2. 注释可以出现在独立的一行，也可以出现在语句之后
3. 空注释是有效的：`#`
4. 字符串字面量内部的 `#` 字符**不是**注释

### 示例 (Examples)

```prose
# 程序头注释
# 作者: Example

session "Do something"  # 解释此操作的作用

# 此注释位于语句之间
session "Do another thing"
```

### 编译行为 (Compilation Behavior)

注释在**编译期间会被移除**。OpenProse 虚拟机永远不会看到它们。它们对执行没有影响，纯粹是为了人类阅读文档而存在。

### 重要说明 (Important Notes)

- **字符串内部的注释不是注释**：

  ```prose
  session "Say hello # this is part of the string"
  ```

  字符串字面量内部的 `#` 是提示词 (prompt) 的一部分，而不是注释。

- **允许在缩进块内使用注释**：
  ```prose
  agent researcher:
      # 此注释在块内部
      model: sonnet
  # 此注释在块外部
  ```

---

## 字符串字面量 (String Literals)

字符串字面量表示文本值，主要用于 `session` 的提示词。

### 语法 (Syntax)

字符串用双引号括起来：

```prose
"这是一个字符串"
```

### 转义序列 (Escape Sequences)

支持以下转义序列：

| 序列 (Sequence) | 含义 (Meaning) |
| :--- | :--- |
| `\\` | 反斜杠 (Backslash) |
| `\"` | 双引号 (Double quote) |
| `\n` | 换行符 (Newline) |
| `\t` | 制表符 (Tab) |

### 示例 (Examples)

```prose
session "Hello world"
session "第一行\n第二行"
session "她说 \"你好\""
session "路径: C:\\Users\\name"
session "第一列\t第二列"
```

### 规则 (Rules)

1. 单行字符串必须以闭合的 `"` 正确终止
2. 未知的转义序列将被视为错误
3. 空字符串 `""` 是有效的，但用作提示词时会生成警告

### 多行字符串 (Multi-line Strings)

多行字符串使用三个双引号 (`"""`)，并保留内部的空格和换行：

```prose
session """
这是一个多行提示词。
它保留了：
  - 缩进
  - 换行
  - 所有内部空格
"""
```

#### 多行字符串规则 (Multi-line String Rules)

1. 起始的 `"""` 必须紧跟一个换行符
2. 内容一直持续到结束的 `"""`
3. 转义序列的工作方式与单行字符串相同
4. 分隔符内部的起始/末尾空格将被保留

### 字符串插值 (String Interpolation)

字符串可以使用 `{varname}` 语法嵌入变量引用：

```prose
let name = session "获取用户名"

session "你好 {name}，欢迎使用本系统！"
```

#### 插值语法 (Interpolation Syntax)

- 通过将变量名包裹在大括号中来引用变量：`{varname}`
- 适用于单行和多行字符串
- 空大括号 `{}` 被视为普通文本，而不是插值
- 不支持嵌套大括号

#### 示例 (Examples)

```prose
let research = session "研究该主题"
let analysis = session "分析发现"

# 单变量插值
session "基于 {research}，提供建议"

# 多变量插值
session "结合 {research} 与 {analysis}，综合见解"

# 带插值的多行字符串
session """
评论摘要：
- 研究：{research}
- 分析：{analysis}
请提供最终建议。
"""
```

#### 插值规则 (Interpolation Rules)

1. 变量名必须是有效的标识符
2. 引用的变量必须在作用域内
3. 空大括号 `{}` 是普通文本
4. 反斜杠可以转义大括号：`\{` 产生字面量 `{`

### 验证 (Validation)

| 检查项 (Check) | 结果 (Result) |
| :--- | :--- |
| 未终止的字符串 | 错误 |
| 未知的转义序列 | 错误 |
| 提示词为空字符串 | 警告 |
| 插值变量未定义 | 错误 |

---

## Use 语句（程序组合） (Use Statements - Program Composition)

`use` 语句从位于 `p.prose.md` 的注册中心导入其他 OpenProse 程序，从而实现模块化工作流。

### 语法 (Syntax)

```prose
use "@handle/slug"
use "@handle/slug" as alias
```

### 路径格式 (Path Format)

导入路径遵循 `@handle/slug` 格式：
- `@handle` 标识程序作者/组织
- `slug` 是程序名称

可选的别名 (`as name`) 允许通过更短的名称进行引用。

### 示例 (Examples)

```prose
# 导入程序
use "@alice/research"

# 带有别名的导入
use "@bob/critique" as critic
```

### 程序 URL 解析 (Program URL Resolution)

当 OpenProse 虚拟机遇到 `use` 语句时：

1. 从 `https://p.prose.md/@handle/slug` 获取程序
2. 解析程序以提取其契约 (contract)（输入/输出）
3. 在导入注册中心 (Import Registry) 中注册程序

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 路径为空 | 错误 | Use 路径不能为空 |
| 路径格式无效 | 错误 | 路径必须符合 @handle/slug 格式 |
| 重复导入 | 错误 | 程序已导入 |
| 重复导入缺少别名 | 错误 | 导入多个程序时需要别名 |

### 执行语义 (Execution Semantics)

`use` 语句在任何 `agent` 定义或 `session` 之前处理。OpenProse 虚拟机：

1. 在执行开始时获取并验证所有导入的程序
2. 从每个程序中提取输入/输出契约
3. 在导入注册中心注册程序以供后续调用

---

## Input 声明 (Input Declarations)

`input` 声明程序期望从调用者处获取哪些值。

### 语法 (Syntax)

```prose
input name: "description"
```

### 示例 (Examples)

```prose
input topic: "要研究的主题"
input depth: "研究深度（shallow, medium, deep）"
```

### 语义 (Semantics)

输入 (Inputs)：
- 在程序顶部声明（在可执行语句之前）
- 具有名称和描述（用于文档说明）
- 在程序主体中作为变量使用
- 在调用程序时必须由调用者提供

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 输入名称为空 | 错误 | 输入名称不能为空 |
| 描述为空 | 警告 | 建议添加描述 |
| 输入名称重复 | 错误 | 输入已声明 |
| 输入在可执行语句之后 | 错误 | 输入必须在可执行语句之前声明 |

---

## Output 绑定 (Output Bindings)

`output` 声明程序为调用者产生哪些值。

### 语法 (Syntax)

```prose
output name = expression
```

### 示例 (Examples)

```prose
let raw = session "研究 {topic}"
output findings = session "综合研究成果"
  context: raw
output sources = session "提取来源"
  context: raw
```

### 语义 (Semantics)

`output` 关键字：
- 将变量标记为输出（在赋值时可见，而不仅仅在文件顶部）
- 工作方式类似于 `let`，但同时将该值注册为程序输出
- 可以出现在程序主体的任何位置
- 支持多个输出

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 输出名称为空 | 错误 | 输出名称不能为空 |
| 输出名称重复 | 错误 | 输出已声明 |
| 输出名称冲突 | 错误 | 输出名称与变量冲突 |

---

## 程序调用 (Program Invocation)

通过提供输入来调用导入的程序。

### 语法 (Syntax)

```prose
name(input1: value1, input2: value2)
```

### 示例 (Examples)

```prose
use "@alice/research" as research

let result = research(topic: "quantum computing")
```

### 访问输出 (Accessing Outputs)

结果包含被调用程序的所有输出，可通过属性访问：

```prose
session "编写摘要"
  context: result.findings

session "引用来源"
  context: result.sources
```

### 输出解构 (Destructuring Outputs)

为了方便起见，输出可以进行解构 (destructured)：

```prose
let { findings, sources } = research(topic: "quantum computing")
```

### 执行语义 (Execution Semantics)

当一个程序调用导入的程序时：

1. **绑定输入**：将调用者提供的值映射到导入程序的输入
2. **执行**：运行导入的程序（生成其自己的会话）
3. **收集输出**：收集导入程序中所有的 `output` 绑定
4. **返回**：将输出作为结果对象提供给调用者

导入的程序在其自己的执行上下文中运行，但共享同一个虚拟机 (VM) 会话。

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 程序未知 | 错误 | 程序未导入 |
| 缺少必需的输入 | 错误 | 未提供必需的输入 |
| 输入名称未知 | 错误 | 输入未在程序中声明 |
| 输出属性未知 | 错误 | 输出未在程序中声明 |

---

## Agent 定义 (Agent Definitions)

Agent（代理）是用于配置子代理行为的可重用模板。一旦定义，Agent 就可以在 `session` 语句中被引用。

### 语法 (Syntax)

```prose
agent name:
  model: sonnet
  prompt: "此代理的系统提示词"
  skills: ["skill1", "skill2"]
  permissions:
    read: ["*.md"]
    bash: deny
```

### 属性 (Properties)

| 属性 (Property) | 类型 (Type) | 值 (Values) | 描述 (Description) |
| :--- | :--- | :--- | :--- |
| `model` | 标识符 | `sonnet`, `opus`, `haiku` | 要使用的 Claude 模型 |
| `prompt` | 字符串 | 任何字符串 | 代理的系统提示词/上下文 |
| `persist` | 值 | `true`, `project`, 或字符串 | 为代理启用持久化记忆 |
| `skills` | 数组 | 字符串数组 | 分配给此代理的技能 |
| `permissions` | 块 (block) | 权限规则 | 代理的访问控制 |

### Persist 属性 (Persist Property)

`persist` 属性使代理能够在多次调用之间保持记忆：

```prose
# 执行级持久化（记忆随运行结束而消失）
agent captain:
  model: opus
  persist: true
  prompt: "你负责协调和审查"

# 项目级持久化（记忆在多次运行之间存续）
agent advisor:
  model: opus
  persist: project
  prompt: "你提供架构指导"

# 自定义路径持久化
agent shared:
  model: opus
  persist: ".prose/custom/shared-agent/"
  prompt: "跨程序共享"
```

| 值 (Value) | 记忆位置 (Memory Location) | 生命周期 (Lifetime) |
| :--- | :--- | :--- |
| `true` | `.prose/runs/{id}/agents/{name}/` | 随执行结束 |
| `project` | `.prose/agents/{name}/` | 跨执行存续 |
| 字符串 | 指定的路径 | 用户控制 |

### Skills 属性 (Skills Property)

`skills` 属性将导入的技能分配给代理：

```prose
use "@anthropic/web-search"
use "@anthropic/summarizer" as summarizer

agent researcher:
  skills: ["web-search", "summarizer"]
```

技能必须在分配之前导入。引用未导入的技能会生成警告。

### Permissions 属性 (Permissions Property)

`permissions` 属性控制代理的访问权限：

```prose
agent secure-agent:
  permissions:
    read: ["*.md", "*.txt"]
    write: ["output/"]
    bash: deny
    network: allow
```

#### 权限类型 (Permission Types)

| 类型 (Type) | 描述 (Description) |
| :--- | :--- |
| `read` | 代理可以读取的文件（glob 模式） |
| `write` | 代理可以写入的文件（glob 模式） |
| `execute` | 代理可以执行的文件（glob 模式） |
| `bash` | Shell 访问：`allow`, `deny`, 或 `prompt` |
| `network` | 网络访问：`allow`, `deny`, 或 `prompt` |

#### 权限值 (Permission Values)

| 值 (Value) | 描述 (Description) |
| :--- | :--- |
| `allow` | 授予权限 |
| `deny` | 拒绝权限 |
| `prompt` | 询问用户以获取权限 |
| 数组 (Array) | 允许的模式列表（用于 read/write/execute） |

### 示例 (Examples)

```prose
# 定义一个研究代理
agent researcher:
  model: sonnet
  prompt: "你是一个研究助手，擅长寻找和综合信息"

# 定义一个写作代理
agent writer:
  model: opus
  prompt: "你是一个技术作家，负责创建清晰、简洁的文档"

# 仅指定模型的代理
agent quick:
  model: haiku

# 仅指定提示词的代理
agent expert:
  prompt: "你是一个领域专家"

# 带有技能的代理
agent web-researcher:
  model: sonnet
  skills: ["web-search", "summarizer"]

# 带有权限的代理
agent file-handler:
  permissions:
    read: ["*.md", "*.txt"]
    write: ["output/"]
    bash: deny
```

### 模型选择 (Model Selection)

| 模型 (Model) | 用例 (Use Case) |
| :--- | :--- |
| `haiku` | 快速、简单的任务；快速响应 |
| `sonnet` | 平衡的性能；通用用途 |
| `opus` | 复杂的推理；详细的分析 |

### 执行语义 (Execution Semantics)

当 session 引用一个代理时：

1. 代理的 `model` 属性决定使用哪个 Claude 模型
2. 代理的 `prompt` 属性作为系统上下文包含在内
3. Session 属性可以覆盖代理的默认值

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 代理名称重复 | 错误 | 代理已定义 |
| 模型值无效 | 错误 | 必须是 sonnet, opus 或 haiku |
| prompt 属性为空 | 警告 | 建议提供提示词 |
| 属性重复 | 错误 | 属性已指定 |

---

## Session 语句 (Session Statement)

Session 语句是 OpenProse 中主要的执行构造。它生成一个子代理来完成任务。

### 语法变体 (Syntax Variants)

#### 简单 Session（带有行内提示词）

```prose
session "提示词文本"
```

#### 带有代理引用的 Session

```prose
session: agentName
```

#### 命名的带有代理的 Session

```prose
session sessionName: agentName
```

#### 带有属性的 Session

```prose
session: agentName
  prompt: "覆盖代理的默认提示词"
  model: opus  # 覆盖代理的模型
```

### 属性覆盖 (Property Overrides)

当 session 引用一个代理时，它可以覆盖代理的属性：

```prose
agent researcher:
  model: sonnet
  prompt: "你是一个研究助手"

# 使用 researcher 代理，但使用不同的模型
session: researcher
  model: opus

# 使用 researcher 代理，但使用不同的提示词
session: researcher
  prompt: "深入研究这个特定主题"

# 同时覆盖两者
session: researcher
  model: opus
  prompt: "专业研究任务"
```

### 执行语义 (Execution Semantics)

当 OpenProse 虚拟机遇到 `session` 语句时：

1. **解析配置**：合并代理默认值与 session 覆盖值
2. **生成子代理**：使用解析后的配置创建一个新的 Claude 子代理
3. **发送提示词**：将提示词字符串传递给子代理
4. **等待完成**：阻塞直到子代理完成
5. **继续**：继续执行下一条语句

### 执行流程图 (Execution Flow Diagram)

```
OpenProse 虚拟机 (VM)            子代理 (Subagent)
    |                              |
    |  生成 session                |
    |----------------------------->|
    |                              |
    |  发送提示词                  |
    |----------------------------->|
    |                              |
    |  [处理中...]                 |
    |                              |
    |  session 完成                |
    |<-----------------------------|
    |                              |
    |  继续执行下一条语句          |
    v                              v
```

### 顺序执行 (Sequential Execution)

多个 session 顺序执行：

```prose
session "任务一"
session "任务二"
session "任务三"
```

每个 session 都会等待前一个完成后再开始。

### 使用 Claude Code 的 Task 工具 (Using Claude Code's Task Tool)

要执行 session，请使用 Task 工具：

```typescript
// 简单 session
```typescript
// 带有代理配置的 Session
Task({
  description: "OpenProse session",
  prompt: "session 提示词",
  subagent_type: "general-purpose",
  model: "opus", // 来自代理或覆盖
});
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 缺少提示词和代理 | 错误 | Session 需要提示词或代理引用 |
| 代理引用未定义 | 错误 | 代理未定义 |
| 提示词为空 `""` | 警告 | Session 提示词为空 |
| 提示词仅包含空格 | 警告 | Session 提示词仅包含空格 |
| 提示词 > 10,000 字符 | 警告 | 考虑拆分为更小的任务 |
| 属性重复 | 错误 | 属性已指定 |

### 示例 (Examples)

```prose
# 简单 session
session "Hello world"

# 带有代理的 session
agent researcher:
  model: sonnet
  prompt: "你会深入研究主题"

session: researcher
  prompt: "研究量子计算的应用"

# 命名的 session
session analysis: researcher
  prompt: "分析竞争格局"
```

### 标准形式 (Canonical Form)

编译后的输出保留了结构：

```
输入：
agent researcher:
  model: sonnet

session: researcher
  prompt: "进行研究"

输出：
agent researcher:
  model: sonnet
session: researcher
  prompt: "进行研究"
```

---

## Resume 语句 (Resume Statement)

`resume` 语句带着累积的记忆继续执行持久化代理 (persistent agent)。

### 语法 (Syntax)

```prose
resume: agentName
  prompt: "从我们上次停下的地方继续"
```

### 语义 (Semantics)

| 关键字 (Keyword) | 行为 (Behavior) |
| :--- | :--- |
| `session:` | 忽略现有记忆，从头开始 |
| `resume:` | 加载记忆，带着上下文继续 |

### 示例 (Examples)

```prose
agent captain:
  model: opus
  persist: true
  prompt: "你负责协调和审查"

# 第一次调用 - 创建记忆
session: captain
  prompt: "审查计划"
  context: plan

# 之后的调用 - 加载记忆
resume: captain
  prompt: "审查计划的第一步"
  context: step1

# 输出捕获同样适用于 resume
let review = resume: captain
  prompt: "对所有步骤进行最终审查"
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 在非持久化代理上使用 `resume:` | 错误 | 代理必须具有 `persist:` 属性才能使用 `resume:` |
| `resume:` 时没有现有记忆 | 错误 | 代理不存在记忆文件；第一次调用请使用 `session:` |
| 在有记忆的持久化代理上使用 `session:` | 警告 | 将忽略现有记忆；请使用 `resume:` 继续 |
| 代理引用未定义 | 错误 | 代理未定义 |

---

## 变量与上下文 (Variables & Context)

变量允许你捕获 session 的结果，并将其作为 `context`（上下文）传递给后续的 session。

### Let 绑定 (Let Binding)

`let` 关键字创建一个可变变量，绑定到 session 的结果：

```prose
let research = session "深入研究该主题"

# research 现在保存了该 session 的输出
```

变量可以被重新赋值：

```prose
let draft = session "编写初稿"

# 修改草稿
draft = session "改进草稿"
  context: draft
```

### Const 绑定 (Const Binding)

`const` 关键字创建一个不可变变量：

```prose
const config = session "获取配置设置"

# 这将是一个错误：
# config = session "尝试修改"
```

### Context 属性 (Context Property)

`context` 属性将之前 session 的输出传递给新的 session：

#### 单个上下文 (Single Context)

```prose
let research = session "研究量子计算"

session "编写摘要"
  context: research
```

#### 多个上下文 (Multiple Contexts)

```prose
let research = session "研究主题"
let analysis = session "分析发现"

session "编写最终报告"
  context: [research, analysis]
```

#### 空上下文（全新开始） (Empty Context - Fresh Start)

使用空数组来启动一个不继承上下文的 session：

```prose
session "独立任务"
  context: []
```

#### 对象上下文简写 (Object Context Shorthand)

为了传递多个命名的结果（特别是来自 `parallel` 块的结果），请使用对象简写：

```prose
parallel:
  a = session "任务 A"
  b = session "任务 B"

session "合并结果"
  context: { a, b }
```

这相当于传递一个对象，其中每个属性都是一个变量引用。

### 完整示例 (Complete Example)

```prose
agent researcher:
  model: sonnet
  prompt: "你是一个研究助手"

agent writer:
  model: opus
  prompt: "你是一个技术作家"

# 收集研究资料
let research = session: researcher
  prompt: "研究量子计算的发展"

# 分析发现
let analysis = session: researcher
  prompt: "分析关键发现"
  context: research

# 使用两个上下文编写最终报告
const report = session: writer
  prompt: "编写一份综合报告"
  context: [research, analysis]
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 变量名重复 | 错误 | 变量已定义 |
| Const 重新赋值 | 错误 | 无法对 const 变量重新赋值 |
| 变量引用未定义 | 错误 | 变量未定义 |
| 变量名与代理名冲突 | 错误 | 变量名与代理名冲突 |
| 上下文变量未定义 | 错误 | 上下文中的变量未定义 |
| 上下文数组包含非标识符 | 错误 | 上下文数组元素必须是变量引用 |

### 扁平命名空间要求 (Flat Namespace Requirement)

所有变量名在**程序内必须是唯一的**。不允许跨作用域的遮蔽 (shadowing)。

**以下是一个编译错误：**

```prose
let result = session "外部任务"

for item in items:
  let result = session "内部任务"   # 错误：'result' 已定义
    context: item
```

**为何有此约束**：由于绑定存储为 `bindings/{name}.md`，两个同名变量会在文件系统上发生冲突。我们与其引入复杂的作用域规则，不如强制要求唯一性。

**此约束防止的冲突场景：**
1. 循环内部的变量遮蔽循环外部的变量
2. 不同 `if`/`elif`/`else` 分支中的变量重名
3. 块 (block) 参数遮蔽外部变量
4. 并行分支重复使用外部变量名

**例外**：导入的程序在隔离的命名空间中运行。主程序中的变量 `result` 不会与导入程序中的 `result`发生冲突（它们写入不同的 `imports/{handle}--{slug}/bindings/` 目录）。

---

## 组合块 (Composition Blocks)

组合块允许你将程序结构化为可重用的命名单元，并以内联方式表达操作序列。

### do: 块（匿名顺序执行块） (Anonymous Sequential Block)

`do:` 关键字创建一个显式的顺序执行块。块中的所有语句按顺序执行。

#### 语法 (Syntax)

```prose
do:
  statement1
  statement2
  ...
```

#### 示例 (Examples)

```prose
# 显式的顺序执行块
do:
  session "研究主题"
  session "分析发现"
  session "编写摘要"

# 将结果赋值给变量
let result = do:
  session "收集数据"
  session "处理数据"
```

### 块定义 (Block Definitions)

命名块 (Named blocks) 创建可重用的工作流组件。定义一次，多次调用。

#### 语法 (Syntax)

```prose
block name:
  statement1
  statement2

#### 调用块 (Invoking Blocks)

使用 `do` 后跟块名来调用已定义的块：

```prose
do blockname
```

#### 示例 (Examples)

```prose
# 定义一个审查管道
block review-pipeline:
  session "安全审查"
  session "性能审查"
  session "综合审查结果"

# 定义另一个块
block final-check:
  session "最终验证"
  session "签核 (Sign off)"

# 使用这些块
do review-pipeline
session "基于审查进行修复"
do final-check
```

### 块参数 (Block Parameters)

块可以接受参数，使其更加灵活和可重用。

#### 语法 (Syntax)

```prose
block name(param1, param2):
  # param1 和 param2 在这里可用
  statement1
  statement2
```

#### 带参数调用 (Invoking with Arguments)

在调用参数化块时传递参数：

```prose
do name(arg1, arg2)
```

#### 示例 (Examples)

```prose
# 定义一个参数化块
block review(topic):
  session "深入研究 {topic}"
  session "分析关于 {topic} 的关键发现"
  session "总结 {topic} 分析"

# 使用不同参数调用
do review("量子计算")
do review("机器学习")
do review("区块链")
```

#### 多个参数 (Multiple Parameters)

```prose
block process-item(item, mode):
  session "使用 {mode} 模式处理 {item}"
  session "验证 {item} 处理结果"

do process-item("data.csv", "strict")
do process-item("config.json", "lenient")
```

#### 参数作用域 (Parameter Scope)

- 参数的作用域限于块主体内
- 参数会遮蔽同名的外部变量（会发出警告）
- 参数在块内隐式地为 `const`（不可变）

#### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 参数数量不匹配 | 警告 | 块期望 N 个参数，但收到了 M 个参数 |
| 参数遮蔽外部变量 | 警告 | 参数遮蔽了外部变量 |

### 行内序列（箭头操作符） (Inline Sequence - Arrow Operator)

`->` 操作符可以将多个 session 在单行中链接成序列。这是顺序执行的语法糖。

#### 语法 (Syntax)

```prose
session "A" -> session "B" -> session "C"
```

这等同于：

```prose
session "A"
session "B"
session "C"
```

#### 示例 (Examples)

```prose
# 快速管道
session "计划" -> session "执行" -> session "审查"

# 结果赋值
let workflow = session "起草" -> session "编辑" -> session "定稿"
```

### 块提升 (Block Hoisting)

块定义是提升 (hoisted) 的——你可以在源文件中定义块之前就使用它：

```prose
# 在定义之前使用
do validation-checks

# 定义在后面
block validation-checks:
  session "检查语法"
  session "检查语义"
```

### 嵌套组合 (Nested Composition)

块 (block) 和 `do:` 块可以嵌套使用：

```prose
block outer-workflow:
  session "开始"
  do:
    session "子任务 1"
    session "子任务 2"
  session "结束"

do:
  do outer-workflow
  session "最终步骤"
```

### 带有块的上下文 (Context with Blocks)

块可以与上下文系统协同工作：

```prose
# 捕获 do 块的结果
let research = do:
  session "收集信息"
  session "分析模式"

# 在后续 session 中使用
session "编写报告"
  context: research
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 块引用未定义 | 错误 | 块未定义 |
| 块定义重复 | 错误 | 块已定义 |
| 块名与代理名冲突 | 错误 | 块名与代理名冲突 |
| 块名为空 | 错误 | 块定义必须有名称 |

---

## 并行块 (Parallel Blocks)

并行块允许并行执行多个 session。所有分支同时执行，块会等待所有分支完成后再继续。

### 基本语法 (Basic Syntax)

```prose
parallel:
  session "安全审查"
  session "性能审查"
  session "风格审查"
```

所有三个 session 同时启动并并发运行。程序会等待它们全部完成后再继续。

### 命名的并行结果 (Named Parallel Results)

将并行分支的结果捕获到变量中：

```prose
parallel:
  security = session "安全审查"
  perf = session "性能审查"
  style = session "风格审查"
```

这些变量随后可以用于后续的 session。

### 对象上下文简写 (Object Context Shorthand)

使用对象简写将多个并行结果传递给 session：

```prose
parallel:
  security = session "安全审查"
  perf = session "性能审查"
  style = session "风格审查"

session "综合所有审查结果"
  context: { security, perf, style }
```

对象简写 `{ a, b, c }` 等同于传递一个具有属性 `a`、`b` 和 `c` 的对象，每个属性的值对应相应的变量。

### 混合组合 (Mixed Composition)

#### 顺序执行中的并行 (Parallel Inside Sequential)

```prose
do:
  session "设置 (Setup)"
  parallel:
    session "任务 A"
    session "任务 B"
  session "清理 (Cleanup)"
```

先运行设置，然后并行运行任务 A 和任务 B，最后运行清理。

#### 并行执行中的顺序 (Sequential Inside Parallel)

```prose
parallel:
  do:
    session "多步任务 1a"
    session "多步任务 1b"
  do:
    session "多步任务 2a"
    session "多步任务 2b"
```

每个并行分支包含一个顺序工作流。这两个工作流并发运行。

### 将并行块赋值给变量 (Assigning Parallel Blocks to Variables)

```prose
let results = parallel:
  session "任务 A"
  session "任务 B"
```

### 完整示例 (Complete Example)

```prose
agent reviewer:
  model: sonnet

# 运行并行审查
parallel:
  sec = session: reviewer
    prompt: "审查安全问题"
  perf = session: reviewer
    prompt: "审查性能问题"
  style = session: reviewer
    prompt: "审查风格问题"

# 合并所有审查
session "创建统一的审查报告"
  context: { sec, perf, style }
```

### 等待策略 (Join Strategies)

默认情况下，并行块等待所有分支完成。你可以指定其他的等待策略：

#### First（竞速） (First - Race)

只要第一个分支完成就返回，并取消其他分支：

```prose
parallel ("first"):
  session "尝试方案 A"
  session "尝试方案 B"
  session "尝试方案 C"
```

第一个成功的结果胜出。其他分支将被取消。

#### Any（M 分之 N） (Any - N of M)

当任意 N 个分支成功完成时返回：

```prose
# 默认：任意 1 个成功
parallel ("any"):
  session "尝试 1"
  session "尝试 2"

# 指定数量：等待 2 个成功
parallel ("any", count: 2):
  session "尝试 1"
  session "尝试 2"
  session "尝试 3"
```

#### All（全部 - 默认） (All - Default)

等待所有分支完成：

```prose
# 隐式 - 这是默认行为
parallel:
  session "任务 A"
  session "任务 B"

# 显式
parallel ("all"):
  session "任务 A"
  session "任务 B"
```

### 失败策略 (Failure Policies)

控制并行块如何处理分支失败：

#### Fail-Fast（快速失败 - 默认） (Fail-Fast - Default)

如果任何分支失败，立即失败并取消其他分支：

```prose
parallel:  # 隐式快速失败
  session "关键任务 1"
  session "关键任务 2"

# 显式
parallel (on-fail: "fail-fast"):
  session "关键任务 1"
  session "关键任务 2"
```

#### Continue（继续） (Continue)

让所有分支完成，然后报告所有失败：

```prose
parallel (on-fail: "continue"):
  session "任务 1"
  session "任务 2"
  session "任务 3"

# 无论哪些分支失败都会继续运行
session "处理结果，包括失败项"
```

#### Ignore（忽略） (Ignore)

忽略所有失败，总是视为成功：

```prose
parallel (on-fail: "ignore"):
  session "可选增强任务 1"
  session "可选增强任务 2"

# 即使所有分支都失败，这也会运行
session "无论如何都继续"
```

### 组合修饰符 (Combining Modifiers)

等待策略和失败策略可以结合使用：

```prose
# 兼顾速度与韧性
parallel ("first", on-fail: "continue"):
  session "快速但不稳定"
  session "慢速但稳定"

# 获取任意 2 个结果，忽略失败
parallel ("any", count: 2, on-fail: "ignore"):
  session "方案 1"
  session "方案 2"
  session "方案 3"
  session "方案 4"
```

### 执行语义 (Execution Semantics)

当 OpenProse 虚拟机遇到 `parallel:` 块时：

1. **分叉 (Fork)**：同时启动所有分支
2. **执行 (Execute)**：每个分支独立运行
3. **汇合 (Join)**：根据等待策略等待：
   - `"all"`（默认）：等待所有分支
   - `"first"`：只要第一个完成就返回
   - `"any"`：只要第一个成功（或使用 `count` 指定的 N 个成功）就返回
4. **处理失败 (Handle failures)**：根据失败策略处理：
   - `"fail-fast"`（默认）：取消剩余分支并立即失败
   - `"continue"`：等待所有分支，然后报告失败
   - `"ignore"`：将失败视为成功
5. **继续 (Continue)**：带着可用结果继续执行下一条语句

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 等待策略无效 | 错误 | 必须是 "all", "first", 或 "any" |
| 失败策略无效 | 错误 | 必须是 "fail-fast", "continue", 或 "ignore" |
| 使用 count 但未指定 "any" | 错误 | count 仅在 "any" 策略下有效 |
| count 小于 1 | 错误 | count 必须至少为 1 |
| count 超过分支数 | 警告 | count 超过了并行分支的数量 |
| 并行块中变量重复 | 错误 | 变量已定义 |
| 变量名与代理名冲突 | 错误 | 变量名与代理名冲突 |
| 对象上下文中变量未定义 | 错误 | 上下文中的变量未定义 |

---

## 固定循环 (Fixed Loops)

固定循环提供对预定次数或集合的受限迭代。

### Repeat 块 (Repeat Block)

`repeat` 块执行其主体固定的次数。

#### 基本语法 (Basic Syntax)

```prose
repeat 3:
  session "生成一个创意点子"
```

#### 使用索引变量 (With Index Variable)

使用 `as` 访问当前迭代索引：

```prose
repeat 5 as i:
  session "处理项目"
    context: i
```

索引变量 `i` 的作用域限于循环主体，且从 0 开始。

### For-Each 块 (For-Each Block)

`for` 块遍历一个集合。

#### 基本语法 (Basic Syntax)

```prose
let fruits = ["苹果", "香蕉", "樱桃"]
for fruit in fruits:
  session "描述这种水果"
    context: fruit
```

#### 使用行内数组 (With Inline Array)

```prose
for topic in ["AI", "气候", "太空"]:
  session "研究这个主题"
    context: topic
```

#### 使用索引变量 (With Index Variable)

同时访问项目及其索引：

```prose
let items = ["a", "b", "c"]
for item, i in items:
  session "处理带索引的项目"
    context: [item, i]
```

### 并行 For-Each (Parallel For-Each)

`parallel for` 块并发运行所有迭代（扇出模式）：

```prose
let topics = ["AI", "气候", "太空"]
parallel for topic in topics:
  session "研究这个主题"
    context: topic

session "合并所有研究成果"
```

这等同于：

```prose
parallel:
  session "研究 AI" context: "AI"
  session "研究气候" context: "气候"
  session "研究太空" context: "太空"
```

但更简洁且动态。

### 变量作用域 (Variable Scoping)

循环变量的作用域限于循环主体：

- 它们在每次迭代中隐式地为 `const`（不可变）
- 它们会遮蔽同名的外部变量（会发出警告）
- 它们在循环外部不可访问

```prose
let item = session "outer"
for item in ["a", "b"]:
  # 这里的 'item' 是循环变量
  session "process loop item"
    context: item
# 这里的 'item' 再次引用外部变量
session "use outer item"
  context: item
```

### 嵌套 (Nesting)

循环可以嵌套：

```prose
repeat 2:
  repeat 3:
    session "内部任务"
```

不同类型的循环可以组合：

```prose
let items = ["a", "b"]
repeat 2:
  for item in items:
    session "处理项目"
      context: item
```

### 完整示例 (Complete Example)

```prose
# 生成点子的多个变体
repeat 3:
  session "生成一个创意的初创公司点子"

session "从上述选项中选择最佳点子"

# 从多个角度研究选定的点子
let angles = ["市场", "技术", "竞争"]
parallel for angle in angles:
  session "从这个角度研究初创公司点子"
    context: angle

session "将所有研究综合成一份商业计划书"
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 重复次数必须为正数 | 错误 | 重复次数必须为正数 |
| 重复次数必须为整数 | 错误 | 重复次数必须为整数 |
| 集合变量未定义 | 错误 | 集合变量未定义 |
| 循环变量遮蔽外部变量 | 警告 | 循环变量遮蔽了外部变量 |

---

## 无界循环 (Unbounded Loops)

无界循环提供带有 AI 评估终止条件的迭代。与固定循环不同，迭代次数事先未知——OpenProse 虚拟机在运行时利用其智能评估条件，以决定何时停止。

### 自由裁量标记 (Discretion Markers)

无界循环使用**自由裁量标记** (`**...**`) 来包裹 AI 评估的条件。这些标记发出信号，表示封闭的文本应由 OpenProse 虚拟机在运行时进行智能解读，而不是作为字面上的布尔表达式。

```prose
# **...** 内部的文本由 AI 进行评估
loop until **这首诗意象生动且行文流畅**:
  session "审查并改进这首诗"
```

对于多行条件，请使用三个星号：

```prose
loop until ***
  文档已完成
  所有部分均已审阅
  且格式一致
***:
  session "继续编写文档"
```

### 基本循环 (Basic Loop)

最简单的无界循环会无限运行，直到被显式限制：

```prose
loop:
  session "处理下一个项目"
```

**警告**：没有终止条件或最大迭代次数的循环会生成警告。务必包含一个安全限制：

```prose
loop (max: 50):
  session "处理下一个项目"
```

### Loop Until (直到...为止)

`loop until` 变体一直运行到条件变为真 (true) 为止：

```prose
loop until **任务已完成**:
  session "继续执行该任务"
```

OpenProse 虚拟机在每次迭代后都会评估自由裁量条件，并在确定满足条件时退出。

### Loop While (当...时)

`loop while` 变体在条件保持为真 (true) 时运行：

```prose
loop while **仍有项目待处理**:
  session "处理下一个项目"
```

从语义上讲，`loop while **X**` 等同于 `loop until **not X**`。

### 迭代变量 (Iteration Variable)

使用 `as` 跟踪当前迭代次数：

```prose
loop until **完成** as attempt:
  session "尝试该方案"
    context: attempt
```

迭代变量：

- 从 0 开始
- 每次迭代递增 1
- 作用域限于循环主体
- 在每次迭代中隐式地为 `const`（不可变）

### 安全限制 (Safety Limits)

使用 `(max: N)` 指定最大迭代次数：

```prose
# 即使未满足条件，也在 10 次迭代后停止
loop until **所有错误已修复** (max: 10):
  session "寻找并修复一个错误"
```

循环在以下情况下退出：

1. 满足条件（对于 `until`/`while` 变体），或者
2. 达到了最大迭代次数

### 完整语法 (Complete Syntax)

所有选项可以组合使用：

```prose
loop until **条件** (max: N) as i:
  body...
```

顺序很重要：条件在修饰符之前，修饰符在 `as` 之前。

### 示例 (Examples)

#### 迭代改进 (Iterative Improvement)

```prose
session "起草初稿"

loop until **草稿已打磨好并准备好进行评审** (max: 5):
  session "评审当前草稿并识别问题"
  session "修改草稿以解决这些问题"

session "展示最终草稿"
```

#### 调试工作流 (Debugging Workflow)

```prose
session "运行测试以识别失败项"

loop until **所有测试通过** (max: 20) as attempt:
  session "识别失败的测试"
  session "修复导致失败的错误"
  session "再次运行测试"

session "确认所有测试通过并总结修复情况"
```

#### 建立共识 (Consensus Building)

```prose
parallel:
  opinion1 = session "获取第一位专家的意见"
  opinion2 = session "获取第二位专家的意见"

loop until **专家已达成共识** (max: 5):
  session "识别分歧点"
    context: { opinion1, opinion2 }
  session "引导讨论以解决分歧"

session "记录最终共识"
```

#### 质量阈值 (Quality Threshold)

```prose
let draft = session "创建初始文档"

loop while **质量分数低于阈值** (max: 10):
  draft = session "评审并改进文档"
    context: draft
  session "计算新的质量分数"

session "定稿文档"
  context: draft
```

### 执行语义 (Execution Semantics)

当 OpenProse 虚拟机遇到无界循环时：

1. **初始化**：将迭代计数器设置为 0
2. **检查条件**（对于 `until`/`while`）：
   - 对于 `until`：如果满足条件则退出
   - 对于 `while`：如果不满足条件则退出
3. **检查限制**：如果迭代次数 >= 最大迭代次数则退出
4. **执行主体**：运行循环主体中的所有语句
5. **递增**：增加迭代计数器
6. **重复**：转到第 2 步

对于没有条件的简单 `loop:`：

- 只有最大迭代限制会导致退出
- 如果没有最大限制，循环将无限运行（会发出警告）

### 条件评估 (Condition Evaluation)

OpenProse 虚拟机利用其智能来评估自由裁量条件：

1. **上下文感知**：在会话截止目前所发生事情的背景下评估条件
2. **语义理解**：条件文本是按语义解读的，而非字面意思
3. **不确定性处理**：当不确定时，OpenProse 虚拟机可能会：
   - 如果正在取得进展，则继续迭代
   - 如果检测到边际收益递减，则提前退出
   - 基于条件的语义使用启发式方法

### 嵌套 (Nesting)

无界循环可以与其他循环类型嵌套：

```prose
# 固定循环内部嵌套无界循环
repeat 3:
  loop until **子任务完成** (max: 10):
    session "执行子任务"

# 无界循环内部嵌套固定循环
loop until **所有批次已处理** (max: 5):
  repeat 3:
    session "处理批次项目"

# 多个无界循环嵌套
loop until **外部条件** (max: 5):
  loop until **内部条件** (max: 10):
    session "深层迭代"
```

### 变量作用域 (Variable Scoping)

循环变量遵循与固定循环相同的作用域规则：

```prose
let i = session "outer"
loop until **完成** as i:
  # 这里的 'i' 是循环变量（遮蔽了外部变量）
  session "use loop i"
    context: i
# 这里的 'i' 再次引用外部变量
session "use outer i"
  context: i
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 循环没有最大次数或条件 | 警告 | 无界循环没有最大迭代次数 |
| 最大迭代次数 <= 0 | 错误 | 最大迭代次数必须为正数 |
| 最大迭代次数不是整数 | 错误 | 最大迭代次数必须为整数 |
| 自由裁量条件为空 | 错误 | 自由裁量条件不能为空 |
| 条件过短 | 警告 | 自由裁量条件可能存在歧义 |
| 循环变量遮蔽外部变量 | 警告 | 循环变量遮蔽了外部变量 |

---

## 管道操作 (Pipeline Operations)

管道操作提供函数式风格的集合转换。它们允许你使用管道操作符 (`|`) 链式调用 `map`、`filter` 和 `reduce` 等操作。

### 管道操作符 (Pipe Operator)

管道操作符 (`|`) 将集合传递给转换操作：

```prose
let items = ["a", "b", "c"]
let results = items | map:
  session "处理该项目"
    context: item
```

### Map (映射)

`map` 操作转换集合中的每个元素：

```prose
let articles = ["文章 1", "文章 2", "文章 3"]

let summaries = articles | map:
  session "用一句话总结这篇文章"
    context: item
```

在 `map` 主体内部，隐式变量 `item` 指向当前正在处理的元素。

### Filter (过滤)

`filter` 操作保留符合条件的元素：

```prose
let items = ["one", "two", "three", "four", "five"]

let short = items | filter:
  session "这个单词的字母是否少于或等于 4 个？回答 yes 或 no。"
    context: item
```

`filter` 主体中的 session 应返回 OpenProse 虚拟机可以解读为真/假 (truthy/falsy) 的内容（如 "yes"/"no"）。

### Reduce (归约/累加)

`reduce` 操作将元素累加成单个结果：

```prose
let ideas = ["AI 助手", "智能家居", "健康追踪器"]

let combined = ideas | reduce(summary, idea):
  session "将这个点子添加到摘要中，创建一个连贯的概念"
    context: [summary, idea]
```

`reduce` 操作需要显式的变量名：

- 第一个变量 (`summary`)：累加器 (accumulator)
- 第二个变量 (`idea`)：当前项目 (current item)

集合中的第一个项目将成为初始累加器值。

### 并行 Map (pmap) (Parallel Map - pmap)

`pmap` 操作类似于 `map`，但并发运行所有转换：

```prose
let tasks = ["任务 1", "任务 2", "任务 3"]

let results = tasks | pmap:
  session "并行处理该任务"
    context: item

session "聚合所有结果"
  context: results
```

这类似于 `parallel for`，但使用的是管道语法。

### 链式调用 (Chaining)

管道操作可以链式调用以组合复杂的转换：

```prose
let topics = ["量子计算", "区块链", "机器学习", "物联网"]

let result = topics
  | filter:
      session "这个主题是否流行？回答 yes 或 no。"
        context: item
  | map:
      session "为这个主题写一句初创公司推介语"
        context: item

session "展示初创公司推介语"
  context: result
```

操作按从左到右的顺序执行：先过滤 (filter)，再映射 (map)。

### 完整示例 (Complete Example)

```prose
# 定义一个集合
let articles = ["AI 突破", "气候解决方案", "太空探索"]

# 使用链式操作进行处理
let summaries = articles
  | filter:
      session "这个主题是否与技术相关？回答 yes 或 no。"
        context: item
  | map:
      session "写一段引人入胜的摘要"
        context: item
  | reduce(combined, summary):
      session "将此摘要合并到综合文档中"
        context: [combined, summary]

# 展示最终结果
session "格式化并展示综合后的摘要"
  context: summaries
```

### 隐式变量 (Implicit Variables)

| 操作 (Operation) | 可用变量 (Available Variables) |
| :--- | :--- |
| `map` | `item` - 当前元素 |
| `filter` | `item` - 当前元素 |
| `pmap` | `item` - 当前元素 |
| `reduce` | 显式命名：`reduce(accVar, itemVar):` |

### 执行语义 (Execution Semantics)

当 OpenProse 虚拟机遇到管道时：

1. **输入**：从输入集合开始
2. **对于每个操作**：
   - **map**：转换每个元素，产生一个新集合
   - **filter**：保留 session 返回真值的元素
   - **reduce**：将元素累加成单个值
   - **pmap**：并发转换所有元素
3. **输出**：返回最终转换后的集合/值

### 变量作用域 (Variable Scoping)

管道变量的作用域限于其操作主体内：

```prose
let item = "outer"
let items = ["a", "b"]

let results = items | map:
  # 这里的 'item' 是管道变量（遮蔽了外部变量）
  session "process"
    context: item

# 这里的 'item' 再次引用外部变量
session "use outer"
  context: item
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| 输入集合未定义 | 错误 | 集合变量未定义 |
| 管道操作符无效 | 错误 | 期望管道操作符 (map, filter, reduce, pmap) |
| reduce 缺少变量 | 错误 | 期望累加器和项目变量 |
| 管道变量遮蔽外部变量 | 警告 | 隐式/显式变量遮蔽了外部变量 |

---

## 错误处理 (Error Handling)

OpenProse 通过 `try`/`catch`/`finally` 块、`throw` 语句和 `retry`（重试）机制提供结构化的错误处理，以实现具有韧性的工作流。

### Try/Catch 块 (Try/Catch Blocks)

`try:` 块包裹可能失败的操作。`catch:` 块处理错误。

```prose
try:
  session "尝试高风险操作"
catch:
  session "从容处理错误"
```

#### 错误变量访问 (Error Variable Access)

使用 `catch as err:` 为错误处理程序捕获错误上下文：

```prose
try:
  session "调用外部 API"
catch as err:
  session "记录并处理错误"
    context: err
```

错误变量 (`err`) 包含关于出错原因的上下文信息，且仅在 `catch` 块内可访问。

### Try/Catch/Finally (Try/Catch/Finally)

无论 `try` 块成功还是失败，`finally:` 块总是会执行：

```prose
try:
  session "获取并使用资源"
catch:
  session "处理任何错误"
finally:
  session "始终清理资源"
```

#### 执行顺序 (Execution Order)

1. **Try 成功**：try 主体 → finally 主体
2. **Try 失败**：try 主体（执行至失败处） → catch 主体 → finally 主体

### Try/Finally（无 Catch） (Try/Finally - No Catch)

对于不需要错误处理的清理工作，使用 `try`/`finally`：

```prose
try:
  session "开启连接并执行工作"
finally:
  session "关闭连接"
```

### Throw 语句 (Throw Statement)

`throw` 语句引发或重新引发 (re-raise) 错误。

#### Rethrow (重新引发)

在 `catch` 块内部，不带参数的 `throw` 会将捕获的错误重新引发给外部处理程序：

```prose
try:
  try:
    session "内部操作"
  catch:
    session "部分处理"
    throw  # 重新引发给外部处理程序
catch:
  session "处理重新引发的错误"
```

#### 带消息引发 (Throw with Message)

引发一个带有自定义消息的新错误：

```prose
session "检查前置条件"
throw "不满足前置条件"
```

### 嵌套错误处理 (Nested Error Handling)

`try` 块可以嵌套。除非显式地 `throw`，否则内部 `catch` 块不会触发外部处理程序：

```prose
try:
  session "外部操作"
  try:
    session "内部高风险操作"
  catch:
    session "处理内部错误"  # 外部 catch 不会运行
  session "继续外部操作"
catch:
  session "仅处理外部错误"
```

### 并行执行中的错误处理 (Error Handling in Parallel)

每个并行分支可以有自己的错误处理：

```prose
parallel:
  try:
    session "分支 A 可能会失败"
  catch:
    session "恢复分支 A"
  try:
    session "分支 B 可能会失败"
  catch:
    session "恢复分支 B"

session "带着恢复后的结果继续"
```

这与 `on-fail:` 策略不同，后者控制发生未处理错误时的行为。

### Retry 属性 (Retry Property)

`retry:` 属性使 session 在失败时自动重试：

```prose
session "调用不稳定的 API"
  retry: 3
```

#### 带退避策略的重试 (Retry with Backoff)

添加 `backoff:` 以控制重试间的延迟：

```prose
session "受频率限制的 API"
  retry: 5
  backoff: exponential
```

**退避策略 (Backoff Strategies)：**

| 策略 (Strategy) | 行为 (Behavior) |
| :--- | :--- |
| `none` | 立即重试（默认） |
| `linear` | 重试间固定延迟 |
| `exponential` | 指数级增加延迟 (1s, 2s, 4s, 8s...) |

#### 带上下文的重试 (Retry with Context)

`retry` 可与其他 session 属性配合使用：

```prose
let data = session "获取输入"
session "处理数据"
  context: data
  retry: 3
  backoff: linear
```

### 模式组合 (Combining Patterns)

`retry` 和 `try`/`catch` 配合使用可获得最大韧性：

```prose
try:
  session "调用外部服务"
    retry: 3
    backoff: exponential
catch:
  session "所有重试均失败，使用备选方案"
```

### 验证规则 (Validation Rules)

| 检查项 (Check) | 严重程度 (Severity) | 消息 (Message) |
| :--- | :--- | :--- |
| try 缺少 catch 或 finally | 错误 | try 块必须至少具有 "catch:" 或 "finally:" |
| 错误变量遮蔽外部变量 | 警告 | 错误变量遮蔽了外部变量 |
| throw 消息为空 | 警告 | throw 消息为空 |
| 重试次数非正数 | 错误 | 重试次数必须为正数 |
| 重试次数不是整数 | 错误 | 重试次数必须为整数 |
| 重试次数过高 (>10) | 警告 | 重试次数异常高 |
| 退避策略无效 | 错误 | 必须是 none, linear 或 exponential |
| 在 agent 定义中使用 retry | 警告 | retry 属性仅在 session 语句中有效 |

### 语法参考 (Syntax Reference)

```
try_block ::= "try" ":" NEWLINE INDENT statement+ DEDENT
              [catch_block]
              [finally_block]

catch_block ::= "catch" ["as" identifier] ":" NEWLINE INDENT statement+ DEDENT

finally_block ::= "finally" ":" NEWLINE INDENT statement+ DEDENT

throw_statement ::= "throw" [string_literal]

retry_property ::= "retry" ":" number_literal

backoff_property ::= "backoff" ":" ( "none" | "linear" | "exponential" )
```

---

### 执行语义 (Execution Semantics)

当错误发生时：

1. **捕获 (Capture)**：如果 `try` 块中的任何语句抛出错误，执行将立即跳至 `catch` 块。
2. **上下文绑定 (Context Binding)**：如果提供了 `as` 子句，错误对象将绑定到指定的标识符（在 `catch` 块作用域内）。
3. **恢复 (Recovery)**：执行 `catch` 块中的语句以处理错误。
4. **清理 (Cleanup)**：无论是否发生错误，最后都会执行 `finally` 块（如果存在）。
5. **继续 (Continue)**：如果错误被捕获（且没有重新抛出），则继续执行 `try/catch/finally` 结构之后的下一条语句。

### 验证规则 (Validation Rules)

| 检查项 | 严重程度 | 消息 |
| :--- | :--- | :--- |
| 缺少 catch 或 finally | 错误 | try 块必须后跟 catch 或 finally 块 |
| 无效的错误变量名 | 错误 | catch 标识符无效 |
| 空的 try 块 | 警告 | try 块为空 |
| 重复的 finally | 错误 | 仅允许一个 finally 块 |

### 语法参考 (Syntax Reference)

```
try_block ::= "try" ":" NEWLINE INDENT statement+ DEDENT catch_block? finally_block?

catch_block ::= "catch" [ "as" IDENTIFIER ] ":" NEWLINE INDENT statement+ DEDENT

finally_block ::= "finally" ":" NEWLINE INDENT statement+ DEDENT
```

---

## 选择块 (Choice Blocks)

`choice` 块允许基于 AI 评估的准则在多个命名的选项 (Options) 之间进行分支。

### 语法

```prose
choice **criteria**:
  option "Label A":
    statements...
  option "Label B":
    statements...
```

### 准则 (Criteria)

准则包裹在自由裁量标记 (`**...**`) 中，由 OpenProse 虚拟机 (VM) 进行评估，以选择执行哪个选项：

```prose
choice **针对当前情况的最佳方法**:
  option "快速修复":
    session "应用一个快速的临时修复"
  option "完整重构":
    session "执行完整的代码重构"
```

### 多行准则 (Multi-line Criteria)

对于复杂的准则，使用三星号标记：

```prose
choice ***
  根据当前项目约束
  和时间线要求
  哪种策略最合适
***:
  option "MVP 方法":
    session "构建最小可行产品"
  option "全量功能集":
    session "构建完整的功能集"
```

### 示例 (Examples)

#### 简单选择

```prose
let analysis = session "分析代码质量"

choice **分析中发现的问题严重程度**:
  option "关键 (Critical)":
    session "停止部署并修复关键问题"
      context: analysis
  option "次要 (Minor)":
    session "记录问题稍后处理并继续"
      context: analysis
  option "无 (None)":
    session "继续部署"
```

#### 每个选项包含多条语句的选择

```prose
choice **用户的经验水平**:
  option "初学者":
    session "首先解释基本概念"
    session "提供分步指导"
    session "包含有用的提示和警告"
  option "专家":
    session "提供简洁的技术摘要"
    session "包含高级配置选项"
```

#### 嵌套选择

```prose
choice **请求类型**:
  option "缺陷报告 (Bug report)":
    choice **缺陷严重程度**:
      option "关键 (Critical)":
        session "立即上报"
      option "普通 (Normal)":
        session "添加到冲刺待办列表"
  option "功能请求 (Feature request)":
    session "添加到功能待办列表"
```

### 执行语义 (Execution Semantics)

当 OpenProse 虚拟机遇到 `choice` 块时：

1. **评估准则 (Evaluate Criteria)**：在当前上下文中解释自由裁量准则。
2. **选择选项 (Select Option)**：选择最合适的命名选项。
3. **执行 (Execute)**：运行所选选项主体中的所有语句。
4. **继续 (Continue)**：继续执行选择块之后的下一条语句。

每个 `choice` 块仅执行一个选项。

### 验证规则 (Validation Rules)

| 检查项 | 严重程度 | 消息 |
| :--- | :--- | :--- |
| 选择块没有选项 | 错误 | choice 块必须至少有一个选项 |
| 准则为空 | 错误 | choice 准则不能为空 |
| 重复的选项标签 | 警告 | 重复的选项标签 |
| 选项主体为空 | 警告 | 选项的主体为空 |

### 语法参考 (Syntax Reference)

```
choice_block ::= "choice" discretion ":" NEWLINE INDENT option+ DEDENT

option ::= "option" string ":" NEWLINE INDENT statement+ DEDENT

discretion ::= "**" text "**" | "***" text "***"
```

---

## 条件语句 (Conditional Statements)

`if/elif/else` 语句使用自由裁量标记通过 AI 评估条件，从而提供条件分支。

### If 语句

```prose
if **condition**:
  statements...
```

### If/Else

```prose
if **condition**:
  statements...
else:
  statements...
```

### If/Elif/Else

```prose
if **first condition**:
  statements...
elif **second condition**:
  statements...
elif **third condition**:
  statements...
else:
  statements...
```

### 自由裁量条件 (Discretion Conditions)

条件包裹在自由裁量标记 (`**...**`) 中，供 AI 评估：

```prose
let analysis = session "分析代码库"

if **代码存在安全漏洞**:
  session "立即修复安全问题"
    context: analysis
elif **代码存在性能问题**:
  session "优化性能瓶颈"
    context: analysis
else:
  session "进行正常审查"
    context: analysis
```

### 多行条件 (Multi-line Conditions)

对于复杂的条件，使用三星号标记：

```prose
if ***
  测试套件通过
  且代码覆盖率在 80% 以上
  且没有 lint 错误
***:
  session "部署到生产环境"
else:
  session "在部署前修复问题"
```

### 示例 (Examples)

#### 简单 If

```prose
session "检查系统健康状况"

if **系统健康**:
  session "继续正常运行"
```

#### If/Else

```prose
let review = session "审查拉取请求 (PR)"

if **代码更改安全且经过充分测试**:
  session "批准并合并 PR"
    context: review
else:
  session "请求更改"
    context: review
```

#### 多个 Elif

```prose
let status = session "检查项目状态"

if **项目按计划进行**:
  session "按计划继续"
elif **项目稍有延迟**:
  session "调整时间线并沟通"
elif **项目严重延迟**:
  session "上报管理层"
  session "创建恢复计划"
else:
  session "评估项目的可行性"
```

#### 嵌套条件

```prose
if **请求已通过身份验证**:
  if **用户拥有管理员权限**:
    session "处理管理员请求"
  else:
    session "处理标准用户请求"
else:
  session "返回身份验证错误"
```

### 与其他结构组合

#### 与 Try/Catch 组合

```prose
try:
  session "尝试操作"
  if **操作部分成功**:
    session "完成剩余步骤"
catch as err:
  if **错误可恢复**:
    session "应用恢复程序"
      context: err
  else:
    throw "不可恢复的错误"
```

#### 与循环组合

```prose
loop until **任务完成** (max: 10):
  session "处理任务"
  if **遇到阻塞器 (Blocker)**:
    session "解决阻塞器"
```

### 执行语义 (Execution Semantics)

当 OpenProse 虚拟机遇到 `if` 语句时：

1. **评估条件 (Evaluate Condition)**：解释第一个自由裁量条件。
2. **如果为真 (If True)**：执行对应的 then-主体，并跳过剩余子句。
3. **如果为假 (If False)**：按顺序检查每个 `elif` 条件。
4. **Elif 匹配 (Elif Match)**：执行该 `elif` 的主体并跳过后续子句。
5. **无匹配 (No Match)**：执行 `else` 主体（如果存在）。
6. **继续 (Continue)**：继续执行下一条语句。

### 验证规则 (Validation Rules)

| 检查项 | 严重程度 | 消息 |
| :--- | :--- | :--- |
| 条件为空 | 错误 | if/elif 条件不能为空 |
| elif 缺少 if | 错误 | elif 必须跟随 if |
| else 缺少 if | 错误 | else 必须跟随 if 或 elif |
| 多个 else | 错误 | 仅允许一个 else 子句 |
| 主体为空 | 警告 | 条件对应的主体为空 |

### 语法参考 (Syntax Reference)

```
if_statement ::= "if" discretion ":" NEWLINE INDENT statement+ DEDENT
                 elif_clause*
                 [else_clause]

elif_clause ::= "elif" discretion ":" NEWLINE INDENT statement+ DEDENT

else_clause ::= "else" ":" NEWLINE INDENT statement+ DEDENT

discretion ::= "**" text "**" | "***" text "***"
```

---

## 执行模型 (Execution Model)

OpenProse 使用两阶段执行模型。

### 第一阶段：编译（静态） (Phase 1: Compilation (Static))

编译阶段处理确定性的预处理：

1. **解析 (Parse)**：将源代码转换为抽象语法树 (AST)。
2. **验证 (Validate)**：检查语法和语义错误。
3. **展开 (Expand)**：标准化语法糖（如果已实现）。
4. **输出 (Output)**：生成规范程序。

### 第二阶段：运行时（智能） (Phase 2: Runtime (Intelligent))

OpenProse 虚拟机执行编译后的程序：

1. **加载 (Load)**：接收编译后的程序。
2. **收集代理 (Collect Agents)**：注册所有代理定义。
3. **执行 (Execute)**：按顺序处理每条语句。
4. **派生 (Spawn)**：根据解析后的配置创建子代理。
5. **协调 (Coordinate)**：管理会话之间的上下文传递。

### OpenProse 虚拟机行为 (OpenProse VM Behavior)

| 维度 | 行为 |
| :--- | :--- |
| 执行顺序 | **严格** - 完全遵循程序顺序 |
| 会话创建 | **严格** - 按照程序规范创建 |
| 代理解析 | **严格** - 确定性地合并属性 |
| 上下文传递 | **智能** - 根据需要进行摘要/转换 |
| 完成检测 | **智能** - 确定会话何时“完成” |

### 状态管理 (State Management)

对于当前的实现，状态是在上下文（对话历史）中跟踪的：

| 状态类型 | 跟踪方法 |
| :--- | :--- |
| 代理定义 | 在程序启动时收集 |
| 执行流 | 隐式推理（“已完成 X，现在执行 Y”） |
| 会话输出 | 保留在对话历史中 |
| 程序中的位置 | 由 OpenProse 虚拟机跟踪 |

---

## 验证规则 (Validation Rules)

验证器在执行前检查程序中的错误和警告。

### 错误（阻塞执行） (Errors (Block Execution))

| 代码 | 描述 |
| :--- | :--- |
| E001 | 未终止的字符串字面量 |
| E002 | 字符串中存在未知的转义序列 |
| E003 | session 缺少提示词 (Prompt) 或代理 |
| E004 | 意外的标记 (Token) |
| E005 | 无效语法 |
| E006 | 重复的代理定义 |
| E007 | 未定义的代理引用 |
| E008 | 无效的模型值 |
| E009 | 重复的属性 |
| E010 | 重复的 use 语句 |
| E011 | use 路径为空 |
| E012 | 无效的 use 路径格式 |
| E013 | skills 必须是数组 |
| E014 | 技能名称必须是字符串 |
| E015 | permissions 必须是块 |
| E016 | 权限模式必须是字符串 |
| E017 | `resume:` 要求持久化代理 |
| E018 | `resume:` 但不存在现有记忆 |
| E019 | 重复的变量名（扁平命名空间） |
| E020 | 输入名称为空 |
| E021 | 重复的 input 声明 |
| E022 | 在可执行语句之后出现 input |
| E023 | 输出名称为空 |
| E024 | 重复的 output 声明 |
| E025 | 调用中存在未知的程序 |
| E026 | 缺少必需的输入 |
| E027 | 调用中存在未知的输入名称 |
| E028 | 未知的输出属性访问 |

### 警告（非阻塞） (Warnings (Non-blocking))

| 代码 | 描述 |
| :--- | :--- |
| W001 | session 提示词为空 |
| W002 | session 提示词仅包含空白字符 |
| W003 | session 提示词超过 10,000 个字符 |
| W004 | prompt 属性为空 |
| W005 | 未知的属性名称 |
| W006 | 未知的导入源格式 |
| W007 | 技能未导入 |
| W008 | 未知的权限类型 |
| W009 | 未知的权限值 |
| W010 | skills 数组为空 |
| W011 | 在具有现有记忆的持久化代理上使用 `session:` |

### 错误消息格式 (Error Message Format)

错误包含位置信息：

```
Error at line 5, column 12: Unterminated string literal
  session "Hello
          ^
```

---

## 示例 (Examples)

### 极简程序

```prose
session "Hello world"
```

### 带有代理的研究流水线

```prose
# 定义专业代理
agent researcher:
  model: sonnet
  prompt: "你是一名研究助理"

agent writer:
  model: opus
  prompt: "你是一名技术作家"

# 执行工作流
session: researcher
  prompt: "研究量子计算的最新进展"

session: writer
  prompt: "撰写研究结果的摘要"
```

### 代码审查工作流

```prose
agent reviewer:
  model: sonnet
  prompt: "你是一位资深代码审查员"

session: reviewer
  prompt: "阅读 src/ 中的代码并识别潜在的缺陷"

session: reviewer
  prompt: "为发现的每个缺陷建议修复方案"

session: reviewer
  prompt: "创建所需的所有更改的摘要"
```

### 带有模型覆盖的多步任务

```prose
agent analyst:
  model: haiku
  prompt: "你擅长快速分析数据"

# 快速初步分析
session: analyst
  prompt: "扫描数据中的明显模式"

# 使用更强大的模型进行详细分析
session: analyst
  model: opus
  prompt: "对发现的模式进行深度分析"
```

### 用于文档的注释

```prose
# 项目：季度报告生成器
# 作者：团队负责人
# 日期：2024-01-01

agent data-collector:
  model: sonnet
  prompt: "你负责收集并组织数据"

agent analyst:
  model: opus
  prompt: "你负责分析数据并产生见解"

# 第一步：收集数据
session: data-collector
  prompt: "收集上一季度的所有销售数据"

# 第二步：分析
session: analyst
  prompt: "对收集的数据进行趋势分析"

# 第三步：报告生成
session: analyst
  prompt: "生成带有图表的格式化季度报告"
```

### 带有技能和权限的工作流

```prose
# 导入外部程序
use "@anthropic/web-search"
use "@anthropic/file-writer" as file-writer

# 定义一个安全的研究代理
agent researcher:
  model: sonnet
  prompt: "你是一名研究助理"
  skills: ["web-search"]
  permissions:
    read: ["*.md", "*.txt"]
    bash: deny

# 定义一个作家代理
agent writer:
  model: opus
  prompt: "你负责创建文档"
  skills: ["file-writer"]
  permissions:
    write: ["docs/"]
    bash: deny

# 执行工作流
session: researcher
  prompt: "研究 AI 安全主题"

session: writer
  prompt: "撰写摘要文档"
```

---

## 未来特性 (Future Features)

Tier 12 之前的所有核心特性均已实现。潜在的未来增强功能：

### Tier 13: 扩展特性

- 带有返回值的自定义函数
- 用于代码组织的模块系统
- 用于验证的类型注解
- 用于高级并发的 async/await 模式

### Tier 14: 工具链

- 语言服务器协议 (LSP) 支持
- VS Code 扩展
- 交互式调试器
- 性能分析 (Profiling)

---

## 语法文法 (已实现) (Syntax Grammar (Implemented))

```
program     → statement* EOF
statement   → useStatement | inputDecl | agentDef | session | resumeStmt
            | letBinding | constBinding | assignment | outputBinding
            | parallelBlock | repeatBlock | forEachBlock | loopBlock
            | tryBlock | choiceBlock | ifStatement | doBlock | blockDef
            | throwStatement | comment

# 程序组合 (Program Composition)
useStatement → "use" string ( "as" IDENTIFIER )?
inputDecl   → "input" IDENTIFIER ":" string
outputBinding → "output" IDENTIFIER "=" expression
programCall → IDENTIFIER "(" ( IDENTIFIER ":" expression )* ")"

# 定义 (Definitions)
agentDef    → "agent" IDENTIFIER ":" NEWLINE INDENT agentProperty* DEDENT
agentProperty → "model:" ( "sonnet" | "opus" | "haiku" )
               | "prompt:" string
               | "persist:" ( "true" | "project" | string )
               | "context:" ( IDENTIFIER | array | objectContext )
               | "retry:" NUMBER
               | "backoff:" ( "none" | "linear" | "exponential" )
               | "skills:" "[" string* "]"
               | "permissions:" NEWLINE INDENT permission* DEDENT
blockDef    → "block" IDENTIFIER params? ":" NEWLINE INDENT statement* DEDENT
params      → "(" IDENTIFIER ( "," IDENTIFIER )* ")"

# 控制流 (Control Flow)
parallelBlock → "parallel" parallelMods? ":" NEWLINE INDENT parallelBranch* DEDENT
parallelMods  → "(" ( joinStrategy | onFail | countMod ) ( "," ( joinStrategy | onFail | countMod ) )* ")"
joinStrategy  → string                              # "all" | "first" | "any"
onFail        → "on-fail" ":" string                # "fail-fast" | "continue" | "ignore"
countMod      → "count" ":" NUMBER                  # 仅在 "any" 时有效
parallelBranch → ( IDENTIFIER "=" )? statement

# 循环 (Loops)
repeatBlock → "repeat" NUMBER ( "as" IDENTIFIER )? ":" NEWLINE INDENT statement* DEDENT
forEachBlock → "parallel"? "for" IDENTIFIER ( "," IDENTIFIER )? "in" collection ":" NEWLINE INDENT statement* DEDENT
loopBlock   → "loop" ( ( "until" | "while" ) discretion )? loopMods? ( "as" IDENTIFIER )? ":" NEWLINE INDENT statement* DEDENT
loopMods    → "(" "max" ":" NUMBER ")"

# 错误处理 (Error Handling)
tryBlock    → "try" ":" NEWLINE INDENT statement+ DEDENT catchBlock? finallyBlock?
catchBlock  → "catch" ( "as" IDENTIFIER )? ":" NEWLINE INDENT statement+ DEDENT
finallyBlock → "finally" ":" NEWLINE INDENT statement+ DEDENT
throwStatement → "throw" string?

# 条件语句 (Conditionals)
choiceBlock → "choice" discretion ":" NEWLINE INDENT choiceOption+ DEDENT
choiceOption → "option" string ":" NEWLINE INDENT statement+ DEDENT
ifStatement → "if" discretion ":" NEWLINE INDENT statement+ DEDENT elifClause* elseClause?
elifClause  → "elif" discretion ":" NEWLINE INDENT statement+ DEDENT
elseClause  → "else" ":" NEWLINE INDENT statement+ DEDENT

# 组合 (Composition)
doBlock     → "do" ( ":" NEWLINE INDENT statement* DEDENT | IDENTIFIER args? )
args        → "(" expression ( "," expression )* ")"
arrowExpr   → session ( "->" session )+

# 会话 (Sessions)
session     → "session" ( string | ":" IDENTIFIER | IDENTIFIER ":" IDENTIFIER )
               ( NEWLINE INDENT sessionProperty* DEDENT )?
resumeStmt  → "resume" ":" IDENTIFIER ( NEWLINE INDENT sessionProperty* DEDENT )?
sessionProperty → "model:" ( "sonnet" | "opus" | "haiku" )
                 | "prompt:" string
                 | "context:" ( IDENTIFIER | array | objectContext )
                 | "retry:" NUMBER
                 | "backoff:" ( "none" | "linear" | "exponential" )

# 绑定 (Bindings)
letBinding  → "let" IDENTIFIER "=" expression
constBinding → "const" IDENTIFIER "=" expression
assignment  → IDENTIFIER "=" expression

# 表达式 (Expressions)
expression  → session | doBlock | parallelBlock | repeatBlock | forEachBlock
            | loopBlock | arrowExpr | pipeExpr | programCall | string | IDENTIFIER | array | objectContext

# 管道 (Pipelines)
pipeExpr    → ( IDENTIFIER | array ) ( "|" pipeOp )+
pipeOp      → ( "map" | "filter" | "pmap" ) ":" NEWLINE INDENT statement* DEDENT
            | "reduce" "(" IDENTIFIER "," IDENTIFIER ")" ":" NEWLINE INDENT statement* DEDENT

# 属性 (Properties)
property    → ( "model" | "prompt" | "context" | "retry" | "backoff" | IDENTIFIER )
            ":" ( IDENTIFIER | string | array | objectContext | NUMBER )

# 原语 (Primitives)
discretion  → "**" text "**" | "***" text "***"
collection  → IDENTIFIER | array
array       → "[" ( expression ( "," expression )* )? "]"
objectContext → "{" ( IDENTIFIER ( "," IDENTIFIER )* )? "}"
comment     → "#" text NEWLINE

# 字符串 (Strings)
string      → singleString | tripleString | interpolatedString
singleString → '"' character* '"'
tripleString → '"""' ( character | NEWLINE )* '"""'
interpolatedString → 包含 "{" IDENTIFIER "}" 的字符串
character   → escape | 非引号字符
escape      → "\\" | "\"" | "\n" | "\t"
```

---

## 编译器 API (Compiler API)

当用户调用 `/prose-compile` 或要求你编译一个 `.prose` 文件时：

1. **完整阅读此文档** (`compiler.md`) 以了解所有语法和验证规则。
2. **解析 (Parse)**：根据语法文法解析程序。
3. **验证 (Validate)**：检查语法正确性、语义有效性和自洽性。
4. **转换 (Transform)**：转换为规范形式（展开语法糖，标准化结构）。
5. **输出 (Output)**：输出编译后的程序，或报告带有行号的错误/警告。

对于不经过编译的直接解释执行，请阅读 `prose.md` 并按照会话语句 (Session Statement) 部分的描述执行语句。


