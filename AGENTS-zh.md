# 项目知识库

**生成时间:** 2026-03-14  
**提交:** 35b3b60  
**分支:** main

## 概述
OpenProse 是一个用于 AI 会话的编程语言。运行时是一个体现 OpenProse 虚拟机的 AI 助手。核心技术栈：OpenProse DSL (.prose 文件)、Markdown 规范、通过 Task 工具进行代理编排。

## 必须遵守的规则
- **使用中文**: 思考过程与输出结果、所有文档、代码注释等均使用中文编写。

## 结构
```
./
├── skills/              # 技能 (open-prose, websh)
├── commands/           # CLI 命令定义
├── assets/             # 图片资源
├── .claude-plugin/     # 插件元数据
└── *.md                # 项目文档
```

## 查找指南
| 任务 | 位置 | 说明 |
|------|----------|-------|
| 运行 .prose 程序 | `skills/open-prose/SKILL.md` | 激活和路由规则 |
| 虚拟机规范 | `skills/open-prose/prose.md` | 执行语义 |
| 语言语法 | `skills/open-prose/compiler.md` | 验证和编译 |
| 示例程序 | `skills/open-prose/examples/` | 51 个示例程序 |
| 共享工具 | `skills/open-prose/lib/`, `skills/open-prose/common/` | 标准库和公共程序 |
| 状态管理 | `skills/open-prose/state/` | 文件系统、上下文内、SQLite、PostgreSQL |
| CLI 命令 | `commands/` | prose-run, prose-compile, prose-boot |
| Websh 技能 | `skills/websh/` | 用于 URL 导航的 Web shell |

## 约定
- **文档优先**: 运行时行为由 Markdown 规范定义，而非编译代码
- **无包清单**: 没有 package.json、pyproject.toml 等
- **技能激活**: 单个 `open-prose` 技能处理所有 `prose` 命令
- **状态模式**: 默认文件系统，可选上下文内/SQLite/PostgreSQL
- **虚拟机/子代理契约**: 子代理写入绑定文件，仅返回指针
- **自由裁量标记**: `**...**` 用于 AI 评估的条件
- **紧凑叙述**: 上下文内模式使用最小标记以节省令牌

## 反模式（本项目）
- **禁止** 在编译期间执行（仅静态分析）
- **禁止** 在 websh 中阻塞主线程（委托给后台 haiku）
- **禁止** 直接编写生产代码（船长模式）
- **禁止** 在用户工作区搜索 OpenProse 文档（它们与 SKILL.md 位于同一位置）
- **禁止** 在 PostgreSQL URL 中存储敏感凭证（对子代理可见）

## 独特风格
- **第四面墙语法**: `**...**` 跳出结构以进行 AI 判断
- **智能控制反转**: 会话通过理解而非仅配置来连接代理
- **模拟即实现**: LLM 以足够保真度模拟 VM 规范即成为运行时
- **主动链接抓取**: websh 预取 2 层深的链接以实现即时导航

## 命令
```bash
# 安装技能
npx skills add openprose/prose

# 运行程序
prose run examples/01-hello-world.prose

# 编译/验证
prose compile my-program.prose

# 初始化
prose boot

# 更新工作区
prose update

# Websh
websh cd https://news.ycombinator.com
```

## 注意事项
- **测试版软件**: 可能存在错误，使用时需谨慎，提供反馈
- **安全**: PostgreSQL 凭证对子代理可见；使用专用数据库
- **无 CI**: 缺少 .github/workflows 和 Makefile（技能分发有意如此）
- **测试模式**: 示例兼具测试功能；UX 测试模式位于 examples/44-run-endpoint-ux-test.prose
- **贡献**: 参见 CONTRIBUTING.md；接受来自代理的 PR
