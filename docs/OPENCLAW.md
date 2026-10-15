# gstack x OpenClaw 集成

gstack 作为方法论源与 OpenClaw 集成,而不是移植的代码库。OpenClaw 的 ACP(Agent Communication Protocol,智能体通信协议)运行时本地 spawn Claude Code 会话。gstack 提供使这些会话更好的规划纪律和方法论。

这是一个编码为提示文本的轻量级协议。没有守护进程。没有 JSON-RPC。没有兼容性矩阵。提示就是桥梁。

## 架构

```
  OpenClaw                               gstack repo
  ─────────────────────                    ──────────────
  Orchestrator: messaging,                 Source of truth for
  calendar, memory, EA                     methodology + planning
       │                                        │
       ├── Native skills (conversational)       ├── Generates native skills
       │   office-hours, ceo-review,            │   via gen-skill-docs pipeline
       │   investigate, retro                   │
       │                                        ├── Generates gstack-lite
       ├── sessions_spawn(runtime: "acp")       │   (planning discipline)
       │       │                                │
       │       └── Claude Code                  ├── Generates gstack-full
       │           └── gstack installed at      │   (complete pipeline)
       │               ~/.claude/skills/gstack  │
       │                                        └── docs/OPENCLAW.md (this file)
       └── Dispatch routing (AGENTS.md)
```

## 派发路由

OpenClaw 在 spawn 时决定使用哪一级 gstack 支持:

| 层级(Tier) | 何时使用 | 提示前缀 |
|------|------|---------------|
| **Simple(简单)** | 单文件编辑、拼写错误、配置更改 | 不注入 gstack 上下文 |
| **Medium(中等)** | 多文件功能、重构 | 追加 gstack-lite CLAUDE.md |
| **Heavy(重量级)** | 需要特定的 gstack skill | "Load gstack. Run /X" |
| **Full(完整)** | 完整功能、目标、项目 | 追加 gstack-full 流水线 |
| **Plan(计划)** | "帮助我规划 Claude Code 项目" | 追加 gstack-plan 流水线 |

### 决策启发式

- 可以在 <10 行代码中完成吗? -> **Simple**
- 涉及多个文件但方法显而易见吗? -> **Medium**
- 用户命名了特定的 skill(/cso、/review、/qa)吗? -> **Heavy**
- 它是一个功能、项目还是目标(不是任务)? -> **Full**
- 用户是否希望在不实现的情况下为 Claude Code 规划某些内容? -> **Plan**

### 派发路由指南(适用于 AGENTS.md)

完整的可粘贴部分位于 `openclaw/agents-gstack-section.md`。将其复制到你的 OpenClaw AGENTS.md 中。

关键行为规则(这些放在派发层之上):

1. **始终 spawn,从不重定向。** 当用户要求使用任何 gstack skill 时,始终 spawn 一个 Claude Code 会话。永远不要告诉用户打开 Claude Code。
2. **解析仓库。** 如果用户命名了仓库,请设置工作目录。如果未知,请询问哪个仓库。
3. **Autoplan 端到端运行。** Spawn,让它运行完整的流水线,然后在聊天中报告回来。用户永远不应该离开 Telegram。

### CLAUDE.md 冲突处理

当在已有 CLAUDE.md 的仓库中 spawn Claude Code 时,将 gstack-lite/full 作为新部分追加。不要替换仓库的现有指令。

## gstack 为 OpenClaw 生成什么

所有工件都存在于 `openclaw/` 目录中,由 `bun run gen:skill-docs --host openclaw` 生成:

### gstack-lite(中等层)
`openclaw/gstack-lite-CLAUDE.md` ——约 15 行规划纪律:
1. 修改前阅读每个文件
2. 写一个 5 行计划:什么、为什么、哪些文件、测试用例、风险
3. 使用决策原则解决歧义
4. 报告完成前自审
5. 完成报告:交付了什么、做了什么决策、任何不确定的内容

A/B 测试:2 倍时间,显著更好的输出。

### gstack-full(完整层)
`openclaw/gstack-full-CLAUDE.md` ——链接现有的 gstack skills:
1. 阅读 CLAUDE.md 并理解项目
2. 运行 /autoplan(CEO + eng + design 评审)
3. 实现已批准的计划
4. 运行 /ship 以创建 PR
5. 使用 PR URL 和决策报告回来

### gstack-plan(计划层)
`openclaw/gstack-plan-CLAUDE.md` ——完整评审 gauntlet,不实现:
1. 运行 /office-hours 以生成设计文档
2. 运行 /autoplan(CEO + eng + design + DX 评审 + codex 对抗)
3. 将评审后的计划保存到 `plans/<project-slug>-plan-<date>.md`
4. 报告回来:计划路径、摘要、关键决策、建议的下一步

