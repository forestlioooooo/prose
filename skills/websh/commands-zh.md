---
role: command-reference
summary: |
  websh 所有命令的完整参考。涵盖导航、查询、进程管理、
  监控、环境、挂载等内容——将 Web 视为 Unix 文件系统。
see-also:
  - shell.md: shell 语义和执行模型
  - state/cache.md: 缓存结构说明
---

# websh 命令参考

## 导航命令 (Navigation Commands)

### `cd <url>`

导航到指定的 URL。获取页面内容，进行缓存，并触发异步提取。

**语法：**
```
cd <url>
cd <relative-path>
cd -                 # 前往上一个位置
cd ~                 # 前往主页/起点（清除导航记录）
```

**示例：**
```
cd https://news.ycombinator.com
cd https://x.com/deepfates
cd /item?id=12345          # 相对于当前域名的路径
cd ..                       # 向上一级路径
cd -                        # 返回上一个 URL
```

**输出：** 导航确认信息，提取状态

---

### `pwd`

打印当前 URL。

**语法：**
```
pwd
pwd -P               # 显示完整解析后的 URL（无别名）
```

**输出：** 当前完整 URL 或 `(no page loaded)`

---

### `back`

返回导航历史中的上一个 URL。

**语法：**
```
back
back <n>             # 向后返回 n 步
```

**行为：** 使用缓存内容，不重新获取。

---

### `forward`

在导航历史中前进（在使用 `back` 之后）。

**语法：**
```
forward
forward <n>
```

---

### `follow <target>`

导航到当前页面上的链接。

**语法：**
```
follow <index>       # 通过 ls 输出中的编号指定
follow "<text>"      # 通过链接文本（部分匹配）指定
follow -n            # 跟随链接但不添加到历史记录中
```

**示例：**
```
follow 3                    # 跟随第 4 个链接（索引从 0 开始）
follow "State of AI"        # 跟随包含此文本的链接
```

---

### `refresh`

重新获取当前 URL，更新缓存。

**语法：**
```
refresh
refresh --hard       # 清除提取内容，重新开始
```

---

### `chroot <url>`

将导航限制在某个子域名或路径前缀内。

**语法：**
```
chroot <url>         # 设置根边界
chroot               # 显示当前 chroot 状态
chroot /             # 清除 chroot
```

**示例：**
```
chroot https://docs.python.org/3/
cd tutorial          # 正常：在 chroot 范围内
cd https://google.com # 报错：超出 chroot 范围
```

---

## 查询命令 (Query Commands)

### `ls [selector]`

列出当前页面上的链接或元素。

**语法：**
```
ls                   # 列出所有链接
ls <selector>        # 列出匹配 CSS 选择器的元素
ls -l                # 详细格式（带 href）
ls -a                # 包含隐藏/导航链接
ls -t                # 按在页面中的位置排序
ls -S                # 按文本长度排序
```

**输出：**
```
[0] 第一个链接文本
[1] 第二个链接文本
```

使用 `-l` 时：
```
[0] 第一个链接文本 → /path/to/page
[1] 第二个链接文本 → https://external.com/
```

**是否可管道传输：** 是

---

### `cat <selector>`

从元素中提取文本内容。

**语法：**
```
cat <selector>
cat .                # 整个页面的文本
cat -n               # 带行号
cat -A               # 显示所有（包括隐藏元素）
```

**示例：**
```
cat .title
cat article
cat .comment | head 3
cat -n .code-block
```

**是否可管道传输：** 是

---

### `grep <pattern>`

通过文本模式过滤内容（支持正则表达式）。

**语法：**
```
grep <pattern>
grep -i <pattern>    # 忽略大小写
grep -v <pattern>    # 反向匹配（不包含模式的行）
grep -c <pattern>    # 统计匹配项数量
grep -n <pattern>    # 显示行号
grep -o <pattern>    # 仅显示匹配的部分
grep -A <n>          # 显示匹配项后的 n 行
grep -B <n>          # 显示匹配项前的 n 行
grep -C <n>          # 显示匹配项前后各 n 行（上下文）
grep -E <pattern>    # 扩展正则表达式
grep -l              # 列出包含匹配项的页面（用于 locate/find）
```

**是否可管道传输：** 是（过滤输入流或搜索页面）

