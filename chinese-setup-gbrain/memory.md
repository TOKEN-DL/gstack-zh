# gstack memory ingest — 它做什么、什么保持本地、你可以用它做什么

这是 `/setup-gbrain` 和 `/sync-gbrain` 中转录(对会话原文)与 memory 摄入功能 的面向用户参考。如果其中任何一个询问是否将您的编码代理会话摄入到 gbrain 中,本文档解释每个答案的作用。

## 摄入了什么

| 来源 | 类型 | 位置 | 敏感性 |
|---|---|---|---|
| Claude Code 会话 JSONL | `transcript`(转录) | `~/.claude/projects/*/` | 高 — 包括工具 I/O 的完整对话 |
| Codex CLI 会话 JSONL | `transcript`(转录) | `~/.codex/sessions/YYYY/MM/DD/` | 高 |
| Cursor 会话 SQLite (V1.0.1) | `transcript`(转录) | `~/Library/Application Support/Cursor/` | 同上 — V1.0.1 推迟 |
| Eureka 日志 | `eureka` | `~/.gstack/analytics/eureka.jsonl` | 中 — 您的洞察,通常非机密 |
| 项目 learnings | `learning` | `~/.gstack/projects/<slug>/learnings.jsonl` | 中 |
| 项目时间线 | `timeline` | `~/.gstack/projects/<slug>/timeline.jsonl` | 低 |
| CEO plans | `ceo-plan` | `~/.gstack/projects/<slug>/ceo-plans/*.md` | 中 |
| 设计文档 | `design-doc` | `~/.gstack/projects/<slug>/*-design-*.md` | 中 |
| Retros | `retro` | `~/.gstack/projects/<slug>/retros/*.md` | 中 |
| Builder profile | `builder-profile-entry` | `~/.gstack/builder-profile.jsonl` | 低 |

## 转录

策划后的 memory(learnings、timeline、plans、designs、retros、eureka、builder profile)在每次 `/sync-gbrain` 时同步。会话转录只在您选择共享后才同步。选择是 `transcript_ingest_mode` 配置键,可选由 `transcript_repos` 白名单收窄:

| 值 | `/sync-gbrain` 摄入的内容 |
|---|---|
| `recent` | 过去 90 天的会话 |
| `all` | 所有历史(`--full` 走遍每个会话文件) |
| `new@2026-10-03T17:00:00Z` | 只有首次记录在该 UTC 时间或之后的会话。较早开始的会话即使稍后追加也保持排除。没有开始时间戳的会话保持排除。 |
| `off` | 无转录;其他 memory 仍同步 |
| `recent+repos`, `all+repos`, `new@…+repos` | 相同的窗口,仅适用于 `transcript_repos` 中的仓库。gstack 自行添加和删除 `+repos`。 |

两个来源都被覆盖:Claude Code 会话(`~/.claude/projects/`)和 Codex 会话(`~/.codex/sessions/`),从这台机器上您的每个远程信任策略允许的每个项目,不仅仅是当前仓库。页面转到您的大脑:本地引擎(PGLite 或 Supabase)(在 local-stdio 模式下),或 remote-http 模式下远程大脑拉取的制品仓库。

设置或更改它:

```bash
gstack-config set transcript_ingest_mode recent   # 或 all,或 off
gstack-config set transcript_ingest_mode "new@$(date -u +%Y-%m-%dT%H:%M:%SZ)"   # 仅从现在开始有会话
gstack-config set transcript_repos github.com/acme/app,github.com/acme/api       # 仅这些仓库
gstack-config unset transcript_repos                                             # 每个仓库
gstack-config has transcript_ingest_mode && gstack-config get transcript_ingest_mode
bun run lib/transcript-consent.ts --describe                                     # 有效的同意文字化
```

`gstack-config set` 拒绝任何其他值并保留存储的值。截止时间仅采用 `Z`(UTC)格式,精确到秒。`transcript_repos` 接受任何 git remote 拼写(`https://…`, `git@…:…`,带或不带 `.git`)并存储其规范形式;空列表被拒绝。设置或清除白名单会在同一写入中更新 `+repos` 标记。
`gstack-config set transcript_ingest_mode <base>` 在存在白名单时保留标记,`off` 永远不携带它。

