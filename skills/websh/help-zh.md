---
role: user-documentation
summary: |
  websh 面向用户的帮助文档。包含快速入门、完整命令速查表和示例。
---

# websh 帮助

一个用于 Web 的类 Unix shell。像操作目录一样导航 URL，使用熟悉的命令查询页面。

## 快速入门

```
websh                                # 启动 shell
ls                                   # 显示建议的网站
go hn                                # 前往 Hacker News (预设书签)
ls | head 5                          # 显示前 5 个链接
grep "AI"                            # 搜索文本
follow 1                             # 点击第 2 个链接
cat .title                           # 通过选择器提取文本
back                                 # 返回
```

## 初始书签 (Starter Bookmarks)

websh 预置了一些有趣的公共网站书签：

| 快捷键 | 网站 |
|----------|------|
| `go hn` | Hacker News |
| `go lobsters` | Lobsters |
| `go tildes` | Tildes |
| `go wiby` | Wiby (独立 Web 搜索) |
| `go marginalia` | Marginalia (独立搜索引擎) |
| `go wiki` | 维基百科 |
| `go sourcehut` | Sourcehut |
| `go arena` | Are.na |

可以使用 `bookmark <name>` 添加你自己的书签。

---

## 命令速查表

### 导航 (Navigation)

| 命令 | 描述 |
|---------|-------------|
| `cd <url>` | 前往指定 URL |
| `cd -` | 前往上一个 URL |
| `cd ~` | 前往起点（清除导航记录） |
| `pwd` | 显示当前 URL |
| `back` / `forward` | 在历史记录中后退/前进 |
| `follow <n>` | 跟随第 n 个链接 |
| `follow "text"` | 跟随包含指定文本的链接 |
| `refresh` | 重新获取当前页面 |
| `chroot <url>` | 将导航限制在指定的 URL 前缀内 |

### 查询与提取 (Query & Extract)

| 命令 | 描述 |
|---------|-------------|
| `ls` | 列出所有链接 |
| `ls -l` | 列出链接及其对应的 URL |
| `ls <selector>` | 列出匹配选择器的元素 |
| `cat <selector>` | 提取文本内容 |
| `grep <pattern>` | 按模式搜索/过滤 |
| `grep -i` | 忽略大小写 |
| `grep -v` | 反向匹配 |
| `stat` | 显示页面元数据 |
| `source` | 查看原始 HTML |
| `dom` | 显示 DOM 树 |

### 预取 (Prefetching)

| 命令 | 描述 |
|---------|-------------|
| `prefetch` | 显示爬取状态 |
| `prefetch on/off` | 开启/关闭主动爬取 |
| `prefetch <url>` | 手动预取某个 URL |
| `prefetch --depth <n>` | 设置预取深度 |
| `crawl <url>` | 显式的深度爬取 |
| `queue` | 显示爬取队列 |

### 搜索与发现 (Search & Discovery)

| 命令 | 描述 |
|---------|-------------|
| `find <pattern>` | 递归搜索/爬取 |
| `find -depth <n>` | 爬取 n 层深度 |
| `locate <term>` | 搜索所有缓存页面 |
| `tree` | 显示站点结构 |
| `which <link>` | 解析重定向 |

### 文本处理 (Text Processing)

| 命令 | 描述 |
|---------|-------------|
| `head <n>` | 获取前 n 项 |
| `tail <n>` | 获取后 n 项 |
| `sort` | 对输出进行排序 |
| `sort -r` | 反向排序 |
| `uniq` | 去除重复项 |
| `wc` | 统计行数/单词数 |
| `wc --links` | 统计链接数 |
| `cut -f <n>` | 提取字段 |
| `tr` | 转换字符 |
| `sed 's/a/b/'` | 流编辑 |

### 比较 (Comparison)

| 命令 | 描述 |
|---------|-------------|
| `diff <url1> <url2>` | 比较两个页面 |
| `diff -t 1h` | 与 1 小时前进行比较 |
| `diff --wayback <date>` | 与 Wayback 快照进行比较 |

### 监控 (Monitoring)

| 命令 | 描述 |
|---------|-------------|
| `watch <url>` | 监控变化 |
| `watch -n 30` | 每 30 秒轮询一次 |
| `watch --notify` | 发生变化时发送系统通知 |
| `ping <url>` | 检查网站是否在线 |
| `traceroute <url>` | 显示重定向链 |
| `time <cmd>` | 测量执行时间 |

