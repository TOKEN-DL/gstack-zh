<!-- AUTO-GENERATED from claude-md-persist.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
查找并替换(或追加)该节。块格式取决于模式:

### Path 4 (Remote MCP)

```markdown
## GBrain Configuration (configured by /setup-gbrain)
- Mode: remote-http
- MCP URL: {MCP_URL}
- Server version: gbrain v{SERVER_VERSION}  (from Step 4c verify)
- Setup date: {today}
- MCP registered: yes (user scope)
- Token: stored in ~/.claude.json (do not commit; never written to CLAUDE.md)
- Artifacts repo: {gstack_artifacts_remote URL or "none"}
- Artifacts sync: {off|artifacts-only|full}
- Current repo policy: {read-write|read-only|deny|unset}
```

承载令牌**永远不会**写入 CLAUDE.md(CLAUDE.md 在许多项目中检入 git)。承载令牌仅位于 `claude mcp add` 放置它的 `~/.claude.json` 中。

### Paths 1、2a、2b、3 (Local stdio)

```markdown
## GBrain Configuration (configured by /setup-gbrain)
- Mode: local-stdio
- Engine: {pglite|postgres}
- Config file: ~/.gbrain/config.json (mode 0600)
- Setup date: {today}
- MCP registered: {yes/no}
- Artifacts sync: {off|artifacts-only|full}
- Current repo policy: {read-write|read-only|deny|unset}
```

不要在这里写 `## GBrain Search Guidance` 块。`/sync-gbrain` 是其唯一的写入者:它在读取检查确认此仓库的代码索引答案后写入该块,并替换同一标题下的任何旧块。
Step 10 提供运行 `/sync-gbrain`。