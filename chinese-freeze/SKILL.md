---
name: freeze
version: 0.1.0
description: 在本次会话中将文件编辑限制到指定目录。(gstack)
triggers:
  - freeze edits to directory
  - lock editing scope
  - restrict file changes
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
hooks:
  PreToolUse:
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
<!-- AUTO-GENERATED from SKILL.md.tmpl ; do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## 何时调用该技能

阻止 Edit、Write 和 NotebookEdit 操作允许路径之外的文件。在调试时使用,以防止意外「修复」无关代码,而是想要把更改范围限定在一个模块。
在用户说「freeze」、「restrict edits」、「only edit this folder」或「lock down edits」时使用。

# /freeze ; 把编辑限制到某个目录

把文件编辑锁定到指定目录。任何 Edit、Write 或 NotebookEdit 操作,只要目标文件不在允许路径内,都将被**拦截**(不仅仅是提示)。

```bash
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
mkdir -p "$GSTACK_STATE_ROOT"/analytics
echo '{"skill":"freeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> "$GSTACK_STATE_ROOT"/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

询问用户要将编辑限制到哪个目录。使用 AskUserQuestion:

- 问题:「我应该把编辑限制到哪个目录?此路径之外的文件将被阻止编辑。」
- 文本输入(非多选) ; 用户输入路径。

一旦用户提供了目录路径:

使用共享状态写入器设置用户选择的边界。它解析物理绝对路径并序列化与调查清理的替换:
```bash
bash "$HOME/.claude/skills/gstack/freeze/bin/freeze-state.sh" set "<user-provided-path>"
```

仅当助手成功时才报告成功。在 `FREEZE_BUSY` 或意外状态时,保留它并要求用户在活动写入器完成后检查恢复;永远不要直接写入或删除状态文件。

告诉用户:「编辑现已限制到 `<path>/`。任何一个 Edit、Write 或 NotebookEdit 操作,只要目标文件不在此目录内,都将被阻止。要更改边界,请再次运行 `/freeze`。要移除边界,请运行 `/unfreeze`。」

## 工作原理

钩子从 Edit/Write 工具输入 JSON 中读取 `file_path`,或从 NotebookEdit(Jupyter notebook)调用中读取 `notebook_path`(与 /careful 共享的真实 JSON 提取器 ; 一个副本,被两个钩子引用),然后检查路径是否以冻结目录开头。如果不是,它返回一个 `hookSpecificOutput` 载荷,带有 `permissionDecision: "deny"` 来阻止操作(嵌套在 `hookSpecificOutput` 下 ; Claude Code 忽略顶层 `permissionDecision`)。

极性是故障关闭(fail-closed):钩子无法解析的工具载荷会被拒绝(denied),而不是允许(allowed) ; 失败开放的安全机制不是真正的安全机制。能解析但既没有路径字段(非文件工具)的载荷被允许。拒绝消息会命名工具、路径字段、边界和 `/unfreeze`。符号链接通过其最终组件解析,因此边界内但目标实指向边界外的符号链接会按目标符号链进行检查。

冻结边界会一直存在,直到显式移除。钩子脚本在每次 Edit/Write/NotebookEdit 调用时读取它。支持含空格的边界。

## 备注

- 冻结目录末尾的 `/` 可防止 `/src` 匹配到 `/src-old`。
- 冻结仅适用于 Edit、Write 和 NotebookEdit ; Read、Bash、PowerShell、Glob、Grep 不受影响。
- 这能防止意外编辑,不是安全边界 ; 像 `sed` 或 `Set-Content` 这样的 Bash 或 PowerShell 命令仍可修改边界之外的文件。
- 要停用,请运行 `/unfreeze`;结束或终止对话不会移除已持久化的状态。