---
name: guard
version: 0.1.0
description: "完整安全模式：破坏性命令警告 + 目录范围内的编辑限制。（gstack）"
triggers:
  - full safety mode
  - guard against mistakes
  - maximum safety
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: 'bash -c "exec bash \"$HOME/.claude/skills/gstack/careful/bin/check-careful.sh\""'
          statusMessage: "Checking for destructive commands..."
    - matcher: "PowerShell"
      hooks:
        - type: command
          command: 'bash -c "exec bash \"$HOME/.claude/skills/gstack/careful/bin/check-careful.sh\""'
          statusMessage: "Checking for destructive commands..."
    - matcher: "Edit"
      hooks:
        - type: command
          command: 'bash -c "exec bash \"$HOME/.claude/skills/gstack/freeze/bin/check-freeze.sh\""'
          statusMessage: "Checking freeze boundary..."
    - matcher: "Write"
      hooks:
        - type: command
          command: 'bash -c "exec bash \"$HOME/.claude/skills/gstack/freeze/bin/check-freeze.sh\""'
          statusMessage: "Checking freeze boundary..."
    - matcher: "NotebookEdit"
      hooks:
        - type: command
          command: 'bash -c "exec bash \"$HOME/.claude/skills/gstack/freeze/bin/check-freeze.sh\""'
          statusMessage: "Checking freeze boundary..."
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

把 /careful（在执行 rm -rf、DROP TABLE、force-push 等之前发出警告）和 /freeze（禁止在指定目录之外编辑）合并在一起。当最大安全地操作生产环境，或调试在线系统时使用。当用户要求 "guard mode"、"full safety"、"lock it down" 或 "maximum safety" 时使用。

# /guard — 完整安全模式

同时激活破坏性命令警告和目录范围的编辑限制。这是 `/careful` + `/freeze` 的组合，单条命令搞定。

**依赖说明：** 本 skill 引用了同级 `/careful` 和 `/freeze` skill 目录中的 hook 脚本。两者必须都被安装（它们由 gstack setup 脚本一起安装）。

```bash
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
mkdir -p "$GSTACK_STATE_ROOT"/analytics
echo '{"skill":"guard","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> "$GSTACK_STATE_ROOT"/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

询问用户要将编辑限制在哪个目录。使用 AskUserQuestion:

- 问题："Guard mode: which directory should edits be restricted to? Destructive command warnings are always on. Files outside the chosen path will be blocked from editing."
- 文本输入（不是多选）—— 用户输入一个路径。

用户给出目录路径后：

用共享的写入器设置用户选定的 boundary，它会把路径解析为物理绝对路径,并与 investigation 清理的替换串行化：
```bash
bash "$HOME/.claude/skills/gstack/freeze/bin/freeze-state.sh" set "<user-provided-path>"
```

如果辅助脚本失败,不要声称 boundary 已生效。保留状态并报告恢复情况；绝不能绕过共享写入器直接写入或删除。

告诉用户：
- "**Guard mode active.** Two protections are now running:"
- "1. **Destructive command guard** — rm -rf, DROP TABLE, force-push, etc. warn before executing (overridable); catastrophic shapes (recursive delete of / or ~, force-push to the default branch) are hard-denied"
- "2. **Edit boundary** — file edits restricted to `<path>/`. Edits outside this directory are blocked."
- "To remove the persistent edit boundary, run `/unfreeze`. Ending the session stops its hooks but does not delete that boundary."

## 被保护的内容

破坏性命令模式和安全的例外清单，请参阅 `/careful`。
Edit boundary 的执行细节,请参阅 `/freeze`。
