---
role: cache-management
summary: |
  websh 如何缓存页面和提取内容。包括驱动 haiku 子代理的迭代提取提示词、
  缓存目录结构以及提取不完整时的优雅降级。
see-also:
  - ../shell.md: Shell 语义 (Shell semantics)
  - ../commands.md: 命令参考 (Command reference)
  - crawl.md: 主动抓取代理 (Eager crawl agent)
---

# websh 缓存管理 (Cache Management)

当你 `cd` 到一个 URL 时，websh 会获取 HTML 并生成一个异步的 haiku 子代理，将丰富的内容提取到 markdown 文件中。本文档定义了缓存结构和提取过程。

---

## 目录结构 (Directory Structure)

```
.websh/
├── session.md                    # 当前会话状态
├── cache/
├── index.md                  # URL → 标识符 (slug) 映射
├── {slug}.html               # 原始 HTML
└── {slug}.parsed.md          # 提取后的内容（由 haiku 生成）
├── history.md                    # 命令历史记录
└── bookmarks.md                  # 已保存的 URL（书签）
```

---

## URL 到标识符 (Slug) 的转换

URL 会转换为可读的文件名：

**算法：**
1. 移除协议头 (`https://`)
2. 将 `/` 替换为 `-`
3. 将特殊字符替换为 `-`
4. 将多个连续的 `-` 合并为一个
5. 截断至合理的长度（最大 100 个字符）
6. 转换为小写

**示例：**

| URL | 标识符 (Slug) |
|-----|------|
| `https://news.ycombinator.com` | `news-ycombinator-com` |
| `https://x.com/deepfates/status/123` | `x-com-deepfates-status-123` |
| `https://techcrunch.com/2024/06/25/smashing/` | `techcrunch-com-2024-06-25-smashing` |
| `https://example.com/path?q=test&a=1` | `example-com-path-q-test-a-1` |

---

## index.md

跟踪所有缓存的 URL：

```markdown
# websh 缓存索引 (cache index)

## 条目 (Entries)

| Slug | URL | 获取时间 (Fetched) | 状态 (Status) |
|------|-----|---------|--------|
| news-ycombinator-com | https://news.ycombinator.com | 2026-01-24T10:30:00Z | extracted |
| x-com-deepfates-status-123 | https://x.com/deepfates/status/123 | 2026-01-24T10:35:00Z | extracting |
| techcrunch-com-article | https://techcrunch.com/... | 2026-01-24T10:40:00Z | fetched |
```

**状态值 (Status values)：**
- `fetched` — HTML 已保存，提取尚未开始
- `extracting` — Haiku 代理正在运行
- `extracted` — 提取完成

---

## 提取：Haiku 子代理 (The Haiku Subagent)

当 `cd` 完成获取 (fetch) 后，生成一个提取代理：

```
Task({
  description: "websh: extract page content",
  prompt: <EXTRACTION_PROMPT>,
  subagent_type: "general-purpose",
  model: "haiku",
  run_in_background: true
})
```

### 提取提示词 (Extraction Prompt)

````markdown
# websh 页面提取 (Page Extraction)

你正在为 websh 缓存从网页中提取有用内容。

## 输入

URL: {url}
HTML 文件: {html_path}
输出文件: {output_path}

## 任务

对 HTML 执行**迭代智能解析**。进行多次轮次 (Pass)，每次提取更多有用的细节。将你的发现写入输出 markdown 文件，并随之更新。

## 流程

```
循环直到提取彻底（通常为 2-4 个轮次）：
  1. 读取 HTML 文件
  2. 读取你当前的输出（如果存在）
  3. 识别缺失的内容或可以更丰富的地方
  4. 使用新发现更新输出文件
  5. 评估：是否还有更多有用的内容需要提取？
```

## 轮次重点 (Pass Focus)

- **第 1 轮 (Pass 1)**: 基础结构
  - 页面标题、主标题
  - 所有链接（文本 + href）
  - 基础元数据（description, og 标签）

- **第 2 轮 (Pass 2)**: 内容提取
  - 正文文章/内容文本
  - 评论或讨论（如果存在）
  - 关键引用或亮点
  - 作者、日期、来源信息

- **第 3 轮 (Pass 3)**: 结构和模式
  - 导航元素
  - 表单和输入框
  - 重复模式（列表项、卡片等）
  - 特定站点的结构（推文、帖子、故事）

- **第 4+ 轮 (Pass 4+)**: 精炼
  - 清理提取的文本
  - 添加上下文和关系
  - 记录任何异常或有趣的事情

## 输出格式

以以下格式写入 {output_path}：

```markdown
# {url}

获取时间: {timestamp}
轮次: {n}
状态: {extracting|complete}

## 摘要 (Summary)

{关于此页面的 2-3 句摘要}

## 链接 (Links)

| # | 文本 | Href | 备注 |
|---|------|------|-------|
| 0 | ... | ... | ... |
| 1 | ... | ... | ... |
...

## 内容 (Content)

### 主要内容 (Main Content)

{提取的文章文本，已清理且可读}

### 评论/讨论 (Comments/Discussion)

{如果适用}

### 侧边栏/导航 (Sidebar/Navigation)

{值得注意的导航或相关链接}

## 结构 (Structure)

页面类型: {文章, 列表, 个人资料, 搜索结果等}

关键模式:
- {selector} → {它包含的内容}
- ...

## 表单 (Forms)

### {表单名称/动作}
- {字段名称} ({类型})
- ...

## 媒体 (Media)

- {图像, 视频, 嵌入内容}

## 元数据 (Metadata)

- title: ...
- description: ...
- og:image: ...
- ...

## 提取备注 (Extraction Notes)

第 1 轮: {提取了什么}
第 2 轮: {添加了什么}
...
```

