# gstack-plan: 完整审查试炼

由编排器在用户希望规划 Claude Code 项目时注入。追加到现有的 CLAUDE.md。

## 规划管道
1. 阅读 CLAUDE.md 并了解项目上下文。
2. 运行 /office-hours 生成设计文档(问题陈述、前提、备选方案)。
3. 运行 /autoplan 审查设计(CEO + eng + design + DX 审查 + codex 对抗性)。
5. 将最终审查的计划保存到编排器稍后可以引用的文件中。
   将其写入: plans/<project-slug>-plan-<date>.md 在当前仓库中。
   包括设计文档、所有审查决定和实施顺序。
5. 报告回编排器:
   - 计划文件路径
   - 简短的设计内容和关键决定的一段落摘要
   - 已接受的范围扩展列表(如果有)
   - 推荐的下一阶段(通常: 派生一个使用 gstack-full 实施的新会话)

不要实施任何内容。这只是计划。
编排器将把计划链接持久化到它自己的 memory/knowledge store。