---

### `stat`

显示当前页面的元数据。

**语法：**
```
stat
stat -v              # 详细模式（显示所有元数据）
```

**输出：**
```
URL:       https://news.ycombinator.com
标题:      Hacker News
获取时间:  2026-01-24T10:30:00Z
提取进度:  分 3 次传递，已完成
链接数:    30
表单数:    2
图片数:    0
大小:      45 KB (html), 12 KB (parsed)
```

---

### `head <n>` / `tail <n>`

获取流中的前 n 项或后 n 项。

**语法：**
```
head <n>
head -n <n>          # 等同于 head <n>
tail <n>
tail -f              # 跟随（用于 watch/stream）
```

**是否可管道传输：** 是（必须在管道中或配合文件使用）

---

### `sort`

对输出行进行排序。

**语法：**
```
sort                 # 按字母顺序
sort -n              # 按数字顺序
sort -r              # 反向排序
sort -u              # 唯一（去重）
sort -k <n>          # 按第 n 个字段排序
sort -t <delim>      # 字段分隔符
```

**是否可管道传输：** 是

---

### `uniq`

去除重复行。

**语法：**
```
uniq
uniq -c              # 在行首加上出现次数
uniq -d              # 仅显示重复行
uniq -u              # 仅显示唯一行
```

**是否可管道传输：** 是

---

### `wc`

统计单词数、行数、字符数。

**语法：**
```
wc                   # 显示所有统计数据
wc -l                # 仅统计行数
wc -w                # 仅统计单词数
wc -c                # 仅统计字符数
wc -L                # 显示最长行的长度
```

**Web 特有选项：**
```
wc --links           # 统计链接数
wc --images          # 统计图片数
wc --forms           # 统计表单数
wc --headings        # 统计标题数
```

**是否可管道传输：** 是

---

### `cut`

从输出中提取列/字段。

**语法：**
```
cut -f <n>           # 第 n 个字段（从 1 开始）
cut -f <n,m>         # 第 n 和第 m 个字段
cut -d <delim>       # 分隔符（默认为制表符 tab）
cut -c <range>       # 字符位置范围
```

**示例：**
```
ls -l | cut -f 1     # 仅提取链接文本，不包含 URL
```

**是否可管道传输：** 是

---

### `tr`

转换/变换字符。

**语法：**
```
tr <set1> <set2>     # 将 set1 中的字符替换为 set2 中的字符
tr -d <set>          # 删除字符
tr -s <set>          # 压缩连续重复的字符
tr '[:upper:]' '[:lower:]'  # 转换为小写
```

**是否可管道传输：** 是

---

### `sed`

用于变换的流编辑器。

**语法：**
```
sed 's/old/new/'     # 替换第一次出现的内容
sed 's/old/new/g'    # 替换所有出现的内容
sed -n '5,10p'       # 打印第 5 到 10 行
sed '/pattern/d'     # 删除匹配的行
```

**是否可管道传输：** 是

---

### `source`

查看原始 HTML 源码。

**语法：**
```
source               # 完整 HTML
source | head 50     # 前 50 行
source -l            # 带行号
```

---

### `dom`

显示 DOM 树结构。

**语法：**
```
dom                  # 完整树结构
dom <selector>       # 从选择器开始的子树
dom -d <n>           # 深度限制
dom --tags           # 仅显示标签名
```

**输出：**
```
html
├── head
│   ├── title
│   ├── meta
│   └── link
└── body
    ├── header
    │   └── nav
    ├── main
    │   ├── article
    │   └── aside
    └── footer
```

---

## 预取与爬取 (Prefetching & Crawling)

### `prefetch`

控制主动链接爬取。默认情况下，当你导航到页面后，websh 会自动在后台预取 1-2 层深的可见链接。

**语法：**
```
prefetch                     # 显示状态
prefetch on                  # 开启主动爬取
prefetch off                 # 关闭主动爬取
prefetch <url>               # 手动预取某个 URL
prefetch --depth <n>         # 设置爬取深度（默认：2）
prefetch --stop              # 停止当前爬取任务
```

**示例：**
```
prefetch                     # 检查爬取进度
prefetch off                 # 在慢速网络下关闭
prefetch https://example.com # 手动将 URL 加入队列
```