## 指南

1. **彻底且高效** — 提取所有有用的内容，跳过模版文字
2. **保留结构** — 保持页面的层级结构
3. **清理文本** — 移除 HTML 伪影、多余空格
4. **索引链接** — 对链接进行编号以便使用 `follow N` 导航
5. **记录模式** — 识别特定站点的结构
6. **保持可读性** — 输出应对人类和 `grep` 都有用

## 完成

在每轮之后，进行评估：
- 我是否捕捉到了主要内容？
- 链接是否已正确索引？
- 是否还有大量内容未提取？
- 增加一轮是否会增加显著价值？

当提取彻底时，将状态 (Status) 更新为 `complete` 并结束。

写下你的最终确认：
```
提取完成: {output_path}
轮次: {n}
链接数: {count}
内容: {简短描述}
```
````

---

## 优雅降级 (Graceful Degradation)

即使提取不完整，命令也能工作：

| 命令 | 如果已提取 | 如果只有 HTML |
|---------|--------------|--------------|
| `ls` | 来自 markdown 的丰富链接 | 基础 `<a>` 标签解析 |
| `cat .selector` | 来自提取的内容 | 直接 HTML 解析 |
| `grep "pattern"` | 搜索提取的文本 | 搜索原始文本 |
| `stat` | 完整元数据 | 基础信息 |

### 检查提取状态

在执行命令之前，检查：

1. `{slug}.parsed.md` 是否存在？
2. 它是否包含 `Status: complete`？

如果已完成，使用丰富的提取内容。否则，回退到 HTML 解析或显示可用内容。

---

## 缓存文件 (Cache Files)

### {slug}.html

与获取时完全一致的原始 HTML。保留用于：
- 提取不完整时的回退
- 需要完整 DOM 的 CSS 选择器查询
- 如果需要，重新进行提取

### {slug}.parsed.md

丰富的提取内容。示例：

```markdown
# https://news.ycombinator.com

获取时间: 2026-01-24T10:30:00Z
轮次: 3
状态: complete

## 摘要 (Summary)

Hacker News 首页。一个以技术为中心的链接聚合器，显示 30 条按分数排名的用户提交故事。包含 Show HN 项目、技术文章和行业新闻。

## 链接 (Links)

| # | 文本 | Href | 备注 |
|---|------|------|-------|
| 0 | Show HN: I built a tool for... | /item?id=41234567 | 142 pts, 87 comments |
| 1 | The State of AI in 2026 | /item?id=41234568 | 891 pts, 432 comments |
| 2 | Why Rust is eating the world | /item?id=41234569 | 234 pts, 156 comments |
| 3 | A deep dive into WebAssembly | /item?id=41234570 | 167 pts, 89 comments |
...

## 内容 (Content)

### 主要内容 (Main Content)

这是一个链接聚合器。故事以排名列表形式显示，包含：
- 链接到外部文章或内部讨论的标题
- 显示社区投票的分数
- 链接到讨论的评论计数

### 导航 (Navigation)

- [new](/newest) - 最新提交
- [past](/front) - 往期首页
- [comments](/newcomments) - 最近评论
- [ask](/ask) - Ask HN 问题
- [show](/show) - Show HN 项目
- [jobs](/jobs) - 招聘信息

## 结构 (Structure)

页面类型: 链接聚合器 / 新闻提要 (Link aggregator / News feed)

关键模式:
- .titleline → 故事标题
- .score → 分数
- .hnuser → 用户名
- .age → 提交时间
- .subtext → 元数据行（分数、用户、时间、评论）

## 表单 (Forms)

### 搜索 (外部)
- q (文本) → Algolia HN 搜索

### 登录 (/login)
- acct (文本)
- pw (密码)

## 媒体 (Media)

无（纯文本设计）

## 元数据 (Metadata)

- title: Hacker News
- (无 meta description)
- (无 og 标签)

## 提取备注 (Extraction Notes)

第 1 轮: 找到 30 个故事链接，基础结构
第 2 轮: 提取了导航，识别了模式
第 3 轮: 添加了元数据，清理了内容描述
```

---

## 会话状态 (Session State)

### session.md

```markdown
# websh 会话 (session)

开始时间: 2026-01-24T10:30:00Z
当前路径 (pwd): https://news.ycombinator.com
路径标识符 (pwd_slug): news-ycombinator-com

## 导航栈 (Navigation Stack)

- https://news.ycombinator.com

## 最近命令 (Recent Commands)

1. cd https://news.ycombinator.com
2. ls | head 5
3. grep "AI"
```

在每个命令后更新。

---

## 缓存过期 (Cache Expiration)

目前，缓存不会自动过期。使用 `refresh` 重新获取。

未来考虑：基于 TTL 的过期、陈旧警告。

---

## 初始化 (Initialization)

在执行第一个 websh 命令时，如果 `.websh/` 不存在：

```bash
mkdir -p .websh/cache
touch .websh/session.md
touch .websh/history.md
touch .websh/bookmarks.md
echo "# websh 缓存索引\n\n## 条目\n\n| Slug | URL | 获取时间 | 状态 |" > .websh/cache/index.md
```

写入初始会话状态：

```markdown
# websh 会话 (session)

开始时间: {now}
当前路径 (pwd): (无)

## 导航栈 (Navigation Stack)

(空)

## 最近命令 (Recent Commands)

(无)
```

