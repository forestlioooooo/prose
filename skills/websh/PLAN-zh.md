# websh: 一个用于 Web 的 shell

## 愿景

一个将 URL 视为路径，将 DOM 视为文件系统的 shell。你可以导航到某个 URL，然后通过命令对缓存的页面内容进行操作——即时、本地化且无需重新获取。

```
websh> cd https://news.ycombinator.com
websh> ls                    # 列出链接
websh> grep "AI" | head 5    # 过滤内容
websh> cat .title            # 使用 CSS 选择器提取内容
websh> follow 3              # 导航到第 3 个链接
```

Web 变成了一个你可以使用熟悉的命令进行探索的计算环境。

---

## 设计原则

1. **一次获取，本地操作** — `cd` 负责获取（fetch）和缓存（cache）；所有其他命令均作用于缓存。
2. **扁平化缓存结构** — URL 转换为扁平的文件名（斜杠 → 连字符）。
3. **你就是这个 shell** — Claude 体现为 websh，维护会话状态。
4. **可组合的原语** — 可以通过管道连接的小型命令。
5. **熟悉的交互体验 (UX)** — 适配 Web 语义的类 Unix 命令。

---

## 目录结构

### 技能文件 (位于此目录中)

```
prose/skills/websh/
├── SKILL.md              # 激活触发器，命令路由
├── PLAN.md               # 本文件
├── shell.md              # 核心 shell 语义（你就是 websh）
├── commands.md           # 命令参考（ls, cat, grep 等）
├── state/
│   └── cache.md          # 缓存管理和格式说明
└── help.md               # 用户帮助和示例
```

### 用户状态 (位于工作目录中)

```
.websh/
├── session.md            # 当前会话状态（pwd, 历史记录）
├── cache/
│   ├── {url-slug}.html       # 原始 HTML
│   ├── {url-slug}.parsed.md  # 迭代提取的内容（由 haiku 生成）
│   └── index.md              # URL → slug 映射，获取时间
├── history.md            # 命令历史记录
└── bookmarks.md          # 已保存的位置（书签）
```

### 缓存文件名约定

URL 被扁平化为易读的标识符（slug）：
- `https://news.ycombinator.com` → `news-ycombinator-com`
- `https://x.com/deepfates/status/123` → `x-com-deepfates-status-123`
- `https://techcrunch.com/2024/06/25/article-name/` → `techcrunch-com-2024-06-25-article-name`

每个缓存的 URL 包含两个文件：
- `{slug}.html` — 原始 HTML
- `{slug}.parsed.md` — 迭代提取的内容

`index.md` 负责将完整 URL 映射到标识符，并跟踪获取/提取状态。

---

## 核心命令

| 命令 | 描述 | 作用对象 |
|---------|-------------|-------------|
| `cd <url>` | 导航到 URL，获取并提取内容（异步） | 网络 → 缓存 → Haiku 提取 |
| `pwd` | 显示当前 URL | 会话 (Session) |
| `ls [selector]` | 列出链接或元素 | 缓存 |
| `cat <selector>` | 提取文本内容 | 缓存 |
| `grep <pattern>` | 通过文本/正则表达式过滤 | 缓存 |
| `head <n>` / `tail <n>` | 切片处理结果 | 管道 (Pipe) |
| `follow <n\|text>` | 导航到第 n 个链接或匹配的文本 | 缓存 → 网络 |
| `back` | 返回上一个 URL | 会话历史记录 |
| `refresh` | 重新获取当前 URL | 网络 → 缓存 |
| `stat` | 显示页面元数据（标题、链接数等） | 缓存 |
| `save <path>` | 将当前页面保存为文件 | 缓存 → 文件系统 |
| `history` | 显示导航历史记录 | 会话 |
| `bookmarks` | 列出已保存的位置 | 用户状态 |
| `bookmark [name]` | 保存当前 URL | 用户状态 |

### 计划中的扩展命令

| 命令 | 描述 |
|---------|-------------|
| `diff <url1> <url2>` | 比较两个页面 |
| `watch <url>` | 轮询检查变化 |
| `form <selector>` | 与表单交互 |
| `click <selector>` | 模拟点击（针对重度使用 JS 的站点） |
| `mount <api> <path>` | 将 API 挂载为虚拟目录 |

---

## `cd` 流程：获取 + 提取

当用户运行 `cd <url>` 时，websh 执行两阶段操作：

### 阶段 1：获取 (Fetch - 同步)

```
cd https://news.ycombinator.com
   │
   ├─→ WebFetch 该 URL
   ├─→ 将原始 HTML 保存到 .websh/cache/{hash}.html
   ├─→ 在 index.json 中更新 URL → hash 映射
   └─→ 在 session.md 中更新新的 pwd
```

用户看到：`fetching... done`

### 阶段 2：提取 (Extract - 异步 haiku 子代理，迭代进行)

获取完成后立即生成一个后台 haiku 代理，通过**循环**构建丰富的 markdown 提取内容：

