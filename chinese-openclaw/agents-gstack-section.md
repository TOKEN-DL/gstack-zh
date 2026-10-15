## 编码任务(gstack)

### 规则(不可协商)

1. **始终派生,绝不重定向。** 当用户要求使用任何 gstack 技能时,始终通过 sessions_spawn 派生 Claude Code 会话。永远不要告诉用户自己打开 Claude Code。永远不要说 "this needs to run in Claude Code." 永远不要说 "you'll need to open Claude Code for that." 直接去做。

2. **解析仓库。** 如果用户命名了仓库或项目,请将工作目录设置为该仓库路径。如果仓库路径未知,请询问哪个项目 — 不要推卸告诉用户打开 Claude Code。

3. **Autoplan 端到端运行。** 对于 /autoplan: 派生会话,让它运行完整的审查管道(CEO → design → eng),并在完成时将计划放在此处聊天中。将计划写入 memory,以便用户稍后可以找到它。用户永远不应该在 Telegram 中离开。

### 调度路由

当要求进行编码工作时,请选择那些离散的分派层:

**SIMPLE:** "fix this typo," "update that config," 单文件更改
→ sessions_spawn(runtime: "acp", prompt: "<just the task>")

**MEDIUM:** 多文件功能、重构、技能编辑
→ sessions_spawn(runtime: "acp", prompt: "<gstack-lite content>\n\n<task>")

**HEAVY:** 需要特定的 gstack 方法
→ sessions_spawn(runtime: "acp", prompt: "Load gstack. Run /qa https://...")
  Skills: /cso, /review, /qa, /ship, /investigate, /design-review, /benchmark, /gstack-upgrade

**FULL:** 构建完整的功能、多日范围、需要规划 + 审查
→ sessions_spawn(runtime: "acp", prompt: "<gstack-full content>\n\n<task>")
  Claude Code 运行: /autoplan → implement → /ship → report back

**PLAN:** 用户希望规划一个 Claude Code 项目,详细说明一个功能,或在编写任何代码之前设计某些东西
→ sessions_spawn(runtime: "acp", prompt: "<gstack-plan content>\n\n<task>")
  Claude Code 运行: /office-hours → /autoplan → saves plan file → reports back
  将计划链接持久化到 memory/knowledge store。
  当用户准备好实施时,派生一个指向该计划的新 FULL 会话。

### 决策启发式

- 可以用少于 10 行代码完成吗? → **SIMPLE**
- 是否触及多文件但方法很明显? → **MEDIUM**
- 用户是否命名特定的技能(/cso、/review、/qa)? → **HEAVY**
- "Upgrade gstack", "update gstack" → **HEAVY** 使用 `Run /gstack-upgrade`
- 是否是功能、项目或目标(而不是任务)? → **FULL**
- 用户是否希望在尚不实施的情况下计划某些东西? → **PLAN**