编排器将计划链接持久化到其自己的记忆存储(大脑仓库、知识库或在 AGENTS.md 中配置的任何内容)。当用户准备好构建时,spawn 一个引用已保存计划的 FULL 会话。

### 原生方法论技能
发布到 ClawHub。使用 `clawhub install` 安装:
- `gstack-openclaw-office-hours` ——产品询问(6 个强制性问题)
- `gstack-openclaw-ceo-review` ——战略挑战(10 节评审,4 种模式)
- `gstack-openclaw-investigate` ——操作调试(4 阶段方法论)
- `gstack-openclaw-retro` ——操作回顾(每周评审)

源代码存在于 gstack 仓库的 `openclaw/skills/` 中。这些是为 OpenClaw 的对话上下文手工制作的方法论适配。没有 gstack 基础设施(无浏览器、无遥测、无 preamble)。

## Spawned 会话检测

当 Claude Code 在 OpenClaw spawn 的会话中运行时,应设置 `OPENCLAW_SESSION` 环境变量。gstack 检测到这一点并调整:
- 跳过交互式提示(自动选择推荐选项;永远不会自动选择破坏性或不可逆的选项——保守的选择获胜并记录在完成报告中)
- 在发出时抑制交互式入门说明块(升级检查、遥测提示、功能发现、路由注入、提示),以便一次性提示在下次人工会话中保持不变
- 抑制 Conductor 散文信号(`CONDUCTOR_SESSION: true`)——Conductor 工作区中的 spawned 会话自动选择而不是向无人渲染散文
- 专注于任务完成和散文报告

`sessions_spawn` 没有每个 spawn 的环境参数([openclaw/openclaw#51654](https://github.com/openclaw/openclaw/issues/51654) 在未添加环境参数的情况下已关闭),因此工具调用中的 `env` 字段会被忽略。在 Claude ACP harness 或 Gateway 服务的环境中设置 `OPENCLAW_SESSION=1`,将凭据保留在该包装器之外,并从 spawned 进程内部确认该变量(例如 `echo $OPENCLAW_SESSION`),而不是信任会话的最终消息。

### 显式覆盖:GSTACK_SESSION_KIND

`GSTACK_SESSION_KIND=spawned` 是同一分类的显式每命令标记,胜过每个环境标记(包括 `OPENCLAW_SESSION` 和 `GSTACK_HEADLESS`)。它的存在是因为 Claude Code 子智能体按字节继承父会话的环境(#2733)——派发 skill 在同一命令行上通过前缀 preamble 调用来标记其子智能体:

```bash
GSTACK_SESSION_KIND=spawned "$_SS" --skill "document-release" ...
```

gstack 本身使用它:`/ship` 步骤 14.5 在最终提交、验证和发布之前用此前缀派发 `/document-release` 子智能体。其 ship-owned 范围覆盖通用的 spawned auto-choice:有风险或不确定的文档更改作为父级的阻塞器返回,无需交互式问题或自动批准。有意限制:仅 `spawned` 受尊重——`headless` 已经有了 `GSTACK_HEADLESS`,让环境变量通过 CI 标记强制 `interactive` 将是一个误分类陷阱。空或其他值是保留的并被忽略(回落到环境检测)。请注意,hook 进程继承 harness 环境,因此每命令前缀永远不会到达 PreToolUse/PostToolUse hooks——hook 文本为该拓扑携带 spawned 转义句子(`hosts/claude/hooks/spawned-directive.ts`)。

**防篡改可见性。** 任何注入会话范围环境的机制(克隆仓库的 `.claude/settings.json` env 块、direnv、CI 包装器)都可能为真实的人工会话设置 `GSTACK_SESSION_KIND=spawned` 并默默地将其确认 gate 翻转为 auto-choice。当环境覆盖驱动分类时,preamble 发出响亮的 `SPAWNED_OVERRIDE: env` 状态行,以便脚本显示会话被 spawn 的原因——审计不受信任仓库中的 `.claude/settings.json` env 块(/cso 涵盖这一点)。

## 安装

对于 OpenClaw 用户:告诉你的 OpenClaw 智能体"为 openclaw 安装 gstack"。

智能体应该:
1. 将 gstack-lite CLAUDE.md 安装到其编码会话模板中
2. 安装 4 个原生方法论技能
3. 将派发路由添加到 AGENTS.md
4. 使用测试 spawn 进行验证

对于 gstack 开发者:`./setup --host openclaw` 输出此文档。实际工件由 `bun run gen:skill-docs --host openclaw` 生成。

## 我们不做什么

- 无派发守护进程(ACP 处理会话 spawn)
- 无 Clawvisor 中继(不需要安全层)
- 无双向学习桥梁(大脑仓库是知识存储)
- 无 JSON schema 或协议版本控制
- 来自 gstack 的无 SOUL.md(OpenClaw 有自己的)
- 无完整 skill 移植(编码 skill 保持对 Claude Code 的原生)