**状态输出示例：**
```
主动爬取 (Eager crawl): 已开启
深度: 2, 仅限同域名: 是, 每页最大数量: 20

当前爬取状态:
  来源: https://news.ycombinator.com
  进度: 第 1 层 - 15/20 已完成
  队列: 第 2 层尚有 42 个 URL
```

---

### `crawl`

显式爬取某个 URL 到指定深度。

**语法：**
```
crawl <url>                  # 从该 URL 开始爬取
crawl --depth <n>            # 深度（默认：2）
crawl --all                  # 包含外部链接
crawl --follow <pattern>     # 仅跟随匹配的 URL
crawl --max <n>              # 最大获取页面数量
```

**示例：**
```
crawl https://docs.example.com --depth 3
crawl https://api.example.com --follow "/docs/*"
crawl https://blog.com --max 50
```

**与 prefetch 的区别：**
- `prefetch` 是自动的，在 `cd` 之后后台运行。
- `crawl` 是手动的，可以进行更深或更广的抓取。

---

### `queue`

显示爬取队列。

**语法：**
```
queue                        # 显示队列状态
queue -l                     # 详细格式，列出所有 URL
queue --clear                # 清空待处理队列
```

**输出示例：**
```
进行中: 3
  [→] https://hn.com/item?id=123 (正在提取)
  [→] https://hn.com/item?id=124 (正在获取)
  [→] https://hn.com/item?id=125 (正在获取)

队列中: 17
  [0] https://hn.com/item?id=126 (深度 1)
  [1] https://hn.com/item?id=127 (深度 1)
  ...

已完成: 12
已跳过: 5 (外部链接/已缓存)
```

---

## 搜索与发现 (Search & Discovery)

### `find <pattern>`

从当前页面开始递归搜索/爬取。

**语法：**
```
find <text-pattern>              # 搜索页面内容
find -name "<pattern>"           # 搜索链接文本
find -href "<pattern>"           # 搜索 URL
find -selector "<css>"           # 查找元素
find -depth <n>                  # 爬取 n 层深度
find -maxpages <n>               # 限制爬取的页面数量
find -type <t>                   # 过滤类型: link, image, form, heading
find -follow                     # 实际获取找到的页面
```

**示例：**
```
find "API documentation"                    # 在链接页面中查找文本
find -name "*.pdf" -depth 2                # 查找 2 层内的 PDF 链接
find -selector "form" -depth 1             # 查找当前及链接页面中的所有表单
find -href "/api/" -follow                 # 爬取所有包含 /api/ 的页面
```

**输出：** 带有来源页面的匹配项列表

---

### `locate <term>`

即时搜索所有缓存页面。

**语法：**
```
locate <pattern>
locate -i <pattern>  # 忽略大小写
locate -r <regex>    # 正则表达式模式
locate --urls        # 仅搜索 URL
locate --titles      # 仅搜索标题
```

**示例：**
```
locate "authentication"    # 在所有缓存内容中查找
locate -i "OAuth"          # 忽略大小写查找
```

**输出示例：**
```
news-ycombinator-com: [3 个匹配项]
  - "OAuth authentication flow..."
  - "...using authentication tokens..."
techcrunch-com-article: [1 个匹配项]
  - "...new authentication method..."
```

---

### `tree`

显示站点结构。

**语法：**
```
tree                 # 从当前页面开始
tree -d <n>          # 深度限制
tree -L <n>          # 同 -d
tree --sitemap       # 如果可用，使用 sitemap.xml
tree --infer         # 从链接中推断
tree -P <pattern>    # 仅显示匹配的路径
```

**输出示例：**
```
https://example.com/
├── /about
├── /products
│   ├── /products/widget
│   └── /products/gadget
├── /blog
│   ├── /blog/post-1
│   └── /blog/post-2
└── /contact
```

---

### `which <link>`

解析链接的实际去向（跟随重定向）。

**语法：**
```
which <url>
which <index>        # 通过 ls 输出中的编号指定
which -a             # 显示链条中的所有重定向
```

**输出示例：**
```
https://bit.ly/xyz → https://example.com/actual-page
```

使用 `-a`：
```
https://bit.ly/xyz
  → https://example.com/redirect
  → https://example.com/actual-page (200 OK)
```

---

