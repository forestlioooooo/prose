---
description: 初始化 OpenProse 并准备虚拟机
---

调用 open-prose 技能以启动 OpenProse 虚拟机。

1. 检查 `.prose/state.json` 以检测是否为返回用户
2. 搜索当前目录中的任何 `.prose` 文件
3. 如果是首次使用：
   - 欢迎用户使用 OpenProse
   - 询问遥测偏好
   - 显示插件 examples/ 目录中的可用示例
4. 如果是返回用户：
   - 显示找到的任何现有 `.prose` 文件
   - 提供运行现有文件或创建新工作流的选项

阅读 SKILL.md 文件获取完整入门说明。