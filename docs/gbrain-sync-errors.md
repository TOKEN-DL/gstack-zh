# gbrain-sync 错误查找

`gstack-brain-*` 可以打印的每条错误消息,以及问题、原因和修复。

通过 `BRAIN_SYNC:` 后的前缀或命令输出中的二进制名称搜索此文件。

---

## `BRAIN_SYNC: brain repo detected: <url>`

**问题。** 你在一台具有 `~/.gstack-artifacts-remote.txt`(或传统的 `~/.gstack-brain-remote.txt`,从另一台机器复制)的机器上,但 `~/.gstack/.git` 处没有本地 git 仓库。

**原因。** 你在其他地方设置了 GBrain 同步,而你的 gstack 尚未在此机器上恢复。

**修复。**
```bash
gstack-brain-restore
```
这将仓库拉到 `~/.gstack/` 并重新注册合并驱动程序。

如果你不想在此处恢复,请使用以下命令消除提示:
```bash
gstack-config set artifacts_sync_mode_prompted true
```

---

## `ARTIFACTS_SYNC: attention: ...` 在 skill start 处

**问题。** 工件同步卡住或部分卡住。Skill start 为每个问题打印一行固定;`ok` 和 `idle` 不打印任何内容。每行都命名 `gstack-brain-sync` 的绝对路径;使用 `--status` 运行它以查看原因、保留的文件和修复。

| 状态 | 触发器 | 注意行(在 `ARTIFACTS_SYNC: attention: ` 之后) | 修复 |
|---|---|---|---|
| `held` | 秘密扫描保留了标记的文件并同步了其余文件 | `status=held held=<N>. The secret scan is holding back <N> file(s); everything else still syncs. See which files and how to fix them: <bin> --status` | 编辑文件以使扫描不再匹配,或 `<bin> --skip-file <path>`(永久;使用 `--unskip-file <path>` 撤消) |
| `blocked` | 重新扫描后仍被标记,因此未提交任何内容 | `status=blocked. The secret scan flagged a file it could not hold back on its own, so nothing is syncing. See the fix: <bin> --status` | 编辑或跳过 `--status` 命名的文件,或使用 `<bin> --drop-queue --yes` 清除批次 |
| `push_failed` | 提交在本地保存但推送失败 | `status=push_failed. Synced files are committed locally but the push failed; it retries at skill start. See the cause: <bin> --status` | `--status` 命名的修复(通常是 `gh auth status`) |
| `error` | 排空在提交前停止并保留队列 | `status=error. The last sync stopped before committing; the queue is kept and retried. See the cause: <bin> --status` | `--status` 命名的原因:git 索引锁、提交 hook、`git add` 失败 |
| `unknown` | 状态文件持有此表之外的状态代码 | `status=unknown. The sync status file is unreadable. See it: <bin> --status` | 运行 `<bin> --once` 重写它 |
| stale-push | 当排空的 `drainable` 计数大于零时,24 小时内没有推送 | `stale-push. Files are ready to sync but nothing has been pushed for over 24 hours. See why: <bin> --status` | `--status` 命名的等待原因(等待一致代的耦合组、`git add` 失败) |
| stale-drain | 当队列持有记录时,`last_drain_at` 超过 24 小时 | `stale-drain. The sync has not run for over 24 hours while files wait in the queue. See why: <bin> --status` | 如果没有运行 gstack 进程,请删除 `~/.gstack/.brain-sync.lock.d`,然后运行 `<bin> --once` |

`<bin>` 是绝对路径,例如 `~/.claude/skills/gstack/bin/gstack-brain-sync`。事实来源是 `bin/gstack-brain-sync` 的使用头;skill start 的测试是从中生成的。

**保留的文件。** 当秘密扫描标记一个文件时,只有该文件被保留(以及与之耦合的任何文件,例如决策日志及其活动快照);其他所有内容都会同步。保留的文件保持排队状态,并在每次同步时重新扫描,因此编辑它会自动清除保留。`--status` 在 `held` 下列出每个保留的文件,以及匹配的扫描器规则(永远不是匹配的文本)、与之一起保留的文件(`dependents`)和两种修复方法:

