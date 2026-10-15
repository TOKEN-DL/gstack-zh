<!-- AUTO-GENERATED from shared-code-reuse.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
**仅使用完整的结构证据重用跳过的共享代码建议:**

1. **读取证据。** 读取所有支撑的调用者和辅助函数目标。建立第一方作者来源和当前提取是否值得;检查器无法决定。保留 `evidence_paths`/`helper_target`。
2. **运行检查器。** 从仓库根开始,将当前发现作为字面 JSON 在 stdin 上传递。将 REVIEW_START 替换为此次流程捕获的令牌,并将示例路径/符号替换为实际证据。保留引号分隔符。

```bash
"$HOME/.claude/skills/gstack/bin/gstack-review-log" --check-shared-libs REVIEW_START <<'GSTACK_SHARED_LIBS_REUSE_JSON'
{"advisory":true,"severity":"INFORMATIONAL","evidence_paths":["src/caller-a.ts","src/caller-b.ts"],"helper_target":{"path":"src/shared.ts","symbol":"sharedHelper"}}
GSTACK_SHARED_LIBS_REUSE_JSON
```

3. **根据其结果行动。** 读取 JSON。只有 `reusable: true` 允许抑制。False、命令失败或不可读的输出需要新的源代码审查和新的决策,绝不抑制。不要提供你自己的快照、先前的记录或覆盖。
4. **通过记录器持久化。** 记录器重新计算最终覆盖;绝不要自己提供证明。真实缺陷独立地保留正常的先修复处理。

**可重用结果证明的内容(不要自己重建这些检查):**
- 身份: `sharedLibsFingerprint` 加上实际仓库、原始分支和当前快照。检查器读取 REVIEW_START 而不消耗/替换它。已消毒的分支名称不是身份。
- 先前决策: 已完成/已收敛的审查、已验证的绑定、显式跳过和记录器版本化的 `snapshot_covered_paths`;较旧的未版本化覆盖需要新决策。
- 来源: `canReuseSharedLibsAdvisory` 要求每个支撑路径的原始文件按字节与其 blob 匹配。排除 assume-unchanged、skip-worktree 和稀疏索引条目;符号链接/祖先、子模块、被忽略/外部或不可读的文件;活动/未知的 Git 过滤器、编码和行转换。
- 安全检查: 禁用 fsmonitor 和可选锁;绝不使用外部 diff/textconv。未知证据失败关闭。