```
Task({
  description: "websh: iterative page extraction",
  prompt: "<extraction prompt - 详见下文>",
  subagent_type: "general-purpose",
  model: "haiku",
  run_in_background: true
})
```

haiku 代理执行**迭代智能解析**：

```
loop until **提取内容足够详尽**:
  1. 读取原始 .html 文件
  2. 读取当前的 .parsed.md 文件（如果存在）
  3. 识别缺失或可以更丰富的内容
  4. 使用新发现的内容追加/更新 .parsed.md
  5. 重复直到收益递减
```

每次处理（pass）侧重不同方面：
- **第 1 次 (Pass 1)**：基础结构（标题、主标题、链接清单）
- **第 2 次 (Pass 2)**：内容提取（正文文本、评论、关键引用）
- **第 3 次 (Pass 3)**：元数据和上下文（作者、日期、相关链接、站点结构）
- **第 4 次及以后**：边缘情况、遗漏内容、清理工作

**输出文件：`.websh/cache/{hash}.parsed.md`**

```markdown
# https://news.ycombinator.com

获取时间: 2026-01-24T10:30:00Z
提取进度: 分 3 次传递

## 摘要

Hacker News 首页。技术新闻聚合器，包含用户提交的链接和讨论。
当前可见 30 个故事，包含 Show HN、技术文章和行业新闻。

## 链接

| # | 标题 | 点数 | 评论数 |
|---|-------|--------|----------|
| 0 | Show HN: I built a tool for... | 142 | 87 |
| 1 | The State of AI in 2026 | 891 | 432 |
| 2 | Why Rust is eating the world | 234 | 156 |
...

## 导航

- [new](/newest) - 最新提交
- [past](/front) - 往期首页
- [comments](/newcomments) - 最新评论
- [ask](/ask) - Ask HN
- [show](/show) - Show HN
- [jobs](/jobs) - 招聘

## 内容模式

这是一个链接聚合器。每个故事包含：
- 标题 (类名: .titleline)
- 点数和提交者 (类名: .score, .hnuser)
- 评论数 (链接至 /item?id=...)
- 括号内的域名

## 原始文本片段

### 热门故事
1. "Show HN: I built a tool for..." - 142 点, 87 条评论
2. "The State of AI in 2026" - 891 点, 432 条评论
...

## 表单

- 搜索: input[name=q] 位于 /hn.algolia.com
- 登录: /login (用户名, 密码)

## 备注

- 首页无图片（纯文本设计）
- 移动端友好，CSS 极简
- 故事刷新频繁
```

**用户体验：**

```
news.ycombinator.com> cd https://example.com

fetching... done
extracting... (pass 1)

example.com> ls

# 显示目前已提取的内容
# 代理在后台继续提取
# 随着处理轮次的完成，后续命令将获得更丰富的数据
```

### 为什么采用迭代方式？

- **渐进式丰富**：第一轮快速提供基础信息，后续轮次增加深度。
- **智能聚焦**：Haiku 根据页面类型决定提取重点。
- **人类可读输出**：Markdown 格式易于检查、调试且非常实用。
- **优雅降级**：第一轮完成后命令即可运行，随着轮次增加效果不断提升。
- **站点感知**：Haiku 能识别模式（如 HN 故事、推文、博客文章）并进行适配。

### 为什么选择 haiku？

- **速度快**：每轮处理都能快速完成。
- **成本低**：多轮处理依然经济。
- **并行化**：不会阻塞用户的命令操作。
- **智能化**：能根据内容类型调整提取策略。

---

## 状态管理

### 会话状态 (`session.md`)

跟踪当前的 shell 会话：

```markdown
# websh 会话

pwd: https://news.ycombinator.com
启动时间: 2026-01-24T10:30:00Z

## 历史记录
1. cd https://news.ycombinator.com
2. ls
3. grep "AI"

## 导航栈
- https://news.ycombinator.com (当前)
```

### 缓存格式

每个缓存页面包含两个文件：

**`{hash}.html`** — 获取的原始 HTML（用于参考和选择器查询）

**`{hash}.parsed.md`** — 智能提取的内容（由 haiku 迭代写入）：

```markdown
# https://news.ycombinator.com

获取时间: 2026-01-24T10:30:00Z
处理轮次: 3
状态: 已完成

## 摘要

Hacker News 首页。包含 30 个故事的技术新闻聚合器。
涵盖 Show HN 项目、技术深度解析和行业新闻。

## 链接

| # | 标题 | 链接 (Href) | 元数据 |
|---|-------|------|------|
| 0 | Show HN: I built... | /item?id=123 | 142 点, 87 条评论 |
| 1 | The State of AI | /item?id=456 | 891 点, 432 条评论 |
...

## 内容

### 主要内容
(提取并清理后的文章正文)

### 评论
(如果适用)

### 侧边栏 / 导航
- [new](/newest)
- [past](/front)
...

## 结构

页面类型: 链接聚合器
关键选择器:
- .titleline → 故事标题
- .score → 点数
- .hnuser → 用户名

## 表单

### 登录 (/login)
- username (文本)
- password (密码)

## 媒体资源

(该页面无媒体资源)

## 元数据

- og:title: Hacker News
- description: 程序员的新闻站

## 提取备注

Pass 1: 基础结构, 发现 30 个链接
Pass 2: 提取元数据, 识别页面类型
Pass 3: 清理内容, 记录模式
```