1. **如果它是真正的秘密**:编辑文件以将其删除。下次 skill 运行会重新扫描并同步它。
2. **如果匹配是你永远不想同步的误报**:
   ```bash
   ~/.claude/skills/gstack/bin/gstack-brain-sync --skip-file <path>
   ```
   这将永久从将来的同步中排除该路径(对于耦合文件,与它一起的文件也会停止)。要撤消它并将路径重新排队:
   ```bash
   ~/.claude/skills/gstack/bin/gstack-brain-sync --unskip-file <path>
   ```
3. **要放弃整个批次**(重新开始):
   ```bash
   ~/.claude/skills/gstack/bin/gstack-brain-sync --drop-queue --yes
   ```

**Git 索引锁。** 在写入中途杀死的 git 进程会留下 `~/.gstack/.git/index.lock`。同步在它超过 10 分钟时将其删除,因为对 `~/.gstack` 的每个 gstack 写入器首先采用相同的同步锁。较年轻的锁意味着 git 进程正在那里运行;同步报告 `error` 并在下次 skill start 时重试。

---

## `BRAIN_SYNC: push failed: auth.`

**问题。** Git 推送被拒绝,因为你的远程身份验证已过期或缺失。

**原因。** 使用当前凭据无法访问远程。

**修复。** 根据你的远程刷新身份验证:

- **GitHub**:`gh auth status`(然后根据需要 `gh auth refresh`)
- **GitLab**:`glab auth status`
- **其他**:`git remote -v` + 检查 SSH 密钥或凭据帮助器

修复身份验证后,运行任何 skill 以自动重试同步。

---

## `BRAIN_SYNC: push failed: <first-line-of-error>`

**问题。** 由于身份验证以外的原因推送失败。Git 错误的第一行出现在冒号后面。

**原因。** 可能是网络问题、拒绝的推送(远程提前)、服务器 500 或撤销了仓库访问。

**修复。** 有关更多详细信息,请查看 `~/.gstack/.brain-sync-status.json`,或运行:
```bash
cd ~/.gstack && git status && git push origin HEAD
```
以查看 git 的完整错误。任何推送尝试后队列都会被清除,但你的本地提交仍然存在 — 下次 skill 运行将重试推送。

---

## `gstack: brain-sync push NOT sent — the egress receipt could not be written`

**问题。** 推送在任何内容离开你的机器之前被拒绝。每次 brain-sync 推送在发送之前将防篡改收据写入出口账本(`~/.gstack/security/egress.jsonl`),失败关闭。无法写入收据,因此未发送任何内容,未进行本地提交,并保留队列 — 下次运行重试整个排空。`gstack-brain-sync --status` 显示 `EGRESS_RECEIPT_FAILED` 作为失败详细信息。

**原因。** `~/.gstack/security/` 不可写(收据写入器在缺失时创建它,因此仅凭缺失不是原因)、磁盘已满,或 `GSTACK_HOME` 指向只读位置。

**修复。**
```bash
mkdir -p ~/.gstack/security && chmod -R u+w ~/.gstack/security
```
然后运行任何 skill(或 `gstack-brain-sync --once`)以重试。使用 `gstack-egress list` 检查账本;使用 `gstack-egress verify` 验证其哈希链。

---

## `gstack-artifacts-init: ~/.gstack/ is already a git repo pointing at: <url>`

**问题。** 你尝试使用与现有远程不匹配的远程 URL 进行初始化。该命令拒绝覆盖。

**原因。** 你已经使用不同的远程运行了 `gstack-artifacts-init`。

**修复。** 任选其一:

- 使用现有远程:运行 `gstack-artifacts-init` 而不带 `--remote`,或使用匹配的 URL。
- 切换远程:`git -C ~/.gstack remote set-url origin <url>`(命令自己的建议),或先 `gstack-brain-uninstall`,然后使用新 URL 重新初始化。两者都不会删除你的数据。

---

## `Remote not reachable via SSH: <url>`

**问题。** Init 无法访问 git 远程以验证连接。

**原因。** URL 错误、缺少身份验证、网络问题。

**修复。** 手动测试:
```bash
git ls-remote <url>
```
如果失败,请检查:
- URL 拼写
- GitHub:`gh auth status`
- GitLab:`glab auth status`
- 专用网络 / VPN / DNS

---