## 比较与差异 (Comparison & Diff)

### `diff`

比较页面或版本。

**语法：**
```
diff <url1> <url2>           # 比较两个 URL
diff <url>                   # 比较当前页面与指定 URL
diff -c                      # 上下文格式
diff -u                      # 统一 (unified) 格式
diff --side-by-side          # 并排显示
diff --links                 # 仅比较链接
diff --text                  # 仅比较文本内容
```

**基于时间的比较：**
```
diff -t <duration>           # 与指定时长前的缓存版本比较
diff --wayback <date>        # 与 Wayback Machine 的快照比较
```

**示例：**
```
diff https://a.com https://b.com
diff -t 1h                   # 与 1 小时前比较
diff --wayback 2024-01-01    # 与 Wayback 快照比较
```

---

### `patch`

应用更改（适用于具有写权限的 API）。

**语法：**
```
patch <file>         # 应用差异文件
```

*注：需要挂载具有写权限的 API。*

---

## 监控 (Monitoring)

### `watch`

监视 URL 的变化。

**语法：**
```
watch <url>                  # 每 60 秒轮询一次
watch -n <seconds>           # 自定义时间间隔
watch -d                     # 高亮显示差异
watch --notify               # 发生变化时发送系统通知
watch --exec <cmd>           # 发生变化时运行命令
watch --selector <css>       # 仅监视特定的元素
```

**示例：**
```
watch https://status.example.com -n 30
watch -d --selector ".price"
watch --notify --exec "echo 'Changed!'"
```

**输出：** 显示内容，原地更新，并高亮显示变化部分

---

### `tail -f <url>`

流式传输实时内容（用于 SSE、Websocket 或轮询）。

**语法：**
```
tail -f <url>                # 流式传输更新
tail -f --sse                # 服务器发送事件 (Server-Sent Events)
tail -f --ws                 # WebSocket
tail -f --poll <n>           # 每 n 秒轮询一次
```

---

### `ping`

检查网站是否在线。

**语法：**
```
ping <url>
ping -c <n>          # ping 的次数
ping -i <seconds>    # 时间间隔
```

**输出示例：**
```
PING https://example.com
200 OK - 145ms
200 OK - 132ms
200 OK - 156ms
--- 3 requests, avg 144ms ---
```

---

### `traceroute`

显示重定向链。

**语法：**
```
traceroute <url>
```

**输出示例：**
```
1. https://short.link/abc (301)
2. https://example.com/redirect (302)
3. https://example.com/final (200)
```

---

### `time`

测量命令执行时间。

**语法：**
```
time <command>
```

**输出示例：**
```
[命令输出内容]

real    0.45s
fetch   0.32s
extract 0.13s
```

---

## 进程与作业管理 (Process & Job Management)

### `ps`

显示正在运行的后台任务。

**语法：**
```
ps                   # 列出所有任务
ps -l                # 详细格式
ps -a                # 所有任务（包含已完成的）
```

**输出示例：**
```
PID   状态        URL/任务
1     正在提取    news-ycombinator-com
2     正在获取    x-com-deepfates
3     正在监视    status.example.com
```

---

### `jobs`

列出后台作业。

**语法：**
```
jobs
jobs -l              # 包含 PID
jobs -r              # 仅显示正在运行的
jobs -s              # 仅显示已停止的
```

**输出示例：**
```
[1]  + 正在运行     cd https://example.com &
[2]  - 正在提取     follow 3 &
[3]    正在监视     watch https://status.com
```

---

### `kill`

取消后台任务。

**语法：**
```
kill <pid>
kill %<job-number>
kill -9 <pid>        # 强制结束
killall watch        # 结束所有 watch 进程
```

---

### `wait`

等待后台任务完成。

**语法：**
```
wait                 # 等待所有任务
wait <pid>           # 等待特定 PID
wait %<job>          # 等待特定作业编号
```

---

### `bg` / `fg`

将作业切换到后台/前台。

**语法：**
```
bg %<job>            # 在后台继续执行作业
fg %<job>            # 将作业切换到前台
```

---

### `&` (后台操作符)

在后台运行命令。

**语法：**
```
cd https://example.com &
watch https://status.com &
```

---

### `nohup`

运行不受挂断影响的命令。

**语法：**
```
nohup watch https://example.com &
```

