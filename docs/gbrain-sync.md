# 跨机器记忆与 GBrain 同步

gstack 将大量有用的状态写入 `~/.gstack/` — learnings、retros、CEO plans、design docs、developer profile。默认情况下,当你切换笔记本电脑时,所有这些都会丢失。**GBrain sync** 将精选子集推送到私有 git 仓库,以便你的记忆跨机器跟随你,并可被 GBrain 索引。

## 你得到什么

- 在机器 A 上工作,在机器 B 上无缝接续。
- 你的 learnings、plans 和 designs 在 GBrain 中可见(如果你使用它)。
- 一个干净的退出路径(`gstack-brain-uninstall`),永远不触及你的数据。
- 没有守护进程,没有系统服务,没有后台进程。

## 什么不会离开你的机器

默认情况下,即使同步开启,以下内容仍保留在本地:

- 凭据:`.auth.json`、`auth-token.json`、`sidebar-sessions/`、`security/device-salt`
- 机器特定状态:Chromium profiles、ONNX model weights、caches、eval-cache、CDP-profile、一次性 prompt 标记(`.welcome-seen`、`.telemetry-prompted`、`.vendoring-warned-*` 等)
- 问题偏好:每台机器的 UX 偏好(`question-preferences.json`、`question-log.jsonl`、`question-events.jsonl`)

确切的允许列表位于 `~/.gstack/.brain-allowlist`。CLI 管理它;你可以在标记行下面追加自己的条目。

## 首次运行设置(30–90 秒)

```bash
gstack-artifacts-init
```

该命令:

1. 将 `~/.gstack/` 变成 git 仓库。
2. 询问远程 URL(默认:`gh repo create --private gstack-artifacts-$USER`)。任何 git 远程都可以 — GitHub、GitLab、Gitea、自托管。
3. 仅推送仅包含配置的初始提交。
4. 写入 `~/.gstack-artifacts-remote.txt`(仅 URL,没有秘密 — 可以安全地复制到另一台机器)。
5. 打印 brain 主机的 `gbrain sources add` 连接命令(永远不要自动执行 — 自己运行,或者在你自己的机器上 `bin/gstack-gbrain-source-wireup` 执行相同的连接),以便 `gbrain search` 可以索引你同步的 learnings、plans 和 designs。旧的 `gstack-brain-reader add --ingest-url ...` HTTP 路径已在 v1.15.1.0 中删除 — 它依赖于 gbrain 从未发布的 `/ingest-repo` 端点。

初始化后,**你运行的下一个 skill** 会就隐私模式问你一个问题:

- **全部允许列表(推荐)**:learnings、reviews、plans、designs、retros、timelines 和 developer profile 全部同步。
- **仅工件**:plans、designs、retros、learnings — 跳过行为数据(timelines、developer profile)。
- **拒绝**:将所有内容保留在本地。你可以稍后使用 `gstack-config set artifacts_sync_mode full` 打开同步。

你的答案被持久化。不会再问你。

## 跨机器工作流

在机器 A 上:运行 `gstack-artifacts-init` 一次。就这样 — 每个 skill 调用现在在其开始和结束边界排空同步队列(每次 skill 约 200–800 毫秒网络暂停)。

在机器 B 上:

1. 将 `~/.gstack-artifacts-remote.txt` 从机器 A 复制到机器 B(密码管理器、dotfile 仓库、U 盘 — 由你决定;传统的 `~/.gstack-brain-remote.txt` 名称仍然被识别)。
2. 运行任何 gstack skill。preamble 看到 URL 文件并打印:
   ```
   BRAIN_SYNC: brain repo detected: <url>
   BRAIN_SYNC: run 'gstack-brain-restore' to pull your cross-machine memory
   ```
3. 运行 `gstack-brain-restore`。它克隆仓库,恢复你的 learnings/plans/retros,并重新注册 git merge 驱动程序。
4. 下一次 skill:你在机器 A 上的昨天学习浮出水面。这就是神奇的时刻。

## 状态、健康和队列深度

```bash
gstack-brain-sync --status
```

显示:状态(`ok`、`idle`、`held`、`blocked`、`push_failed`、`error`)、被秘密扫描保留的文件及其修复、可排空计数(`drainable`)、上次同步和推送时间、待处理队列深度以及当前隐私模式。

