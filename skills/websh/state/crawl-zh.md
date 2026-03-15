---
role: crawl-management
summary: |
  websh 的主动链接抓取。获取页面后，自动在后台预获取
  1-2 层深的链接页面。使导航感觉即时。
see-also:
  - cache.md: 缓存结构
  - ../shell.md: Shell 语义
  - ../commands.md: 命令参考
---

# websh 主动抓取

当你 `cd` 到一个 URL 时，websh 可以自动在后台预获取链接页面。这使得 `follow` 和导航感觉即时——当你需要时内容已经缓存好了。

---

## 工作原理

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   cd https://news.ycombinator.com                         │
│         │                                                  │
│         ▼                                                  │
│   ┌───────────────┐                                       │
│   │ Fetch + Extract│  ← Background haiku (existing)       │
│   │ the main page  │                                      │
│   └───────┬───────┘                                       │
│           │ After Pass 1 (links identified)               │
│           ▼                                                │
│   ┌───────────────┐                                       │
│   │ Spawn Eager   │  ← New background haiku               │
│   │ Crawl Agent   │                                       │
│   └───────┬───────┘                                       │
│           │                                                │
│           ▼                                                │
│   For each link (prioritized, rate-limited):             │
│   ┌───────────────┐                                       │
│   │ Fetch + Extract│  ← Parallel background tasks          │
│   │ linked page    │                                       │
│   └───────┬───────┘                                       │
│           │ If depth < max_depth                          │
│           ▼                                                │
│   Queue its links for next layer...                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

用户立即获得提示符返回。所有抓取都是异步进行的。

---

## 抓取设置

存储在 `.websh/session.md` 的 Environment 部分：

```markdown
## Environment

EAGER_CRAWL: true
CRAWL_DEPTH: 2
CRAWL_SAME_DOMAIN: true
CRAWL_MAX_PER_PAGE: 20
CRAWL_MAX_CONCURRENT: 5
CRAWL_DELAY_MS: 200
```

### 设置说明

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `EAGER_CRAWL` | `true` | 启用/禁用主动抓取 |
| `CRAWL_DEPTH` | `2` | 预获取的层深度 |
| `CRAWL_SAME_DOMAIN` | `true` | 仅抓取同域链接 |
| `CRAWL_MAX_PER_PAGE` | `20` | 每页最大链接抓取数 |
| `CRAWL_MAX_CONCURRENT` | `5` | 最大同时抓取数 |
| `CRAWL_DELAY_MS` | `200` | 请求间延迟（速率限制） |

### 更改设置

```
export EAGER_CRAWL=false           # 禁用主动抓取
export CRAWL_DEPTH=3               # 设置3层深度
export CRAWL_SAME_DOMAIN=false     # 包含外部链接
prefetch off                       # 快捷禁用命令
prefetch on --depth 3              # 启用并设置3层深度
```

---

## 抓取队列

在 `.websh/crawl-queue.md` 中跟踪：

```markdown
# websh crawl queue

## Active Crawl

origin: https://news.ycombinator.com
started: 2026-01-24T10:30:00Z
depth: 2
same_domain: true

## 进行中

| 标识符 | 网址 | 深度 | 状态 |
|------|-----|-------|--------|
| news-ycombinator-com-item-id-41234567 | https://news.ycombinator.com/item?id=41234567 | 1 | extracting |
| news-ycombinator-com-item-id-41234568 | https://news.ycombinator.com/item?id=41234568 | 1 | fetching |

## 队列中

| 网址 | 深度 | 优先级 |
|-----|-------|----------|
| https://news.ycombinator.com/item?id=41234569 | 1 | 2 |
| https://news.ycombinator.com/item?id=41234570 | 1 | 3 |
...

## 已完成

| 标识符 | 网址 | 深度 | 找到的链接数 |
|------|-----|-------|-------------|
| news-ycombinator-com | https://news.ycombinator.com | 0 | 30 |

## 已跳过

| 网址 | 原因 |
|-----|--------|
| https://external.com/article | external (same_domain=true) |
| https://news.ycombinator.com/login | already cached |
```

---

## 优先级算法

链接按以下规则优先抓取：

1. **页面位置** — 出现较早的链接获得更高优先级
2. **同域优先** — 内部链接优先于外部链接
3. **内容信号** — 主要内容中的链接 > 导航/页脚中的链接
4. **避免重复** — 跳过已缓存的 URL
5. **跳过非内容** — 忽略登录、登出、设置等链接