## `Failed to create or find '<name>'. Try --remote <url>.`

**问题。** 通过 `gh repo create` 自动创建仓库失败,并且该仓库也无法通过 `gh repo view` 发现。

**原因。** `gh` 未通过身份验证、该名称的仓库已存在并由其他人拥有,或你的 GitHub 账户已达到配额。

**修复。**
```bash
gh auth status
```
如果未通过身份验证,请运行 `gh auth login`。如果仓库名称冲突,请传递不同的名称:
```bash
gstack-artifacts-init --remote git@github.com:YOURUSER/custom-name.git
```

---

## `gstack-brain-restore: ~/.gstack/.git already points at <url>`

**问题。** 你尝试从不与现有 git 配置匹配的 URL 进行恢复。

**原因。** 来自使用不同远程的先前初始化的过时 `.git`。

**修复。** `gstack-brain-uninstall`,然后使用 `<url>` 重新运行 `gstack-brain-restore`。

---

## `gstack-brain-restore: ~/.gstack/ has existing allowlisted files that would be clobbered`

**问题。** 你正在尝试恢复,但 `~/.gstack/` 已包含将被覆盖的 learnings 或 plans。

**原因。** (a) 此机器从同步前的 gstack 会话累积了状态,或 (b) 先前失败的恢复留下了部分状态。

**修复(三选一)。**

1. **如果此机器的状态应该成为新的事实来源**:运行 `gstack-artifacts-init` 而不是 restore — 这将从该机器的状态创建一个全新的 brain 仓库。
2. **如果你想采用远程并丢弃此机器的状态**:首先备份 `~/.gstack/projects/`,然后删除违规文件并重新运行 restore。
3. **如果要进行合并**:没有自动合并。在已启用同步的机器上的运行 gstack 中手动将 learnings 从 `~/.gstack/` 复制到正在运行的 gstack,然后在此处恢复。

---

## `gstack-brain-restore: <url> does not look like a gstack-brain repo`

**问题。** 克隆成功,但仓库缺少 `.brain-allowlist` 和 `.gitattributes`。

**原因。** 你将 restore 指向了随机 git 仓库,或者有人从 brain 仓库中删除了规范配置文件。

**修复。** 验证 URL。如果正确,请运行 `gstack-artifacts-init --remote <url>` 以重新植入规范配置。

---

## `database host unreachable (<code> <host>); your gbrain config is unchanged. Fix: check network or VPN, then re-run /sync-gbrain.`

**问题。** `/sync-gbrain` 跳过代码、内存和梦想阶段,或 `/setup-gbrain` 报告 `gbrain_local_status: db-unreachable`。

**原因。** 由于网络错误(`ENOTFOUND` / `EAI_AGAIN` 在离线沙箱中、关闭的 VPN、被拒绝或超时的连接),gbrain 无法访问 `~/.gbrain/config.json` 中命名的数据库。配置没有已知的问题,因此 gstack 保留它并保持 brain-aware skill 块的渲染,与对待慢 `timeout` 的方式相同。

**修复。** 恢复网络访问(或 VPN),然后:
```bash
/sync-gbrain
```
不要将 `~/.gbrain/config.json` 移开;该修复仅适用于 `broken-db` / `broken-config`。

---

## `[memory-ingest] ERR: gbrain import accounted for N-1 of N staged page(s) ... Refusing to advance state` on every run (a `.claude`-style project)

**问题。** 在此版本之前,如果暂存的页面来自项目 slug 以点开头的项目(例如 `.claude`),则每次运行中导入都会拒绝每个批次。

**原因。** gbrain 的导入 walker 跳过以点开头的每个路径段,因此暂存的页面永远不会被收集。

**修复。** 升级后无需执行任何操作:此类 slug 现在暂存为 `dot-claude`,并且旧名称下记录的状态映射到新名称。运行 `/sync-gbrain` 一次以导入卡住的页面。

---

## `[memory-ingest] FAILED <path>: <error> (left un-stamped; retried next run)`

**问题。** gbrain 在 `/sync-gbrain` 的内存阶段拒绝了一个暂存的页面(例如无效的 frontmatter 或过大的页面)。

**原因。** `gbrain import` 将页面命名为每文件失败(在其 `--json` `failures` 列表或 `Skipped <path>:` 行中)。在此版本之前,此类页面被标记为已导入并且永远不会重试。

