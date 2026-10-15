<!-- AUTO-GENERATED from handoff.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## Step 18-H: 无 PR 的交接(自动化通道)

当运行不得打开 PR(`--no-pr`、自动化通道、无人值守父级说了)时,替换 pr-body 部分。这里没有任何内容发布。

1. 仅验证检查:像 pr-body 部分一样将 Step 16 的每通道通过/失败/跳过计数(命名需要远程的通道(`requires-remote`))写入 `.gstack/tmp/ship-gate.json`,然后:
   ```bash
   ~/.claude/skills/gstack/bin/gstack-ship-receipt write --base <base> --gate @.gstack/tmp/ship-gate.json
   ```
2. 按此顺序发送完成消息、标识符优先: `SHIP_RECEIPT:` 行和接收块(`pr: none`);`branch:` 和 `sha:` (`git rev-parse HEAD`),已推送或 `not pushed`;`inputs:` 计划或问题、基础 SHA 和支出上限;`before/after:` 每个声称的指标一行,在基础和此 SHA 上测量,使用使用的命令;`escalation:` 停止合并的一个条件(`requires-remote` 通道、打开闸门发现、未经验证的声明)或 `none`。
   任何跨线程引用中的完整线程 id(`jam_…`),永不是短代码。
   散文可以跟随标识符;没有任何东西可以优先于它们。
3. 跳过 Step 19 和 21。当 NO_VERSION 应用时使用 `"version":null` 运行 Step 20。