### 链接评分

```python
def score_link(link, index, is_same_domain, in_main_content):
    score = 1000 - index  # Position: earlier = higher

    if is_same_domain:
        score += 500

    if in_main_content:
        score += 300

    # Penalize common non-content patterns
    skip_patterns = ['login', 'logout', 'signup', 'settings', 'account', '#']
    if any(p in link.href.lower() for p in skip_patterns):
        score -= 1000

    return score
```

---

## The Crawl Agent Prompt

After initial page extraction completes Pass 1, spawn this agent:

````markdown
# websh Eager Crawl Agent

You are prefetching linked pages for websh to make navigation instant.

## Origin Page

URL: {url}
Slug: {slug}
Parsed file: {parsed_path}

## Settings

depth: {depth}
same_domain: {same_domain}
max_per_page: {max_per_page}
max_concurrent: {max_concurrent}

## Task

1. Read the parsed markdown file to get the link list
2. Filter and prioritize links:
   - Skip already-cached URLs (check .websh/cache/index.md)
   - Skip external if same_domain=true
   - Skip login/logout/settings/account URLs
   - Take top {max_per_page} by priority (earlier position = higher)
3. For each link, spawn a fetch+extract task (like cd does)
4. Track progress in .websh/crawl-queue.md
5. 如果深度 > 1，将发现的链接放入下一层的队列中

## 速率限制 (Rate Limiting)

- 最大 {max_concurrent} 个并发获取任务
- 启动新任务之间有 {delay_ms} 毫秒的延迟
- 尊重源服务器的负载

## 生成模式 (Spawn Pattern)

对于每个要预取的 URL：

```python
Task(
    description=f"websh: prefetch {slug}",
    prompt=FETCH_AND_EXTRACT_PROMPT,  # 与 cd 使用的相同
    subagent_type="general-purpose",
    model="haiku",
    run_in_background=True
)
```

## 完成 (Completion)

当所有深度的所有链接都处理完毕时：
1. 使用最终统计数据更新 crawl-queue.md
2. 记录完成日志："Prefetch complete: {n} pages cached from {origin}"

## 优雅处理 (Graceful Handling)

- 如果获取失败，记录日志并继续处理其他任务
- 如果受到速率限制，退避并重试
- 永远不要阻塞在缓慢的网站上——直接移动到下一个链接
- 用户可以通过 `kill %crawl` 或 `prefetch stop` 取消任务
````

---

## 深度-2 抓取 (Depth-2 Crawling)

当深度 > 1 时，抓取将递归继续：

```
第 0 层 (Layer 0): cd https://news.ycombinator.com
                  → 提取出 30 个链接

第 1 层 (Layer 1): 预取前 20 个链接
                  → 每个页面再提取约 10-30 个链接

第 2 层 (Layer 2): 从每个第 1 层页面预取前 20 个链接
                  → 但跳过所有层级中的重复项
```

### 去重 (Deduplication)

抓取队列会跟踪所有见过的 URL：

```markdown
## 已见 URL (Seen URLs)

(已经缓存、正在处理或已入队的 URL——不再重复抓取)

- https://news.ycombinator.com
- https://news.ycombinator.com/item?id=41234567
- https://news.ycombinator.com/item?id=41234568
...
```

这可以防止在不同深度重复抓取相同的 URL。

---

## 命令 (Commands)

| 命令 | 描述 |
|---------|-------------|
| `prefetch` | 显示当前抓取状态 |
| `prefetch on` | 启用主动抓取 (eager crawl) |
| `prefetch off` | 禁用主动抓取 |
| `prefetch <url>` | 手动预取特定的 URL |
| `prefetch --depth N` | 设置抓取深度 |
| `prefetch --stop` | 停止当前抓取 |
| `crawl <url>` | 对 URL 进行显式的完整抓取 |
| `crawl --depth N` | 为显式抓取设置深度 |
| `queue` | 显示抓取队列 |

### prefetch 状态输出示例