**修复。** 没有丢失:页面未标记为已导入,并在每次运行时重试。如果同一页面继续失败,请阅读错误,修复其源文件,然后运行 `/sync-gbrain`。

---

## `[memory-ingest] re-queued <slug>: not found in gbrain source <id> after import (retried next run)`

**问题。** gbrain 说已导入(或未更改)页面,但着陆检查无法在导入到的源中找到它,或找到不同的内容。

**原因。** 通常是被接受但仍在发布的托管导入(gbrain 输出中的 `Pending:`),或 gbrain 未命名的跳过页面。

**修复。** 无需执行任何操作;下次 `/sync-gbrain` 会再次导入。如果每次运行都重复,请升级 gbrain(`gstack-gbrain-install`)。

---

## `[memory-ingest] ERR: gbrain reported N failure(s) it did not attribute to a staged page ... Refusing to advance state.`

**问题。** 内存阶段失败且未标记任何已导入。

**原因。** gbrain 计算了它未命名的失败(它在相同类型的五个之后停止打印它们),因此 gstack 无法判断哪些页面已着陆。

**修复。** 重新运行 `/sync-gbrain`。三次拒绝运行后,批次自动一分为二以查找 gbrain 无法导入的页面;该页面被隔离(下一项),其余被导入。

---

## `[memory-ingest] quarantined <slug> (<source path>): gbrain refuses every batch that contains it (...)`

**问题。** 一个页面持续导致 gbrain 拒绝整个批次。

**原因。** 在三次拒绝批次后,gstack 将批次一分为二地导入,直到隔离出该页面。所有其他页面都已导入。

**修复。** 当其源文件更改时,会自动重试该页面。要立即重试,请修复或删除消息中命名的源文件,然后运行 `/sync-gbrain`。

---

## `[memory-ingest] reconcile: checked N, present M, re-queued K, not yet checked L (run again to continue)`

**问题。** 不是错误。这是升级后的一次性追赶:gstack 重新检查它已标记为已导入的脚本,并重新排队那些从 brain 中丢失的脚本,以便旧版本丢失的脚本再次导入。

**原因。** 旧版本可以标记 gbrain 已跳过的页面为已导入。升级记录需要协调;每次导入运行然后检查有界批次(200 个条目)直到完成。

**修复。** 无需执行任何操作;每次 `/sync-gbrain` 从上次停止的地方继续。要立即运行它或检查进度:
```bash
gstack-memory-ingest --reconcile --dry-run   # report only
gstack-memory-ingest --reconcile             # re-check the next 200 (--limit N)
```
"not yet checked" 计算查找失败的页面(gbrain 忙或无法访问);它们永远不会被视为缺失。"unrecoverable" 计算从 brain 中缺失且其脚本文件不再存在的页面。状态文件在每次传递之前备份到 `~/.gstack/.transcript-ingest-state.json.pre-reconcile.bak`。

---

## `[memory-ingest] reconcile: not run (...)`

**问题。** 已请求协调传递但无法运行。

**原因。** gbrain 未安装或缺少 `import`,或者 brain 是远程 HTTP brain(其页面是为 brain 管理员的拉取而暂存的,因此本地没有要检查的内容)。

**修复。** 对于缺少的 gbrain,请运行 `/setup-gbrain`,然后 `gstack-memory-ingest --reconcile`。对于远程 HTTP brain,无需执行任何操作。

---

## `[memory-ingest] ERR: another memory ingest (pid <N>) is writing <state file>; not run.`

**问题。** 内存阶段本次运行未执行任何操作。

**原因。** 另一个导入、协调或 `--request-reconcile` 持有状态锁(`~/.gstack/.transcript-ingest-state.json.lock`)。由不再存在的进程留下的锁会自动接管。

**修复。** 等待其他运行完成,然后再次运行 `/sync-gbrain`。

---

## `[memory-ingest] ERR: could not save ingest state <path>: <error>.`

**问题。** 导入后内存阶段失败。

**原因。** 状态文件无法写入(磁盘已满、权限,或其中有目录)。运行失败而不是假装它记录了进度;它导入的页面将在下次运行时重新检查,这很便宜,因为 gbrain 跳过未更改的内容。