每个 skill 运行在 preamble 输出顶部附近打印 `ARTIFACTS_SYNC:` 行。当同步卡住时,它会添加 `ARTIFACTS_SYNC: attention:` 行;这些行及其修复的表格位于 [gbrain-sync-errors.md](gbrain-sync-errors.md#artifacts_sync-attention--at-skill-start)。

## 详细的隐私模式

| 模式 | 同步的内容 |
|------|------------|
| `off` | 无内容(默认)。 |
| `artifacts-only` | Plans、designs、retros、learnings、reviews。跳过 timelines + developer-profile。 |
| `full` | 允许列表中的所有内容,包括行为状态。 |

随时使用以下命令更改:
```bash
gstack-config set artifacts_sync_mode full
gstack-config set artifacts_sync_mode off
```

## 删除和移动的文件

默认情况下,同步从不发布删除操作:删除同步的设计文档,或移动项目目录(项目 slug 迁移这样做),旧路径保留在远程,在那里 brain 继续将其与新副本一起索引。要也发布删除操作:

```bash
gstack-config set artifacts_sync_removals on
```

然后每次同步都会提交对磁盘上消失的同步文件的删除。脚本页面保留原样,决策日志及其活动快照仅一起进行。一次性删除超过 20 个文件的同步不会发布其中任何一个:这看起来像是丢失,而不是清理。无论设置如何,都要自己检查并发布这些文件:

```bash
gstack-brain-sync --publish-removals        # list them
gstack-brain-sync --publish-removals --yes  # commit and push their removal
```

## 秘密保护

每次提交在离开你的机器之前都会扫描凭据形状的内容。被阻止的模式包括:

- AWS 访问密钥(`AKIA…`)
- GitHub tokens(`ghp_`、`gho_`、`ghu_`、`ghs_`、`ghr_`、`github_pat_`)
- OpenAI keys(`sk-…`)
- PEM 块(`-----BEGIN …-----`)
- JWTs(`eyJ…`)
- JSON 中的 Bearer tokens(`"authorization": "..."`、`"api_key": "..."` 等)

如果扫描命中,则只有标记的文件被保留(以及与之耦合的任何文件);其他所有内容都会同步。保留的文件保持排队状态,并在每次同步时重新扫描,skill start 打印:

```
ARTIFACTS_SYNC: attention: status=held held=1. The secret scan is holding back 1 file(s); everything else still syncs. See which files and how to fix them: <bin> --status
```

要修复:

1. 运行 `gstack-brain-sync --status`(该行打印其绝对路径)。`held` 下的每个条目都命名文件、扫描器规则和两种修复方法。
2. 如果匹配是你永远不想同步的误报,请运行 `gstack-brain-sync --skip-file <path>` 以永久排除该路径;`gstack-brain-sync --unskip-file <path>` 撤消它。
3. 否则,编辑该文件以删除秘密;下次 skill 运行会同步它。

`~/.gstack/.git/hooks/pre-commit` 处有一个防御性 hook,如果你手动对仓库运行 `git commit`,它会运行相同的扫描。

另外(v1.63.0.0+),每次推送在发送任何内容**之前**将防篡改收据写入出口账本(`~/.gstack/security/egress.jsonl`),失败关闭:如果无法写入收据,则推送被拒绝,队列被保留。使用 `gstack-egress list` 检查账本,并使用 `gstack-egress verify` 验证其哈希链。

## 双机器冲突

如果你在机器 A 和机器 B 同一天写入,两者都将推送追加提交。Git 的默认会在文件尾部冲突,但 `.jsonl` 和 markdown 文件已注册了自定义合并驱动程序:

- JSONL 文件使用 sort-and-dedup 驱动程序,按 ISO 时间戳对追加进行排序(每行的 SHA-256 哈希用于确定性)。
- Markdown 工件(retros、plans、designs)使用 union 合并驱动程序连接两侧。

你不应该看到冲突提示。如果你看到了(真正的语义冲突,例如两台机器编辑同一个计划),git 将停止并提示。

## 跨机器拉取节奏

Preamble 每 24 小时运行一次 `git fetch` + `git merge --ff-only`(通过 `~/.gstack/.brain-last-pull` 缓存)。你不需要考虑这个 — 它在每天第一次 skill 调用时自动发生。

历史说明 (#2516):该每日拉取仅刷新 `~/.gstack` 本身 — 而不是 gbrain 实际索引的 `~/.gstack-brain-worktree` 中的分离 worktree,因此 brain 在下次 setup-gbrain/sync-gbrain 运行之前默默地提供过时的页面。由于此修复,每日同步还会推进 brain worktree(`gstack-gbrain-source-wireup --advance-only`,通过 `~/.gstack/.brain-worktree-last-advance` 进行节流);失败的推进会发出警告而不是静默失败,并且永远不会强制重置脏 worktree。

## 卸载

```bash
gstack-brain-uninstall
```

这:

- 删除 `~/.gstack/.git/` 和所有 `.brain-*` 配置文件。
- 清除 `gstack-config` 中的 `artifacts_sync_mode`。
- 不会触及你的 learnings、plans、retros 或 developer profile。

添加 `--delete-remote` 也删除私有 GitHub 仓库(仅 GitHub,使用 `gh repo delete`)。

随时使用 `gstack-artifacts-init` 重新初始化。

## 故障排除

有关 gbrain-brain 可能打印的每条错误消息的索引,以及每个的问题、原因和修复,请参见 [gbrain-sync-errors.md](gbrain-sync-errors.md)。

## 幕后

此功能背后的架构决策:允许列表优于拒绝列表(未知文件默认保留在本地)、preamble-boundary 同步优于守护进程(没有要照看的后台进程)、JSONL 合并驱动程序以便并发机器联合其队列而不是冲突,以及隐私停止门在同步之前询问一次。