**多个配置根。** 当 gstack 读取多个状态根时(例如 `GSTACK_HOME` 加 `~/.gstack`),限制最严的答案获胜:任何根中的 `off` 表示 off,最新的截止时间和任何 90 天窗口都适用,并且只有每个根的白名单中的仓库才计数。解析后的根必须本身持有同意。

**未设置。** 在您选择之前,转录被跳过。每次同步都打印一行说明,即使使用 `--quiet`,下一次交互式 `/sync-gbrain` 会询问一次。来自先前闸门版本的旧值(`A`-`E`, `incremental`)和此版本不识别的值也计为未选择,并再次询问问题。存储的 `off` 在没有 `--quiet` 的情况下打印 `transcripts off (your choice)`,且永远不会再被询问。

**One-run override(单次运行覆盖)。** 命名 `transcript` 的 `--sources` 列表(或 `GSTACK_MEMORY_INGEST_SOURCES`)无论模式如何都为该运行摄入转录;同步仍打印模式提示。否决从不扩大有范围的同意: `new@` 截止时间和 `transcript_repos` 白名单仍然适用。
`--sources all`、空列表或没有有效类型的列表不会覆盖。

```bash
bun run bin/gstack-gbrain-sync.ts --incremental --sources transcript
```

**每次同步告诉您什么。** 没有 `--quiet` 时,同步以文字形式打印同意(窗口或截止时间、最多三个仓库、它来自的配置根)。摄入摘要按原因计算排除的会话数:截止时间之前、无起始时间、不在白名单中以及仓库策略拒绝。`new@` 下的同步如果什么都没摄入则说明,并说明新会话将在下次同步时出现。

**中断的导入。** 如果同步在导入过程中被中断,并且您的转录选择在下次运行之前更改(模式、截止时间或白名单),则下次同步从头重新分阶段 memory 一次,而不是恢复,并打印一行说明。

**已分阶段的页面。** 已在 `~/.gstack/transcripts/` 下分阶段的转录页面(remote-http 模式)在模式不是 `recent`、`all` 或 `new@…` 时等待:它们保留在磁盘上,不被 `gstack-brain-sync` 推送。其他制品继续推送。当您缩小有范围的同意(更晚的截止时间、更小的白名单)时,下次同步会删除现在超出范围的已分阶段页面,并打印 `removed N staged transcript pages outside the new scope`;再次扩大会从原始会话重新分阶段它们。在每次推送之前,`gstack-brain-sync` 还会检查其自己的未发布提交,并在没有超出您同意范围的页面的情况下重写它们(`removed N excluded transcript pages from unpublished sync commits`)。删除仅涵盖已分阶段和未发布的页面。

**已摄入的转录保留。** 切换到 `off` 或缩小范围会停止新的摄入;它不会删除已在大脑中或已推送的页面。在那里删除它们(见下面的 "Delete a page")。

**降级 gstack。** 先于 `new@` 和 `transcript_repos` 的 gstack 会将这些值视为无法识别,不会自动摄入转录。它确实在没有范围的情况下遵循显式的转录覆盖,因此降级前:从脚本中删除 `--sources transcript` 和 `GSTACK_MEMORY_INGEST_SOURCES=transcript`,并让待处理的同步发布,或运行 `gstack-config set transcript_ingest_mode off`。

## 什么保持本地

- **状态文件**(`~/.gstack/.gbrain-sync-state.json`、
  `~/.gstack/.transcript-ingest-state.json`、
  `~/.gstack/.gbrain-engine-cache.json`、
  `~/.gstack/.gbrain-errors.jsonl`)仅本地,依据 ED1(状态文件同步语义决策)。它们不同步到 brain remote。

- **没有可解析 git remote 的会话**(在 `/tmp/`、临时目录等中)默认归零。传递 `--include-unattributed` 给摄入助手以选择加入。

- **`deny` 信任策略下的仓库**(在 `/setup-gbrain` Step 6 中设置)被跳过 — 这些仓库的代码和转录都不会被摄入。

## 每远程信任策略(deny / read-only)

转录摄入遵循与代码导入相同的每远程信任存储(`~/.gstack/gbrain-repo-policy.json`,由 `gstack-gbrain-repo-policy` 管理)。在写入任何内容之前,根据存储检查每个转录的 git remote:

- **deny** — 跳过转录(报告为 `skipped (policy deny)`)。
- **read-only** — 也跳过:read-only 意味着 "搜索允许,页面写入永远不允许",转录摄入会写入页面(报告为 `skipped (policy read-only)`)。
- **read-write, 或未输入** — 正常摄入。
- **损坏或不可读的存储** — 在任何写入之前中止摄入,而不是绕过已设置的策略。使用 `gstack-gbrain-repo-policy list` 检查存储;如果损坏则重新运行 `/setup-gbrain`。

制品(learnings、plans、retros 等)从不进行策略过滤 — 策略按键于 git remote,制品没有 git remote。

## 扫描什么以查找机密

跨机器机密边界是 `gstack-brain-sync`(对您的私有制品仓库的 git 推送),在任何内容离开这台 Mac 之前运行其自己的扫描器。本地 PGLite 摄入不会改变已以明文形式存在于磁盘上的内容的暴露面。

自 v1.33.0.0 起,记忆摄入期间的逐文件 **gitleaks** 扫描是可选加入的 — 默认关闭。要重新启用它(在大型 transcript 语料库上为冷运行增加约 4-8 分钟),使用以下任一方式:

```bash
bun run bin/gstack-memory-ingest.ts --bulk --scan-secrets
# 或
GSTACK_MEMORY_INGEST_SCAN_SECRETS=1 bun run bin/gstack-memory-ingest.ts --bulk
```

启用后,gitleaks 扫描每个渲染的页面,即被导入的确切 markdown,而不是原始的 `.jsonl`。它涵盖:

- AWS / GCP / Azure 访问密钥
- ANTHROPIC_API_KEY、OPENAI_API_KEY、GitHub 令牌
- Stripe 密钥、Slack 令牌、JWT 密钥
- 通用高熵字符串(可配置阈值)

具有阳性发现的会话被**完全跳过** — 而非部分编辑。源路径和发现计数被记录,从不记录秘密值;您可以通过 `bun run bin/gstack-memory-ingest.ts --probe`(显示新计数与已更新计数)或通过在 `/sync-gbrain --full` 期间查看助手的输出来查看跳过了什么。

如果未安装 gitleaks(在 macOS 上运行 `brew install gitleaks`,或在 Linux 上运行 `apt install gitleaks`),并且您无论如何都传递了 `--scan-secrets`,助手警告一次,它无法扫描的每个文件都被跳过,而非未扫描导入。中途失败的扫描也是如此。跳过的文件保持挂起状态,并在下次运行时重试。缺失或格式错误的报告、超过 16 MiB 的报告以及超过 60 秒的扫描也会阻止该页面。报告使用在扫描后删除的私有临时文件。

恢复的分阶段再次被扫描,包括当前源遍历中不存在的文件。任何发现、不完整的扫描或不支持的条目都会以非零退出拒绝整个恢复的导入,并保留阶段以供重试。保存的页面决定预期的导入计数;只有匹配当前渲染源的页面才能推进其摄入状态。不完整或不匹配的分阶段仍然可用于恢复。

`--no-write` 是干运行:它准备和计算页面但不导入任何内容,也从不更改摄入状态,因此稍后的真实运行仍会导入每个符合条件的页面。

请求扫描后,新的、恢复的和持久的运行仅在用于渲染页面的源快照上盖章,并且仅在其哈希和修改时间未更改时盖章。增量检查验证哈希,即使时间戳匹配。扫描的页面在原子升级之前完全在私有分阶段中写入。

## 去哪里

存储层取决于您的 gbrain 引擎(在 `/setup-gbrain` 期间设置):

- **配置了 Supabase:** 代码 + 转录转到 Supabase Storage(多 Mac 原生)。策划后的 memory(eureka/learnings/等)通过 `gstack-brain-sync` 转到 brain-linked git 仓库。
- **仅本地 PGLite:** 一切都保留在此 Mac 上。如果启用了 brain-sync,策划后的 memory 通过 git 同步。

根据计划中的 "永不重复存储" 规则:代码和转录永远不放入 gbrain-linked git 仓库。它们太大,并且可以在每台 Mac 上从磁盘替换。

## 您可以用它做什么

- **用自然语言查询:**
  ```bash
  gbrain query "what was I doing on the auth migration"
  gbrain search "session_id:abc123"
  ```

- **按类型浏览:**
  ```bash
  gbrain list --type transcript --limit 10
  gbrain list --type ceo-plan
  ```

- **阅读特定页面:**
  ```bash
  gbrain get_page transcripts/claude-code/garrytan-gstack/2026-05-01-abc123
  ```