```
Eager crawl: enabled (主动抓取：已启用)
Depth: 2, Same domain: yes, Max per page: 20

Current crawl (当前抓取):
  Origin (源): https://news.ycombinator.com
  Progress (进度): Layer 1 - 15/20 complete
  Queued (队列中): 42 URLs for Layer 2

Recent (最近):
  [✓] news-ycombinator-com-item-id-41234567 (12 links)
  [✓] news-ycombinator-com-item-id-41234568 (8 links)
  [→] news-ycombinator-com-item-id-41234569 (fetching...)
```

---

## 与 cd 集成 (Integration with cd)

`cd` 命令在提取开始后会触发主动抓取：

```python
def cd(url):
    # ... 现有的 cd 逻辑 (获取 + 提取) ...

    # 在生成提取任务后，如果已启用则同时生成抓取任务
    if env.EAGER_CRAWL:
        # 稍等片刻待第一阶段 (Pass 1) 完成，然后开始抓取
        Task(
            description=f"websh: eager crawl {slug}",
            prompt=EAGER_CRAWL_PROMPT.format(
                url=full_url,
                slug=slug,
                parsed_path=f".websh/cache/{slug}.parsed.md",
                depth=env.CRAWL_DEPTH,
                same_domain=env.CRAWL_SAME_DOMAIN,
                max_per_page=env.CRAWL_MAX_PER_PAGE,
                max_concurrent=env.CRAWL_MAX_CONCURRENT,
                delay_ms=env.CRAWL_DELAY_MS,
            ),
            subagent_type="general-purpose",
            model="haiku",
            run_in_background=True
        )
```

抓取代理会等待链接可用，然后开始预取。

---

## 性能考量 (Performance Considerations)

### 为什么要进行主动抓取？ (Why Eager Crawl?)

| 不使用主动抓取 | 使用主动抓取 |
|---------------------|------------------|
| `follow 3` → 等待获取 | `follow 3` → 瞬时完成 (已缓存) |
| `back` → 可能需要重新获取 | `back` → 瞬时完成 (已缓存) |
| 探索过程感觉缓慢 | 探索过程感觉是即时的 |

### 成本/收益 (Cost/Benefit)

| 优点 | 缺点 |
|------|------|
| 导航感觉是即时的 | 使用更多带宽 |
| 内容在需要时已就绪 | 缓存占用更多磁盘空间 |
| 自然的浏览流 | 可能会获取从未访问过的页面 |
| 对已缓存页面可离线工作 | 后台 CPU 使用 |

### 何时禁用 (When to Disable)

```
prefetch off
```

在以下情况下禁用主动抓取：
- 使用按流量计费的连接
- 抓取大型/慢速网站
- 磁盘空间受限
- 仅打算访问一个页面

---

## 会话示例 (Example Session)

```
~> cd https://news.ycombinator.com

news.ycombinator.com> (fetching...)

news.ycombinator.com> prefetch
Eager crawl: enabled (主动抓取：已启用)
Current crawl (当前抓取):
  Origin (源): https://news.ycombinator.com
  Progress (进度): Waiting for extraction... (正在等待提取...)

news.ycombinator.com> ls | head 5
[0] Show HN: I built a tool for...
[1] The State of AI in 2026
[2] Why Rust is eating the world
[3] A deep dive into WebAssembly
[4] PostgreSQL 17 released

news.ycombinator.com> prefetch
Current crawl:
  Origin: https://news.ycombinator.com
  Progress: Layer 1 - 8/20 complete
  [✓] .../item?id=41234567
  [✓] .../item?id=41234568
  [→] .../item?id=41234569 (fetching...)
  ...

news.ycombinator.com> follow 1

news.ycombinator.com/item> (cached)    # 瞬时完成！已经预取过了。

news.ycombinator.com/item> cat .title
The State of AI in 2026

news.ycombinator.com/item> back

news.ycombinator.com> (cached)         # 同样瞬时完成
```

---

## 尊重 robots.txt (robots.txt Respect)

在抓取之前，检查 robots.txt：

```python
def should_crawl(url, domain):
    robots = get_robots(domain)  # 已缓存
    return robots.can_fetch("websh/1.0", url)
```

如果不允许，跳过该 URL 并记录：

```markdown
## 已跳过 (Skipped)

| URL | 原因 (Reason) |
|-----|--------|
| https://example.com/private | robots.txt 不允许 |
```

---

## 取消 (Cancellation)

用户可以随时停止抓取：

```
prefetch stop
# 或
kill %crawl
```

这将取消待处理的获取任务，但保留已经缓存的内容。