---

## 环境与认证 (Environment & Auth)

### `env`

显示当前环境（头部、Cookie、设置）。

**语法：**
```
env                  # 所有变量
env | grep COOKIE    # 过滤
```

**输出示例：**
```
USER_AGENT=websh/1.0
ACCEPT=text/html
COOKIE_session=abc123
HEADER_Authorization=Bearer xyz
TIMEOUT=30
RATE_LIMIT=10/min
```

---

### `export`

设置环境变量（头部、Cookie）。

**语法：**
```
export VAR=value
export HEADER_X-Custom=value
export COOKIE_session=abc123
export USER_AGENT="Custom Agent"
export TIMEOUT=60
```

**示例：**
```
export HEADER_Authorization="Bearer mytoken"
export COOKIE_session="abc123"
export USER_AGENT="Mozilla/5.0..."
```

**爬取设置：**
```
export EAGER_CRAWL=true              # 开启/关闭预取
export CRAWL_DEPTH=2                 # 预取深度
export CRAWL_SAME_DOMAIN=true        # 仅预取同域名链接
export CRAWL_MAX_PER_PAGE=20         # 每页最大链接数
export CRAWL_MAX_CONCURRENT=5        # 并行获取数量
export CRAWL_DELAY_MS=200            # 频率限制延迟
```

---

### `unset`

移除环境变量。

**语法：**
```
unset VAR
unset HEADER_Authorization
unset COOKIE_session
```

---

### `whoami`

显示已登录的身份（如果可检测到）。

**语法：**
```
whoami
whoami -v            # 详细模式（显示检测方式）
```

**输出示例：**
```
@deepfates (检测自: meta 标签, cookie)
```

或：
```
(not logged in)
```

---

### `login`

交互式登录流程。

**语法：**
```
login                        # 登录当前站点
login <url>                  # 登录指定站点
login --form <selector>      # 指定登录表单
login -u <user> -p <pass>    # 提供凭据
login --cookie <file>        # 从文件导入 Cookie
login --browser              # 从浏览器导入
```

**流程：**
1. 检测登录表单
2. 提示输入凭据（或使用提供的凭据）
3. 提交表单
4. 存储会话 Cookie

---

### `logout`

清除当前站点的会话。

**语法：**
```
logout               # 当前站点
logout <domain>      # 指定域名
logout --all         # 所有会话
```

---

### `su`

切换用户/配置方案 (profile)。

**语法：**
```
su <profile>         # 切换到指定配置
su -                 # 切换到默认配置
su -l <profile>      # 作为指定配置登录（新会话）
```

配置方案存储独立的 Cookie、头部信息和身份信息。

---

## 挂载与虚拟文件系统 (Mounting & Virtual Filesystems)

### `mount`

将 API 或服务挂载为可浏览的目录。

**语法：**
```
mount <source> <mountpoint>
mount -t <type> <source> <mountpoint>
```

**类型 (Types)：**
- `rest` — REST API
- `github` — GitHub API
- `rss` — RSS/Atom 订阅源
- `json` — JSON 端点

**示例：**
```
mount https://api.github.com /gh
mount -t github octocat/Hello-World /repo
mount -t rss https://example.com/feed.xml /feed
mount -t rest https://api.example.com /api
```

**挂载后操作：**
```
cd /gh/users/octocat
ls                           # 列出用户属性
cat repos                    # 获取仓库列表
cd /gh/repos/octocat/Hello-World
ls issues
cat issues/1
```

---

### `umount`

卸载已挂载的路径。

**语法：**
```
umount <mountpoint>
umount -a            # 卸载所有
```

---

### `df`

显示已挂载的文件系统和缓存使用情况。

**语法：**
```
df
df -h                # 以易读的格式显示大小
```

**输出示例：**
```
挂载点           类型    大小    已用    配额
/               web     -       12MB    -
/gh             github  -       45KB    5000 req/hr (还剩 4892)
/api            rest    -       2KB     100 req/min (还剩 98)

缓存: 156 个页面, 45MB
```

---

### `quota`

显示频率限制状态。

**语法：**
```
quota
quota <domain>
```

**输出示例：**
```
api.github.com: 剩余 4892/5000 次请求 (45 分钟后重置)
api.twitter.com: 剩余 98/100 次请求 (12 分钟后重置)
```