### 作业与后台 (Jobs & Background)

| 命令 | 描述 |
|---------|-------------|
| `<cmd> &` | 在后台运行 |
| `ps` | 显示正在运行的任务 |
| `jobs` | 列出后台作业 |
| `fg %<n>` | 将作业切换到前台 |
| `bg %<n>` | 在后台继续作业 |
| `kill %<n>` | 取消作业 |
| `wait` | 等待所有作业完成 |

### 环境与认证 (Environment & Auth)

| 命令 | 描述 |
|---------|-------------|
| `env` | 显示环境信息 |
| `export VAR=val` | 设置变量 |
| `export HEADER_X=val` | 设置请求头部 |
| `export COOKIE_x=val` | 设置 Cookie |
| `unset VAR` | 移除变量 |
| `whoami` | 显示已登录身份 |
| `login` | 交互式登录 |
| `logout` | 清除会话 |
| `su <profile>` | 切换配置方案 |

### 挂载 (Mounting)

| 命令 | 描述 |
|---------|-------------|
| `mount <api> <path>` | 将 API 挂载为目录 |
| `mount -t github ...` | 挂载 GitHub API |
| `mount -t rss ...` | 挂载 RSS 订阅源 |
| `umount <path>` | 卸载 |
| `df` | 显示挂载点和缓存使用情况 |
| `quota` | 显示频率限制 |

### 归档与快照 (Archives & Snapshots)

| 命令 | 描述 |
|---------|-------------|
| `tar -c <file> <urls>` | 归档页面 |
| `tar -x <file>` | 提取归档内容 |
| `snapshot` | 保存带时间戳的版本 |
| `snapshot -l` | 列出快照 |
| `wayback <url>` | 列出 Wayback 快照 |
| `wayback <url> <date>` | 从 Wayback 获取内容 |

### 站点元数据 (Site Metadata)

| 命令 | 描述 |
|---------|-------------|
| `robots` | 显示 robots.txt |
| `sitemap` | 显示 sitemap.xml |
| `headers` | 显示 HTTP 头部 |
| `cookies` | 管理 Cookie |

### 交互 (Interaction)

| 命令 | 描述 |
|---------|-------------|
| `click <selector>` | 点击元素 |
| `submit <form>` | 提交表单 |
| `type <sel> "text"` | 填写输入框 |
| `scroll` | 触发无限滚动 |
| `screenshot <file>` | 捕获页面截图 |

### 调度 (Scheduling)

| 命令 | 描述 |
|---------|-------------|
| `cron "<sched>" <cmd>` | 安排周期性任务 |
| `at "<time>" <cmd>` | 安排一次性任务 |
| `cron -l` | 列出已安排的任务 |

### 别名与快捷键 (Aliases & Shortcuts)

| 命令 | 描述 |
|---------|-------------|
| `alias name='cmd'` | 创建别名 |
| `alias` | 列出所有别名 |
| `unalias name` | 移除别名 |
| `ln -s <url> <name>` | 创建 URL 快捷方式 |

### 状态与历史记录 (State & History)

| 命令 | 描述 |
|---------|-------------|
| `history` | 显示命令历史记录 |
| `!!` | 重复最后一条命令 |
| `!<n>` | 重复第 n 条命令 |
| `bookmark <name>` | 保存当前 URL |
| `bookmarks` | 列出所有书签 |
| `go <name>` | 前往书签 |

### 文件操作 (File Operations)

| 命令 | 描述 |
|---------|-------------|
| `save <path>` | 将页面保存到文件 |
| `save --parsed` | 保存提取后的 markdown 内容 |
| `tee <file>` | 在显示的同时保存内容 |
| `xargs <cmd>` | 从输入构建命令 |
| `parallel` | 并行运行 |

---

## 管道与重定向

```
ls | grep "AI" | head 5              # 管道连接命令
ls > links.txt                       # 写入文件
ls >> links.txt                      # 追加到文件
ls | tee links.txt                   # 保存并显示
cd $(wayback https://x.com 2020)     # 命令替换
```

---

## 选择器 (Selectors)

CSS 选择器可配合 `ls`, `cat`, `click` 使用：

```
cat .article          # 类名
cat #main             # ID
cat article           # 标签
cat .post .title      # 后代
cat h1:first          # 第一个匹配项
ls nav a              # nav 内部的链接
click button.submit   # 带有指定类名的按钮
```

