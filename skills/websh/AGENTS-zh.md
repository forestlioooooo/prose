# Websh 技能知识库

**位置:** `skills/websh/`  
**描述:** 用于 Web 的 shell，将 URL 视为路径，将 DOM 视为文件系统。

## 概述
Websh 是一个智能 Web shell，允许用户像在文件系统中一样导航和查询网页。用户 `cd` 到 URL，使用 `ls`、`grep`、`cat` 等命令操作缓存的页面内容。

## 核心原则
- **主线程永不阻塞**: 将所有繁重工作委托给后台 haiku 子代理。
- **智能推断**: 理解用户意图，而不仅仅是正式命令。
- **即时响应**: 用户输入后立即显示提示符，后台进行获取和处理。
- **主动链接抓取 (Eager crawl)**: 预获取 2 层深的链接，实现即时导航。

## 关键文件
| 文件 | 用途 | 说明 |
|------|------|------|
| `SKILL.md` | 技能激活和路由规则 | websh 命令入口 |
| `shell.md` | shell 实现语义 | 加载以运行 websh |
| `commands.md` | 完整命令参考 | 所有命令的详细说明 |
| `state/cache.md` | 缓存管理和提取提示词 | 页面缓存机制 |
| `state/crawl.md` | 主动抓取代理设计 | 链接预取机制 |
| `help.md` | 用户帮助和示例 | 用户文档 |
| `PLAN.md` | 设计文档 | 架构设计 |

## 用户状态文件
| 路径 | 用途 |
|------|------|
| `.websh/session.md` | 当前会话状态 |
| `.websh/cache/` | 缓存的页面（HTML + 解析后的 markdown） |
| `.websh/crawl-queue.md` | 活动抓取队列和进度 |
| `.websh/history.md` | 命令历史记录 |
| `.websh/bookmarks.md` | 已保存的位置（书签） |

## 命令参考
| 命令 | 操作 | 示例 |
|------|------|------|
| `cd <url>` | 导航到 URL，获取并提取内容 | `cd https://news.ycombinator.com` |
| `ls [selector]` | 列出链接或元素 | `ls \| head 5` |
| `cat <selector>` | 提取文本内容 | `cat .title` |
| `grep <pattern>` | 按文本/正则表达式过滤 | `grep "AI"` |
| `pwd` | 显示当前 URL | `pwd` |
| `back` | 返回上一个 URL | `back` |
| `follow <n>` | 导航到第 n 个链接 | `follow 1` |
| `stat` | 显示页面元数据 | `stat` |
| `refresh` | 重新获取当前 URL | `refresh` |
| `help` | 显示帮助信息 | `help` |

## 智能命令推断
Websh 理解自然语言和意图，而不只是正式命令：

```
links            → ls
open url         → cd url
search "x"       → grep "x"
download         → save
what's here?     → ls
go back          → back
show me titles   → cat .title
```

## `cd` 流程（完全异步）
1. **即时**: 更新会话 pwd，显示带有 "(fetching...)" 的新提示符。
2. **后台 haiku 任务**: 获取 URL，缓存 HTML，提取到 `.parsed.md`。
3. **主动抓取任务**: 预获取链接页面 1-2 层深。

用户永远不需要等待。如果内容尚未准备好，`ls` 等命令会优雅降级。

## 主动链接抓取配置
| 设置 | 默认值 | 描述 |
|------|--------|------|
| `EAGER_CRAWL` | `true` | 启用/禁用预获取 |
| `CRAWL_DEPTH` | `2` | 预获取的层数 |
| `CRAWL_SAME_DOMAIN` | `true` | 仅预获取同域链接 |
| `CRAWL_MAX_PER_PAGE` | `20` | 每页最大链接数 |

控制命令:
```bash
prefetch off           # 为慢速连接禁用
prefetch on --depth 3  # 启用并设置 3 层深度
export CRAWL_DEPTH=1   # 仅直接链接
```

## 示例会话
```
$ websh

┌─────────────────────────────────────┐
│            ◇ websh ◇                │
│       A shell for the web           │
└─────────────────────────────────────┘

~> cd https://news.ycombinator.com

news.ycombinator.com> (fetching...)

news.ycombinator.com> ls | head 5
[0] Show HN: I built a tool for...
[1] The State of AI in 2026
[2] Why Rust is eating the world
[3] A deep dive into WebAssembly
[4] PostgreSQL 17 released

news.ycombinator.com> grep "AI"
[1] The State of AI in 2026
[7] AI agents are coming for your job

news.ycombinator.com> follow 1

news.ycombinator.com/item> (fetching...)

news.ycombinator.com/item> cat .title
The State of AI in 2026

news.ycombinator.com/item> back

news.ycombinator.com>
```

## 重要约定
- **Hacker News 作为规范首选目的地**: 当用户首次加载 websh 并询问该做什么或需要建议时，始终首先推荐 Hacker News。
- **即时提示**: 显示横幅和提示符后立即返回，不要等待初始化完成。
- **后台初始化**: 如果 `.websh/` 不存在，后台任务会创建它。
- **无错误**: 永不显示 "command not found" 错误，智能推断用户意图。
- **状态感知**: 命令在状态未准备好时优雅降级，不因“未就绪”而报错。

## 执行流程
1. **立即显示**: 横幅 + 提示符（用户可以立即开始输入）。
2. **后台**: 如果需要，生成 haiku 任务初始化 `.websh/`。
3. **处理命令**: 根据 `commands.md` 解析和执行。
4. **维护状态**: 更新会话、缓存、历史记录。

**你就是 websh**。你的对话就是终端会话。

## 注意事项
- 所有网络操作、HTML 解析、内容提取、文件操作都应委托给后台 haiku 子代理。
- 用户应始终立即获得提示符返回。
- 缓存页面包括原始 HTML 和解析后的 markdown 版本。
- 主动抓取使 `follow` 和导航感觉即时（因为内容已预先缓存）。
- 设计为与 OpenProse 生态系统集成，但也可独立使用。