**修复。** 释放磁盘空间或修复 `~/.gstack/` 的权限,然后运行 `/sync-gbrain`。

---

## `[memory-ingest] kept N unattributed transcript(s) on this machine: the brain is remote (Postgres or HTTP), ...`

**问题。** `--include-unattributed` 脚本(没有 git 远程的会话)未被导入。

**原因。** 无法归属于仓库的脚本仅转到机器本地的、永远不会被联合的源 `gstack-transcripts-unattributed`。你的 brain 是远程的(一个 Postgres 或 Supabase `database_url`,或远程 HTTP MCP brain),因此发送它们会将它们从这台机器上移走。它们未标记为已导入,也不会写入可发布的 `~/.gstack/transcripts/` 暂存。

**修复。** 如果这是你想要的,则无需执行任何操作。要包括此类会话,请从具有 `origin` 远程的 git 仓库运行它,以便它被归因。

---

## `[memory-ingest] kept N transcript page(s) for <repo> on this machine: could not register gbrain source <id> (...)`

**问题。** 来自一个仓库的脚本未被导入。

**原因。** gstack 将每个仓库的脚本导入到该仓库自己的 gbrain 源(`gstack-transcripts-...`,机器本地且不联合)中,并在第一次导入之前注册它。已安装的 gbrain 拒绝 `gbrain sources add`,因此页面保留在本地且未标记。

**修复。**
```bash
gstack-gbrain-install      # upgrade gbrain
/sync-gbrain               # the pages import on the next run
```

---

## `curated artifacts pushed to git, but gbrain source <id> has 0 indexed pages. Fix: gbrain sync --source <id>, then re-run /sync-gbrain`

**问题。** 即使 git 推送有效,`/sync-gbrain` 的 brain-sync 阶段也会出现 `ERR`。

**原因。** 推送仅将你精选的工件移动到 git。gbrain 单独索引它们,并且此机器维护的工件源报告零页面,因此搜索不会从它返回任何内容。在此版本之前,该阶段表示"精选工件已推送"并通过。

**修复。**
```bash
gbrain sync --source <id>
/sync-gbrain
```
如果它保持在零,请使用 `gstack-gbrain-source-wireup` 重新注册源。

---

## `[gbrain-sync] gbrain source <id>: path unavailable (<path>); gstack skips it.`

**问题。** gbrain 代码源指向此机器上不再存在的目录,通常是已删除的 Conductor worktree。gbrain 的自动驾驶不断失败。

**原因。** 每个 worktree 都有自己的 `gstack-code-<repo>-<hash>` 源。删除 worktree 会留下源。gstack 不会自行删除它:其索引页面可能是从未提交的工作唯一副本,并且源可能属于共享 brain 的另一台机器。

**修复。** 检查,然后删除 gstack 可以证明已消失的源:
```bash
gstack-gbrain-sync --prune-gone-worktrees --dry-run
gstack-gbrain-sync --prune-gone-worktrees
```
仅当其路径在两次连续同步中丢失、其父目录可读、其 id 从此主机和路径重新计算(因此此机器创建了它)并且其仓库仍然存在但不再列出 worktree 时,才会删除源。每次删除都会记录到 `~/.gstack/.gbrain-prune.log`。从同一仓库的主 checkout 运行它,以便 gstack 可以找到仓库。它保留的任何内容都会打印原因;只有在确定时才使用 `gbrain sources remove <id>` 手动删除此类源。

---

## `dream: skipped — the installed gbrain cannot run only the resolve_symbol_edges phase, and the full dream cycle costs about 35 minutes (LLM phases).`

**问题。** `/sync-gbrain --dream`(或 `--full`)未构建调用图。

**原因。** 调用图只需要 gbrain 的 `resolve_symbol_edges` 阶段。已安装的 gbrain 无法单独运行该阶段(没有 `dream --phase`,或没有此类阶段),并且 gstack 不再自行启动 gbrain 的完整维护周期:它需要约 35 分钟并运行 LLM 阶段。

**修复。**
```bash
gstack-gbrain-install          # upgrade gbrain, then:
```

[desktop write output truncated: original_bytes=27284; if the omitted remainder is needed, continue with read using offset=493 and the same path.]