Markdown 格式的优点：
- **人类可读**：你可以直接 `cat` 它来理解页面。
- **对 grep 友好**：`grep "AI"` 等命令可以自然地工作。
- **迭代构建**：每一轮处理都会添加/完善各部分内容。
- **站点感知**：Haiku 根据内容类型调整结构。

`ls`, `grep`, `cat` 等命令为了速度会读取 `.json` 文件。`.html` 文件可用于基于选择器的提取。

---

## shell 体现模式 (Shell Embodiment Pattern)

遵循 OpenProse VM 模式，websh 采用“你就是这个 shell”的方法：

```markdown
# 摘自 shell.md

你是 websh——一个用于导航和查询 Web 的 shell。

当你收到命令时：
1. 使用命令语法进行解析
2. 检查是否需要网络（cd, refresh, follow）或作用于缓存
3. 对于缓存操作，从 .websh/cache/ 读取内容
4. 更新会话状态
5. 以 shell 格式返回输出

你维护以下内容：
- 当前工作 URL (pwd)
- 导航历史 (后退栈)
- 命令历史记录
- 缓存索引

你的提示符格式：
{domain}>

示例：
news.ycombinator.com> ls
```

---

## 待创建的文件

### 阶段 1：核心 shell

1. **SKILL.md** — 激活触发器，命令路由
   - 激活条件：`websh`、`web shell`、在 shell 语境下的 URL
   - 路由至 shell.md 进行执行

2. **shell.md** — shell 语义
   - 体现指令
   - 命令解析
   - 状态管理
   - 输出格式化

3. **commands.md** — 命令参考
   - 每个命令的详细语法
   - 示例
   - 管道行为

4. **state/cache.md** — 缓存管理
   - 获取与存储
   - **迭代提取提示词**（驱动 haiku 循环的提示词）
   - 索引管理
   - 优雅降级（命令在提取完成前即可运行）
   - 过期与刷新

5. **help.md** — 用户文档
   - 快速入门
   - 命令速查表
   - 示例

### 阶段 2：扩展功能

- 表单交互
- JavaScript 渲染的页面（如果可用，通过浏览器工具实现）
- API 挂载
- Diff/watch 命令

---

## 示例会话

```
$ websh

┌─────────────────────────────────────┐
│          ◇ websh ◇                  │
│     A shell for the web             │
└─────────────────────────────────────┘

~> cd https://news.ycombinator.com

fetching... cached
navigated to news.ycombinator.com

news.ycombinator.com> ls | head 5
[0] Show HN: I built a tool for...
[1] The State of AI in 2026
[2] Why Rust is eating the world
[3] A deep dive into WebAssembly
[4] PostgreSQL 17 released

news.ycombinator.com> grep "AI"
[1] The State of AI in 2026
[7] AI agents are coming for your job
[12] OpenAI announces GPT-5

news.ycombinator.com> follow 1

fetching... cached
navigated to news.ycombinator.com/item?id=...

news.ycombinator.com/item> cat .title
The State of AI in 2026

news.ycombinator.com/item> cat .comment | head 3
[0] Great article, but I disagree with...
[1] This matches what I've seen at...
[2] The author missed the point about...

news.ycombinator.com/item> back

news.ycombinator.com> bookmark hn

Bookmarked: hn → https://news.ycombinator.com

news.ycombinator.com> stat
URL:      https://news.ycombinator.com
标题:     Hacker News
获取时间: 2026-01-24T10:30:00Z (5 分钟前)
链接数:   30
大小:     45 KB
```

---

## 开放性问题

1. **JS 渲染的页面**：许多站点需要 JavaScript。选项：
   - 优雅地失败并显示有用的提示。
   - 如果可用，集成浏览器自动化工具。
   - 尽可能使用 API（例如 Twitter/X API 而非爬取）。

2. **身份认证**：如何处理已登录的会话？
   - 从浏览器导入 Cookie？
   - 手动设置头部信息？

3. **频率限制**：websh 是否应自动限制获取频率？

4. **缓存过期**：基于 TTL？还是仅手动刷新？

---

## 下一步行动

1. 创建带有激活触发器的 SKILL.md
2. 编写具有核心体现语义的 shell.md
3. 编写带有命令语法的 commands.md
4. 编写带有缓存逻辑的 state/cache.md
5. 为用户编写 help.md
6. 使用真实的 URL 进行测试

---

## 灵感来源

- Unix shell 哲学（小型工具、管道、文本流）
- OpenProse VM 模式（体现、状态文件）
- 最初的推文：“有没有针对 Web 的 shell？”