---

## 示例

### 浏览 Hacker News

```
cd https://news.ycombinator.com
ls | head 10                         # 前 10 个故事
grep "Show HN"                       # 过滤
follow "Show HN"                     # 前往第一个匹配项
cat .comment | head 20               # 阅读评论
back
```

### 研究某个主题

```
cd https://en.wikipedia.org/wiki/Unix
cat #mw-content-text | head 50       # 简介
ls #toc                              # 目录
follow "History"
bookmark unix-history
```

### 监控页面

```
watch https://status.example.com -n 30 --notify
# 每 30 秒轮询一次，发生变化时发送通知
```

### 挂载 GitHub API

```
mount https://api.github.com /gh
cd /gh/users/torvalds
cat bio
cd /gh/repos/torvalds/linux
ls issues | head 10
```

### 比较随时间变化的页面

```
cd https://example.com
snapshot "before"
# ... 等待 ...
refresh
diff --snapshot "before"
```

### 批量获取

```
parallel cd ::: https://a.com https://b.com https://c.com
locate "error" | head 10
```

### 在缓存页面中搜索

```
locate "authentication"
# 立即搜索所有缓存页面

locate -i "OAuth" --urls
# 忽略大小写，并显示 URL
```

### 预取实现即时导航

```
cd https://news.ycombinator.com
# 自动在后台预取可见链接

prefetch
# 检查预取进度

follow 3
# 即时完成！内容已缓存。

prefetch off
# 在慢速网络下禁用
```

### 归档研究资料

```
cd https://paper1.com &
cd https://paper2.com &
cd https://paper3.com &
wait
tar -cz research.tar.gz https://paper1.com https://paper2.com https://paper3.com
```

### 设置认证头部

```
export HEADER_Authorization="Bearer mytoken"
cd https://api.example.com/protected
cat .
```

### 安排监控任务

```
cron "0 * * * *" 'cd https://news.com && ls | head 5 >> hourly.txt'
cron "0 9 * * *" 'snapshot "daily"'
```

---

## 工作原理

当你 `cd` 到一个 URL 时：

1. **获取 (Fetch)** — 下载 HTML
2. **缓存 (Cache)** — 保存到 `.websh/cache/`
3. **提取 (Extract)** — 后台 haiku 代理将其解析为丰富的 markdown 内容

`ls`, `grep`, `cat` 等命令作用于缓存内容——即时响应，无需重新获取。

挂载的 API 工作原理类似——API 响应被缓存并可供导航。

---

## 文件结构

```
.websh/
├── session.md      # 当前会话
├── cache/          # 缓存页面 (HTML + 解析后的 markdown)
├── history.md      # 命令历史记录
├── bookmarks.md    # 已保存的 URL
├── profiles/       # 认证配置方案
└── snapshots/      # 已保存的版本
```

---

## 自然语言

websh 理解意图，而不仅仅是命令。以下这些都可以正常工作：

```
links                    → ls
open https://example.com → cd https://example.com
search "AI"              → grep "AI"
what's on this page?     → ls + stat
show me the title        → cat title
go back                  → back
how many links?          → wc --links
download this            → save
```

直接说出你的需求，websh 会自动识别。

---

## 提示

- **即时导航**：链接会自动预取——`follow` 通常是即时的
- **利用索引**：`ls` 会为链接编号，`follow 3` 点击第 4 个链接
- **万物皆可管道**：`ls | grep "foo" | head 5 | tee results.txt`
- **后台运行耗时任务**：`cd https://slow-site.com &`
- **搜索缓存**：`locate` 立即搜索所有缓存页面
- **挂载 API**：`mount` 让 REST API 像目录一样可导航
- **随时间比较**：`snapshot` + `diff --snapshot`
- **安排检查任务**：`cron` 用于循环任务，`at` 用于一次性任务
- **控制预取**：在慢速连接下使用 `prefetch off`，使用 `prefetch` 检查进度

---

## 局限性

- **JavaScript 站点**：部分内容需要 JS 渲染
- **认证**：`login` 是尽力而为的，可能需要手动设置 Cookie
- **频率限制 (Rate limits)**：尊重站点限制，使用 `quota` 检查
- **交互**：在没有完整浏览器的情况下，`click`, `submit` 功能有限

---

## 获取帮助

```
help                 # 此帮助文档
help <command>       # 特定命令的帮助
man <command>        # 详细手册
```