---

## 归档与快照 (Archives & Snapshots)

### `tar`

对多个页面进行归档。

**语法：**
```
tar -c <file> <urls...>      # 创建归档
tar -c site.tar https://example.com/*   # 通配符
tar -x <file>                # 提取（恢复到缓存）
tar -t <file>                # 列出内容
tar -z                       # 压缩 (gzip)
```

**示例：**
```
tar -cz research.tar.gz https://paper1.com https://paper2.com
tar -t research.tar.gz
tar -x research.tar.gz       # 恢复到缓存
```

---

### `snapshot`

保存当前页面的带时间戳版本。

**语法：**
```
snapshot                     # 自动使用时间戳保存
snapshot <name>              # 使用指定名称保存
snapshot -l                  # 列出快照
snapshot -r <name>           # 恢复快照
```

**示例：**
```
snapshot "before-update"
# ... 时间流逝 ...
diff --snapshot "before-update"
```

---

### `wayback`

与 Wayback Machine 交互。

**语法：**
```
wayback <url>                # 列出可用快照
wayback <url> <date>         # 获取特定日期的快照
wayback --save <url>         # 请求 Wayback 进行归档
```

**示例：**
```
wayback https://example.com
wayback https://example.com 2023-06-15
cd $(wayback https://example.com 2020-01-01)
```

---

## 站点元数据 (Site Metadata)

### `robots`

显示 robots.txt 内容。

**语法：**
```
robots
robots <url>
```

---

### `sitemap`

显示/解析 sitemap.xml 内容。

**语法：**
```
sitemap
sitemap <url>
sitemap --urls       # 仅列出 URL
sitemap --tree       # 以树状结构显示
```

---

### `headers`

显示 HTTP 响应头部。

**语法：**
```
headers              # 当前页面
headers <url>        # 仅获取头部（HEAD 请求）
headers -v           # 详细模式（请求 + 响应）
```

---

### `cookies`

管理 Cookie。

**语法：**
```
cookies              # 列出当前域名的 Cookie
cookies <domain>     # 列出指定域名的 Cookie
cookies -a           # 所有域名
cookies --set <name>=<value>
cookies --delete <name>
cookies --clear      # 清除域名的所有 Cookie
cookies --export <file>
cookies --import <file>
```

---

## 交互命令 (Interaction)

### `click`

模拟点击元素。

**语法：**
```
click <selector>
click <index>        # 通过 ls 输出中的编号指定
click --js           # 执行 onclick 处理器
```

*注：在没有完整浏览器的情况下功能受限，尽力而为。*

---

### `submit`

提交表单。

**语法：**
```
submit <form-selector>
submit -d "field=value&field2=value2"
submit --json '{"field": "value"}'
```

**交互模式：**
```
submit               # 如果只有一个表单，将提示输入字段
```

---

### `type`

填写输入框。

**语法：**
```
type <selector> "text"
type --clear <selector>      # 先清空再填写
```

---

### `scroll`

触发滚动/翻页。

**语法：**
```
scroll               # 向下滚动（触发无限滚动）
scroll --bottom      # 滚动到底部
scroll --page <n>    # 前往第 n 页
scroll --next        # 下一页
```

*注：在没有完整浏览器的情况下功能受限。*

---

### `screenshot`

捕获视觉快照（需要浏览器工具）。

**语法：**
```
screenshot <file>
screenshot --full    # 全屏
screenshot --selector <css>  # 指定元素
```

---

## 调度 (Scheduling)

### `cron`

安排定期执行的命令。

**语法：**
```
cron "<schedule>" <command>
cron -l              # 列出已安排的任务
cron -r <id>         # 移除任务
```

**示例：**
```
cron "0 * * * *" 'watch https://status.com --notify'
cron "0 9 * * *" 'cd https://news.com && ls | head 5 > daily.txt'
```

---

### `at`

安排一次性执行的命令。

**语法：**
```
at <time> <command>
at -l                # 列出待处理任务
at -r <id>           # 移除
```

**示例：**
```
at "10:00" 'refresh'
at "+1h" 'snapshot "hourly"'
at "2024-12-25 00:00" 'cd https://xmas.com'
```

---

## 别名与脚本 (Aliases & Scripts)

