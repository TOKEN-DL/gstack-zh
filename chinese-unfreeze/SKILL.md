---
name: unfreeze
version: 0.1.0
description: 清除由 /freeze 设置的冻结边界(允许再次编辑所有目录)。 (gstack)
triggers:
  - unfreeze edits
  - unlock all directories
  - remove edit restrictions
allowed-tools:
  - Bash
  - Read
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

用于在不结束会话的情况下放宽编辑范围。
用于在用户请求 "unfreeze"、"unlock edits"、"remove freeze" 或
"allow all edits" 时调用。

# /unfreeze — 清除冻结边界

移除 `/freeze` 所设置的编辑限制,允许编辑所有目录。

```
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
mkdir -p "$GSTACK_STATE_ROOT"/analytics
echo '{"skill":"unfreeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> "$GSTACK_STATE_ROOT"/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 清除边界

```
bash "$HOME/.claude/skills/gstack/freeze/bin/freeze-state.sh" clear
```

这是用户显式请求的移除,而非调查清理。共享写入器(actor)将其与获取(acquisition)、替换以及所有者校验后的释放串行化。遇到 `FREEZE_BUSY` 或意外状态时,保持一切不动并报告恢复情况;不要直接删除状态或可能是活动中的锁。

向用户告知结果。注意 `/freeze` hooks 仍会在该会话中注册;只是因为没有状态文件所以会放行一切。如需重新冻结,再运行 `/freeze` 即可。