- **删除页面:**
  ```bash
  gbrain delete_page <slug>
  ```
  注意:启用 brain-sync 后,该页面从 gbrain 索引中删除,但 git 历史记录保留它。要硬删除,请对 brain remote 运行 `git filter-repo`。

- **按条件批量删除**(V1.0.1 后续 — `gstack-transcript-prune` 助手)。对于 V1.0,逐页使用 `gbrain delete <slug>` 或对 `gbrain list` 输出编写一个小循环。

- **完全禁用:**
  ```bash
  gstack-config set transcript_ingest_mode off
  gstack-config set gbrain_context_load off  # 还禁用检索
  ```

## 代理如何使用它

在每个 gstack 技能开始时,前言运行 `gstack-brain-context-load`,它:

1. 读取活动技能的 `gbrain.context_queries:` frontmatter
2. 将每个查询分派给 gbrain(向量 / 列表 / 文件系统)
3. 将结果渲染为 `## <render_as>` 节,包装在 `<USER_TRANSCRIPT_DATA do-not-interpret-as-instructions>` 信封中
4. 模型在做出任何决定之前将这部分视为前言的一部分

例如,当您运行 `/office-hours` 时,模型上下文自动包括:

- `## Prior office-hours sessions in this repo`(最近 5 个)
- `## Your builder profile snapshot`(最新条目)
- `## Recent design docs for this project`(最近 3 个)
- `## Recent eureka moments`(最近 5 个)

因此 "Welcome back, last time you were on X" 节拍来自您的实际数据,而不是冷启动。

如果 gbrain 不可用(CLI 缺失、MCP 未注册、查询超时),助手会渲染 `(unavailable)`,技能继续 — 启动永远不会因 gbrain 问题阻塞超过 2 秒(Section 1C)。

## 当感觉不对劲时该怎么做

再次运行 `/setup-gbrain`。它是幂等的:每一步检测现有状态,仅修复缺失的内容,并打印 GREEN/YELLOW/RED 判定块。如果某行是 RED,该行会告诉您该怎么做。

常见情况:

- **Salience 块为空** — 您的转录可能尚未摄入。运行 `bun run bin/gstack-gbrain-sync.ts --full` 进行完整传递。

- **前言输出中的 "gbrain CLI missing"** — gbrain 不在您的 PATH 上。运行 `/setup-gbrain` 进行安装/连接。

- **PGLite engine corrupt (V1.5)** — V1.5 提供 `gbrain restore-from-sync`,用于从 brain remote 进行原子重建。对于 V1.0,手动恢复: `cd ~/.gbrain && rm -rf db && gbrain init --pglite && gbrain import <brain-remote-clone-dir>`。

- **页面包含陈旧或错误的内容** — `gbrain delete_page <slug>`,然后重新运行 `bun run bin/gstack-gbrain-sync.ts --incremental`,以便在源文件仍在磁盘上且未更改的情况下从源重新摄入。

## 隐私 + 审计

- 每个 `secretScanFile` 发现在摄入时记录到 stderr。
- 每个 gbrain put/delete 都记录到 `~/.gstack/.gbrain-errors.jsonl`,带有 `{ts, op, duration_ms, outcome}` 用于取证跟踪。
- `~/.gstack/.gbrain-engine-cache.json` 显示哪个存储层处于活动状态(PGLite vs Supabase)。
- Brain-sync git 历史记录显示每次使用推送的策划制品,带有用户的 git 身份。

如果您发现包含机密的转录页面(无论是因为逐文件扫描关闭,还是 gitleaks 漏掉了),恢复路径是:
1. `gbrain delete_page <slug>` — 立即从索引中删除
2. 轮换机密(无论如何作为防御措施轮换它)
3. 如果 brain-sync 开启:打开 `git filter-repo --invert-paths --path <relative-path>` 对 brain remote 进行硬删除历史记录
4. 如果漏掉看起来像是 gitleaks 规则差距,请使用该模式向 gitleaks 提出问题(或在 `~/.gitleaks.toml` 扩展 gitleaks 配置)。

## Path 4: 远程 MCP 设置(v1.27.0.0+)

如果您不在本地运行 gbrain — 您有一个队友或另一台机器通过 HTTP 运行 `gbrain serve`,可通过 Tailscale、ngrok 或内部 LAN 访问 — `/setup-gbrain` Path 4 是一次性粘贴的流程。