### `alias`

创建命令快捷方式。

**语法：**
```
alias <name>='<command>'
alias                # 列出所有别名
alias <name>         # 显示特定别名
unalias <name>       # 移除别名
```

**示例：**
```
alias hn='cd https://news.ycombinator.com'
alias top5='ls | head 5'
alias search='grep -i'
```

---

### `ln -s`

创建 URL 别名/符号链接。

**语法：**
```
ln -s <url> <name>
```

**示例：**
```
ln -s https://news.ycombinator.com hn
cd hn                # 效果等同于 cd https://news.ycombinator.com
```

---

## 状态命令 (State Commands)

### `history`

显示命令历史记录。

**语法：**
```
history
history <n>          # 最近 n 条命令
history -c           # 清除历史记录
history | grep <pattern>
!<n>                 # 执行历史记录中第 n 条命令
!!                   # 重复最后一条命令
```

---

### `bookmark [name]`

将 URL 保存为书签。

**语法：**
```
bookmark              # 从域名自动命名
bookmark <name>
bookmark -d <name>    # 删除
bookmark -l           # 列出（等同于 bookmarks）
```

---

### `bookmarks`

列出所有书签。

**语法：**
```
bookmarks
bookmarks | grep <pattern>
```

---

### `go <bookmark>`

导航到书签。

**语法：**
```
go <name>
```

---

## 文件操作 (File Commands)

### `save`

将页面保存到本地文件。

**语法：**
```
save <path>                  # 保存 HTML
save <path> --parsed         # 保存提取后的 markdown
save <path> --complete       # 连同资源文件一起保存
```

---

### `tee`

在显示的同时保存输出内容。

**语法：**
```
<command> | tee <file>
<command> | tee -a <file>    # 追加
```

**示例：**
```
ls | grep "AI" | tee ai-links.txt
```

---

### `xargs`

从输入构建并执行命令。

**语法：**
```
<command> | xargs <cmd>
<command> | xargs -I {} <cmd> {}
<command> | xargs -P <n>     # 并行执行
```

**示例：**
```
cat urls.txt | xargs -I {} cd {}
ls | head 5 | xargs -P 5 follow    # 并行获取前 5 个链接
```

---

### `parallel`

并行运行命令。

**语法：**
```
parallel <cmd> ::: <args...>
parallel -j <n>              # 同时进行的作业数量
```

**示例：**
```
parallel cd ::: https://a.com https://b.com https://c.com
```

---

## 帮助与文档 (Help & Documentation)

### `help`

显示帮助信息。

**语法：**
```
help                 # 通用帮助
help <command>       # 特定命令的帮助
```

---

### `man`

详细手册（或获取站点的 API 文档）。

**语法：**
```
man <command>        # websh 命令手册
man <domain>         # 尝试获取该域名的 API 文档
```

---

## 特殊语法 (Special Syntax)

### 管道 (Pipes)

命令可以串联：
```
ls | grep "AI" | head 3 | tee results.txt
```

### 后台运行 (Background)

在末尾追加 `&` 以在后台运行：
```
cd https://slow-site.com &
```

### 命令替换 (Command substitution)

使用 `$()` 替换命令输出：
```
cd $(wayback https://example.com 2020-01-01)
diff $(locate "config" | head 1) $(locate "config" | tail 1)
```

### 通配符模式 (Glob patterns)（针对缓存页面）

```
locate "error" --in "api-*"      # 在匹配 api-* 的页面中搜索
tar -c backup.tar news-*         # 归档所有新闻页面
```

### 选择器 (Selectors)

在命令中使用 CSS 选择器：
```
cat .article-body
ls nav a
cat h1:first
click button.submit
```

---

## 错误消息 (Error Messages)

| 错误 | 含义 |
|-------|---------|
| `error: no page loaded` | 请先运行 `cd <url>` |
| `error: selector not found` | 没有匹配该选择器的元素 |
| `error: fetch failed` | 网络错误 |
| `error: rate limited` | 请求过于频繁，触发频率限制 |
| `error: outside chroot` | URL 超出了 chroot 的边界范围 |
| `error: mount failed` | 无法挂载 API |
| `error: permission denied` | 需要身份认证 |
| `error: job not found` | 无效的 PID 或作业编号 |