您提供:
- MCP URL(例如,`https://wintermute.tail554574.ts.net:3131/mcp`)
- 承载令牌(由 brain admin 通过 `gbrain access-token issue` 颁发)

`/setup-gbrain` 做什么:
1. 通过 `gstack-gbrain-mcp-verify` 验证 URL + 令牌。三种失败模式分类为一行一行修复提示:
   **NETWORK**(检查 Tailscale/DNS)、**AUTH**(轮换令牌)、**MALFORMED**(Accept-header 陷阱 — 同时传递 `application/json` 和 `text/event-stream`)。
2. 在用户范围注册 MCP:
   ```
   claude mcp add --scope user --transport http gbrain "$URL" \
     --header "Authorization: Bearer $TOKEN"
   ```
3. 跳过本地安装、本地 doctor、转录摄入和联合来源注册。所有四个都需要 Path 4 不安装的本地 `gbrain` CLI。
4. 可选地在 GitHub 或 GitLab 上配置 `gstack-artifacts-$USER` 私有仓库,并为您的 brain admin 打印一行 `gbrain sources add` 命令,以在 brain 主机上运行。

### 令牌存储权衡

承载令牌位于 `~/.claude.json`(模式 0600)中,Claude Code 在其中存储每个 MCP 服务器的凭证。在 `claude mcp add --header "Authorization: Bearer $TOKEN"` 期间,令牌在进程 argv 中短暂可见(约 10ms) — 同时运行的 `ps` 可见。窗口期很小但不是零。

我们考虑的缓解措施:
- **用于标头的 stdin 或环境变量输入形式** — 将关闭 argv 窗口。截至 Claude Code v1.0.x,CLI 不公开其中任何一个。当它公开时,`/setup-gbrain` Path 4 将自动切换。
- **钥匙串存储** — 明确超出范围(令牌的静止状态在 `~/.claude.json` 中是每个 MCP 凭证的现有信任表面;扩展到钥匙串将涉及每个 MCP 服务器,而不仅仅是 gbrain)。

### 为什么 Path 4 对 brain-admin 挂钩是 "始终打印"

`gstack-artifacts-init` 始终打印标记为 "Send this to your brain admin" 的 `gbrain sources add` 命令 — 即使当用户就是 brain admin 时(一致的 UX,无模式检测脆弱性)。

以前的设计提出探测用户的浏览器授权 bearer 是否具有 admin 范围(通过可视化查询写入调用,如 `add_tag`),并在范围足够时自动执行来源注册。设计审查标记页面写入实际上无法证明来源管理权限 — 在任何合理的身份验证模型中,这些都是不同的范围。在 gbrain 中提供之前:
- 返回承载者的范围集的 `mcp__gbrain__whoami` 能力工具,和
- 具有 admin 范围门控的 `mcp__gbrain__sources_add` MCP工具

我们始终打印该命令,而不是假装我们知道谁有权运行它。

### Path 4 中的 CLAUDE.md 块

与 local-stdio 模式不同。令牌**永远不会**写入 CLAUDE.md(许多项目将 CLAUDE.md 检入 git)。该块记录 URL、经验证的服务器版本、制品仓库 URL(如果已配置)和每仓库信任策略。

```markdown
## GBrain Configuration (configured by /setup-gbrain)
- Mode: remote-http
- MCP URL: https://wintermute.tail554574.ts.net:3131/mcp
- Server version: gbrain v0.27.1
- Setup date: 2026-05-06
- MCP registered: yes (user scope)
- Token: stored in ~/.claude.json (do not commit; never written to CLAUDE.md)
- Artifacts repo: github.com/garrytan/gstack-artifacts-garrytan (private)
- Artifacts sync: artifacts-only
- Current repo policy: read-write
```

### 令牌轮换

服务器端。当 verify 命中 `AUTH`(例如,brain admin 轮换了令牌)时,助手说: "rotate token on the brain host, re-run /setup-gbrain." 在 wintermute 或您的 gbrain 服务器所在的任何位置:

```
gbrain access-token rotate    # 使旧的失效,颁发新的
```

(有关完整的 Path 4 流程,以及围绕作用域令牌的 gbrain 增强请求,将允许 gstack 在 V2 中自动轮换,请参见 `gstack/setup-gbrain/SKILL.md.tmpl`。)