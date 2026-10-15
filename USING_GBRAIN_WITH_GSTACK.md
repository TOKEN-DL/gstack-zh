# Using GBrain with GStack

你的代码代理,带着真正能记住的记忆。

[GBrain](https://github.com/garrytan/gbrain) 是一个为 AI 代理设计的持久知识库。它存储你的代理所学到的、你所决定的、哪些有效、哪些无效,并让代理按需搜索这一切。GStack 给你一条从零到「gbrain 在跑,且我的代理能调用它」的命令路径——包括先试本地、与团队共享,以及中间的一切。

这是全本:每种场景、每个 flag、每个 helper bin、每步故障排除。要快速看卖点,见 [README's GBrain section](README.md#gbrain--persistent-knowledge-for-your-coding-agent)。要错误码与同步特定的问题,见 [docs/gbrain-sync.md](docs/gbrain-sync.md)。

---

## 一条命令的安装

```bash
/setup-gbrain
```

就这样。该 skill 检测你当前的状态,最多问三个问题,并带你走完安装、初始化、为 Claude Code 注册 MCP,以及按 repo 的信任策略。在干净的 Mac 上什么都没装的情况下,五分钟左右完成。在已装好一部分的 Mac 上只需几秒(它检测现有状态并跳过已完成的工作)。

## 安装后你获得什么

一旦 `/setup-gbrain` 结束,你的代码代理就拥有了两个之前没有的检索面:

- **跨本仓库的语义代码搜索。** `gbrain search "browser security canary"` 返回按相关性排序的文件区段,而不是精确匹配的 grep 结果。`gbrain code-def`、`code-refs`、`code-callers`、`code-callees` 按符号遍历调用图——在你不知道哪个文件持有实现、但知道它在做什么时很有用。当问题是语义性的时,代理优先选择这些而非 Grep;CLAUDE.md 获得一个 `## GBrain Search Guidance` 块,教它路由规则。
- **跨会话记忆。** 过去的计划、retros、决策与 learnings 存在 `~/.gstack/`,并且(如果你选择加入 artifacts 同步)被推送到一个 gbrain 会索引的私有 git 仓库。`gbrain search "what did we decide about auth?"` 真的能找到之前的 CEO 计划,而不是你每个会话都重新描述上下文。

如果你还启用了远程 MCP(下面的 Path 4),brain 查询会路由到一个共享 brain 服务器,其他机器也能写入——你的笔记本、台式机、队友的机器都看到相同的记忆。

## 四条路径

当该 skill 问「你的 brain 应该住在哪里?」时,你选一条。

### Path 1: Supabase,你已经有一条连接字符串

适合:你(或队友的云代理)已经 provision 了一个 Supabase brain,你希望本机使用相同的数据。

**会发生什么:** 粘贴 Session Pooler URL(Settings → Database → Connection Pooler → Session → 复制 URI,端口 6543)。该 skill 用 echo off 读取它,向你展示一个脱敏预览(`aws-0-us-east-1.pooler.supabase.com:6543/postgres`——host 可见,密码遮罩),通过 `GBRAIN_DATABASE_URL` 环境变量把它交给 `gbrain init`,URL 永远不会被写到 argv 或你的 shell 历史中。

**信任警告:** 粘贴此 URL 会让你的本地 Claude Code 获得对共享 brain 中每个页面的完整读/写访问。如果那不是你想要的信任级别,改选 PGLite 本地(Path 3),并接受 brain 是不相交的。

### Path 2a: Supabase,自动 provision 一个新项目

适合:全新的 Supabase 账号,你想要一个干净的新项目,零点击。

**会发生什么:** 你粘贴一个 Supabase Personal Access Token (PAT)。该 skill 首先展示 scope 披露——*该 token 授予对你 Supabase 账号中每个项目的完整访问,不仅仅是即将创建的那个*。它列出你的组织,询问你选哪个以及哪个区域(默认 `us-east-1`),生成一个数据库密码,调用 `POST /v1/projects`,每 5 秒轮询 `GET /v1/projects/{ref}` 直到项目 `ACTIVE_HEALTHY`(180 秒超时),获取 pooler URL,把它交给 `gbrain init`。端到端:约 90 秒。

最后:显式提醒在 https://supabase.com/dashboard/account/tokens 撤销该 PAT。该 skill 已把它从记忆中丢弃。

**如果你在 provision 中按 Ctrl-C:** SIGINT 陷阱打印你进行中的项目 ref 与一条 resume 命令。你可以在 Supabase 仪表板删除该孤儿,或跑 `/setup-gbrain --resume-provision <ref>` 接着上次继续。

### Path 2b: Supabase,手动创建

适合:你宁愿自己点击 supabase.com,也不愿粘贴一个 PAT。

**会发生什么:** 该 skill 带你走四个手动步骤(注册 → 新项目 → 等约 2 分钟 → 复制 Session Pooler URL),然后从 Path 1 的粘贴步骤接手。与 Path 1 同样的安全处理。

### Path 3: PGLite 本地

适合:先试一下,无账号、无云、无共享。或一个专用的「这台 Mac 的 brain」,与任何云代理隔离。

**会发生什么:** `gbrain init --pglite`。Brain 位于 `~/.gbrain/brain.pglite`。init 本身无网络调用。30 秒搞定。

**Embedding model。** 当设置了 `VOYAGE_API_KEY` 时,gstack 用 `voyage-code-3`(1024 维)初始化 PGLite——Voyage 专门用于代码的 embedding 模型,在这个代码库的符号查询上击败他们自己的通用 `voyage-4-large` 与 OpenAI `text-embedding-3-large`。没有 `VOYAGE_API_KEY` 时,gbrain 自动选择(`OPENAI_API_KEY` 在场时为 OpenAI 1536 维,否则沿其 provider 链回落)。无论哪种方式,embedding 调用在同步期间会调到所选 provider 的 API——在跑 `/sync-gbrain` 之前为想要的 provider 设置 key。

如果你想先感受一下 gbrain 而不承诺云,这是最佳首选。你以后随时可以用 `/setup-gbrain --switch` 迁移。

### Path 4: 远程 gbrain MCP(split-engine)

适合:你的 brain 跑在你能控制的另一台机器上(Tailscale、ngrok、内网 LAN)或队友的服务器。你想要跨机器记忆的好处而无需自己架起本地数据库,且你仍想要在这台 Mac 上做符号感知的代码搜索。

**会发生什么:** 你粘贴一个 MCP URL(例如 `https://wintermute.tail554574.ts.net:3131/mcp`)与一个 bearer token。该 skill 通过 wire 验证 URL,以用户作用域在 `~/.claude.json` 中把 gbrain 注册为 HTTP MCP,并提议再架起一个小型本地 PGLite 以做代码搜索(约 30 秒,约 120 MB 磁盘)。

如果你接受本地 PGLite,你最终进入**分引擎(split-engine)模式**:

- **Brain/上下文查询**(`mcp__gbrain__search`、`mcp__gbrain__query`、`mcp__gbrain__get_page`)路由到远程 MCP。计划、retros、learnings、跨机器记忆——全在共享服务器上。
- **代码查询**(`gbrain code-def`、`code-refs`、`code-callers`、`code-callees`、用于代码的 `gbrain search`)通过每个 worktree 中的 `.gbrain-source` pin 路由到本地 PGLite。本地索引,快速,从不离开本机。

两个引擎相互独立。擦除本地 PGLite 不会触及远程 brain;轮换远程 MCP bearer 不会影响本地代码搜索。如果你的远程 brain 管理员不能(或不该)索引每位开发者的 checkout,本地代码保持本地——这也是正确配置。

## 为 Claude Code 注册 MCP

默认情况下,该 skill 问「给 Claude Code 一个 gbrain 的类型化工具面?」如果你说 yes,它跑:

```bash
claude mcp add gbrain -- gbrain serve
```

那把 gbrain 的 stdio MCP 服务器注册到 Claude Code。现在 `gbrain search`、`gbrain put`、`gbrain get` 等在每个会话中显示为一等公民工具,而不是 bash shell-out。

**如果 `claude` 不在 PATH 上**,该 skill 优雅地跳过 MCP 注册,带一个手动注册的提示。CLI resolver 仍可从任何 shell-out 到 `gbrain` 的 skill 中工作——MCP 是一个升级,而不是前置条件。

**其他本地代理**(Cursor、Codex CLI 等)需要它们自己的 MCP 注册。该 skill 在 v1 是 Claude Code 目标的;其他 host 可以在它们自己的 MCP config 中手动注册 `gbrain serve`。

## 按远程的信任策略(三位一体)

你机器上的每个仓库得到一个策略决定:**read-write**、**read-only** 或 **deny**。

- **read-write** —— 你的代理可以从本仓库的上下文中 `gbrain search` 并向 brain 写新页面。默认用于你自己的项目。
- **read-only** —— 你的代理可以搜索 brain 但从本仓库的会话中永远不写新页面。适合多客户的咨询师:搜索共享 brain,在客户 B 的仓库里时不要让客户 A 的代码洞见污染它。
- **deny** —— 完全无 gbrain 交互。该仓库对 gbrain 工具不可见。

该 skill 在每个仓库第一次跑一个 gstack skill 时问一次。决定之后是粘性的——同一 git remote 的每个 worktree + 分支共享相同策略,所以你设一次,它会一直跟随。

SSH 与 HTTPS 的远程变体坍缩为相同的键:`https://github.com/foo/bar.git` 与 `git@github.com:foo/bar.git` 是同一仓库。

**更改一个策略:**

```bash
/setup-gbrain --repo      # re-prompt for this repo only

# Or directly:
~/.claude/skills/gstack/bin/gstack-gbrain-repo-policy set "github.com/foo/bar" read-only
```

**查看每个策略:**

```bash
~/.claude/skills/gstack/bin/gstack-gbrain-repo-policy list
```

存储:`~/.gstack/gbrain-repo-policy.json`,模式 0600,带 schema 版本号,让未来迁移保持确定性。

## 用 `/sync-gbrain` 保持 brain 最新

`/setup-gbrain` 是一次性 onboard。`/sync-gbrain` 是每次想让 gbrain 看到本仓库代码新鲜变更时跑的动词。

```bash
/sync-gbrain                # incremental: mtime fast-path, ~seconds on a clean tree
/sync-gbrain --full         # full reindex (~25-35 minutes on a big Mac)
/sync-gbrain --code-only    # only the code stage; skip memory + brain-sync
/sync-gbrain --dry-run      # preview what would sync; no writes
```

该 skill 独立跑三个阶段——code、memory、brain-sync。一个的失败不会阻塞其他。状态持久化到 `~/.gstack/.gbrain-sync-state.json`,所以重跑能干净地接续。

会向离机发送数据的阶段(可能远程的 gbrain DB 中的代码同步、内存摄入、brain-sync 推送)在发送前各自向出账台账(`~/.gstack/security/egress.jsonl`)写一条防篡改的回执,fail-closed:如果回执写不进去,阶段以 `EGRESS_RECEIPT_FAILED` 拒绝,而不是不同步记录未登记的内容。修复通常是 `mkdir -p ~/.gstack/security && chmod -R u+w ~/.gstack/security`,然后重跑。用 `gstack-egress list` 检查回执。

**它在全新 worktree 上做什么:**

1. **Pre-flight.** 检查 `gbrain_local_status`(本地引擎的健康)。如果引擎是 `broken-db` 或 `broken-config`,该 skill 停止并显示一个修复菜单——它拒绝静默降级。如果本地引擎缺失且你在 remote-MCP 模式(Path 4)下,代码阶段干净地 SKIP,只跑 brain-sync。
2. **Code stage.** 通过 `gbrain sources add` 把 cwd 注册为一个联邦源,在仓库根写一个 `.gbrain-source` pin 文件(kubectl-style context——每个 worktree 获得自己的 pin,所以 Conductor 兄弟 worktree 不会冲突),跑 `gbrain sync --strategy code`。
3. **Memory stage.** 暂存你的 `~/.gstack/` 转录与策展记忆。在 local-stdio MCP 模式下,摄入到本地引擎。在 remote-http MCP 模式下,把暂存的 markdown 持久化到 `~/.gstack/transcripts/run-<pid>-<ts>/`,供远程 brain 管理员的 pull 流水线使用。摄入超时默认 30 分钟;为大型 brain 调高用 `GSTACK_INGEST_TIMEOUT_MS`(接受 1 分钟-24 小时)。超时时 gbrain import checkpoint 被保留,所以下一次 `/sync-gbrain` 接着 resume,而不是从头开始。
4. **Brain-sync stage.** 把策展的产物(计划、设计、retros)推送到你配置的私有 artifacts 仓库,如果你有的话。
5. **CLAUDE.md guidance.** 通过 round-trip(写一页 → 搜索 → 找到它)进行能力检查。如果绿色,把 `## GBrain Search Guidance` 块写入你项目的 CLAUDE.md。如果红色,移除该块——代理永远不应该被告知去用一个没安装的工具。

**水印。** 同步状态按 commit hash 推进。如果 gbrain 撞到一个它无法索引的文件(单文件 5 MB 硬限制,或文件在同步中途消失),水印保持不动,后续同步会重试。要承认一次不可修复的失败并向前推进:

```bash
gbrain sync --source <source-id> --skip-failed
```

可重跑、幂等,安全地从同一台机器的多个终端运行(锁在 `~/.gstack/.sync-gbrain.lock`)。

## 稍后切换引擎

选了 PGLite,现在想加入团队 brain?一条命令:

```bash
/setup-gbrain --switch
```

该 skill 跑 `gbrain migrate --to supabase --url "$URL"`,被 `timeout 180s` 包裹。迁移是双向的(Supabase → PGLite 也工作)且无损——页面、块、embeddings、链接、标签与时间线都复制。你原始的 brain 保留为备份。

**如果迁移挂起:** 另一个 gstack 会话可能正持有一个锁在源 brain 上。超时在 3 分钟触发并带一条可操作的消息。关闭其它 workspace,然后重跑 `/setup-gbrain --switch`。

## GStack 记忆同步(一个独立的事)

这与 gbrain 本身不同。你的 gstack 状态(`~/.gstack/`——learnings、plans、retros、timeline、developer profile)默认是本机的。「GStack 记忆同步」可选地把一个策展、秘密扫描过的子集推送到一个私有 git 仓库,这样你的记忆就能跨机器跟随你——并且,如果你在跑 gbrain,该 git 仓库也会在那里可索引。

用以下命令开启:

```bash
gstack-artifacts-init
```

你会收到一次性的隐私提示:**全部允许列表** / **仅 artifacts**(plans、designs、retros、learnings——跳过 timelines 等行为数据) / **关闭**。每次 skill 运行在开始与结束时同步队列——无守护进程,无后台进程。

形状像 secret 的内容(AWS keys、GitHub tokens、PEM 块、JWTs、bearer tokens)在离开本机前就被阻止同步。

**在一台新机器上:** 复制 `~/.gstack-artifacts-remote.txt` 过来(遗留名 `~/.gstack-brain-remote.txt` 仍可工作),跑 `gstack-brain-restore`,昨天的 learnings 就出现在今天的笔记本上。

完整指南:[docs/gbrain-sync.md](docs/gbrain-sync.md)。错误索引:[docs/gbrain-sync-errors.md](docs/gbrain-sync-errors.md)。

`/setup-gbrain` 在初始安装结束时提议为你把这些都接好——它就是再一个 AskUserQuestion,并与同一个私有仓库基础设施集成。

## 清理孤儿项目

如果你在 provision 中按了 Ctrl-C、在确定名字前试了三个不同名字,或以其他方式累积了你不再使用的 gbrain 形状的 Supabase 项目,有一个子命令可以处理:

```bash
/setup-gbrain --cleanup-orphans
```

该 skill 重新收集一个 PAT(一次性,之后丢弃),列出你 Supabase 账号中每个以 `gbrain` 开头且 ref 与你的活跃 `~/.gbrain/config.json` pooler URL 不匹配的项目。对每个孤儿,逐项目问:*「删除孤儿项目 `<ref>` (`<name>`, 创建于 `<date>`)?」*——无批处理、无「全删」快捷方式。活跃 brain 从不被提议删除。

## 命令 + flag 参考

### `/setup-gbrain` 入口模式

| Invocation | What it does |
|---|---|
| `/setup-gbrain` | 完整流程:检测状态、选路径、安装、init、MCP、policy、可选 memory-sync |
| `/setup-gbrain --repo` | 仅切换当前仓库的按远程信任策略 |
| `/setup-gbrain --switch` | 迁移引擎(PGLite ↔ Supabase)而不重跑其它步骤 |
| `/setup-gbrain --resume-provision <ref>` | Resume 一次在轮询期间被打断的 path-2a 自动 provision |
| `/setup-gbrain --cleanup-orphans` | 列出 + 逐项目删除孤儿 Supabase 项目 |

### Bin helpers(用于脚本)

| Bin | Purpose |
|---|---|
| `gstack-gbrain-detect` | Emit current state as JSON: gbrain on PATH, version, config engine, doctor status, sync mode |
| `gstack-gbrain-install` | Detect-first installer (probes `~/git/gbrain`, `~/gbrain`, then fresh clone). Has `--dry-run` and `--validate-only` flags. PATH-shadow check exits 3 with remediation menu. |
| `gstack-gbrain-lib.sh` | Sourced, not executed. Provides `read_secret_to_env VARNAME "prompt" [--echo-redacted "<sed-expr>"]` |
| `gstack-gbrain-supabase-verify` | Structural URL check. Rejects direct-connection URLs (`db.*.supabase.co:5432`) with exit 3 |
| `gstack-gbrain-supabase-provision` | Management API wrapper. Subcommands: `list-orgs`, `create`, `wait`, `pooler-url`, `list-orphans`, `delete-project`. All require `SUPABASE_ACCESS_TOKEN` in env. `create` and `pooler-url` also require `DB_PASS`. `--json` mode available on every subcommand. |
| `gstack-gbrain-repo-policy` | Per-remote trust triad. Subcommands: `get`, `set`, `list`, `normalize` |
| `gstack-gbrain-source-wireup` | Registers your `~/.gstack/` brain repo with gbrain as a federated source via `gbrain sources add` + `git worktree`, then runs an initial `gbrain sync`. Idempotent. Replaces the dead `consumers.json + /ingest-repo` HTTP wireup from v1.12.x. Flags: `--strict`, `--source-id <id>`, `--no-pull`, `--uninstall`, `--probe`. |

### gbrain CLI(上游工具)

Gbrain 本身随附这些,gstack 会包装:

| Command | Purpose |
|---|---|
| `gbrain init --pglite` | Initialize a local PGLite brain |
| `gbrain init --non-interactive` | Initialize via env (`GBRAIN_DATABASE_URL` or `DATABASE_URL`). Never pass a URL as argv — it'll leak to shell history. |
| `gbrain doctor --json` | Health check. Returns `{status: "ok"|"warnings"|"error", health_score: 0-100, checks: [...]}` |
| `gbrain migrate --to supabase --url ...` | Move a PGLite brain to Supabase (lossless, preserves source as backup) |
| `gbrain migrate --to pglite` | Reverse migration |
| `gbrain search "query"` | Search the brain |
| `gbrain put "<slug>" --content "<markdown-with-frontmatter>"` | Write a page (title/tags go in YAML frontmatter inside `--content`) |
| `gbrain get "<slug>"` | Fetch a page |
| `gbrain serve` | Start the MCP stdio server (used by `claude mcp add`) |

### 配置文件 + 状态

| Path | What lives there |
|---|---|
| `~/.gbrain/config.json` | Engine (pglite/postgres), database URL or path, API keys. Mode 0600. Written by `gbrain init`. |
| `~/.gstack/gbrain-repo-policy.json` | Per-remote trust triad. Schema v2. Mode 0600. |
| `~/.gstack/.setup-gbrain.lock.d` | Concurrent-run lock (atomic mkdir). Released on normal exit + SIGINT. |
| `~/.gstack/.brain-queue.d/` | Pending sync records for gstack memory sync — maildir-style spool, one file per record. A legacy `.brain-queue.jsonl` from older releases migrates automatically on the next drain. |
| `~/.gstack/.brain-last-push` | Timestamp of last sync push (for `/health` scoring) |
| `~/.gstack-artifacts-remote.txt` | URL of your gstack memory sync remote (safe to copy between machines; legacy name `~/.gstack-brain-remote.txt` still read) |
| `~/.gstack/.setup-gbrain-inflight.json` | Reserved for future `--resume-provision` persisted state |

### Environment variables

| Var | Where it's read | What it does |
|---|---|---|
| `SUPABASE_ACCESS_TOKEN` | `gstack-gbrain-supabase-provision` | PAT for Management API calls. Discarded after each setup run. |
| `DB_PASS` | `gstack-gbrain-supabase-provision` (create, pooler-url) | Generated DB password. Never in argv. |
| `GBRAIN_DATABASE_URL` | `gbrain init`, `gbrain doctor`, etc. | Postgres connection string (Supabase pooler URL for us). Env takes precedence over `~/.gbrain/config.json`. |
| `DATABASE_URL` | `gbrain init` (fallback) | Same semantics as `GBRAIN_DATABASE_URL`; checked second. |
| `SUPABASE_API_BASE` | `gstack-gbrain-supabase-provision` | Override the Management API host. Used by tests to point at a mock server. |
| `GBRAIN_INSTALL_DIR` | `gstack-gbrain-install` | Override default install path (`~/gbrain`) |
| `GSTACK_HOME` | every bin helper | Override `~/.gstack` state dir. Heavy test use. |
| `VOYAGE_API_KEY` | `gbrain embed` subprocess; gstack PGLite init | When set, gstack inits PGLite with `voyage-code-3` (1024-dim), Voyage's code-specialized embedding model. Beats `voyage-4-large` and OpenAI `text-embedding-3-large` head-to-head on this codebase's symbol queries. See CHANGELOG v1.43.1.0 for the A/B numbers. |
| `OPENAI_API_KEY` | `gbrain embed` subprocess | Used for embeddings during `gbrain sync` / `/sync-gbrain` when `VOYAGE_API_KEY` is not set (gbrain's auto-selected fallback, `text-embedding-3-large` 1536-dim). Without either key, pages are imported structurally (symbol tables, chunks) but semantic search degrades — you'll see `[gbrain] embedding failed for code file ...` in the sync log. |
| `ANTHROPIC_API_KEY` | `claude-agent-sdk`, paid evals | Required for paid evals (`bun run eval:bg:pr`) and any direct `query()` call against Claude. |
| `GSTACK_OPENAI_API_KEY` | `lib/conductor-env-shim.ts` | Conductor-injected fallback. Promoted to `OPENAI_API_KEY` when the canonical name is empty. |
| `GSTACK_ANTHROPIC_API_KEY` | `lib/conductor-env-shim.ts` | Same pattern as above for Anthropic. |

## Conductor + GSTACK_* env vars

如果你在 [Conductor](https://conductor.build) workspace 内跑 gstack,**Conductor 显式从 workspace env 中剥离 `ANTHROPIC_API_KEY` 与 `OPENAI_API_KEY`**。在 `~/.zshrc` 或 `.env` 中设置它们无济于事——剥离发生在 env 继承之后。要把一个可用的 API key 放入 workspace,在 Conductor 的 workspace env 配置中设置 `GSTACK_ANTHROPIC_API_KEY` 与 `GSTACK_OPENAI_API_KEY`。Conductor 原样传递这些。

`lib/conductor-env-shim.ts` 在 gstack 一侧桥接这个空隙:作为副作用被 import 时(`import "../lib/conductor-env-shim";`),它把 `GSTACK_FOO_API_KEY` 提升为 `FOO_API_KEY`,供任何看不到规范名称的子进程使用。shim 已经接入:

- `bin/gstack-gbrain-sync.ts` —— 让 `/sync-gbrain` 拿到 OpenAI 做 embedding
- `bin/gstack-model-benchmark` —— 让 `--judge` 跑起来,无需手动 env 映射
- `scripts/preflight-agent-sdk.ts` —— 让付费 eval 鉴权探针工作
- `test/helpers/e2e-helpers.ts` —— 让付费 eval 找到 Anthropic

如果你添加新的 TS 入口,它们会用到付费 API 或需要 gbrain embedding,在顶部加同样的单行 import。见 [CONTRIBUTING.md "Conductor workspaces"](CONTRIBUTING.md#conductor-workspaces) 的贡献者清单。

`bin/gstack-codex-probe` 是 bash,不会直接读这些——它依赖 `~/.codex/` 中由 Codex CLI 管理的鉴权。

## Security model

该 skill 触及的每个 secret 有一条规则:**仅走 env var,永不 argv,永不记录,永不由我们写入磁盘**。唯一的持久化存储是 gbrain 自己的 `~/.gbrain/config.json`,模式 0600,这是 gbrain 的纪律,不是我们的。

**代码强制:**

- `test/skill-validation.test.ts` 中的 CI grep 测试如果 `$SUPABASE_ACCESS_TOKEN` 或 `$GBRAIN_DATABASE_URL` 出现在 argv 位置会让构建失败
- CI grep 测试如果 `--insecure`、`-k` 或 `NODE_TLS_REJECT_UNAUTHORIZED=0` 出现在 `bin/gstack-gbrain-supabase-provision` 中会失败
- provision helper 顶部的 `set +x` 防止 debug trace 泄漏 PAT
- 遥测 payload 仅包含枚举的分类值(scenario、install result、MCP opt-in、trust tier)——永远不是可能包含 secret 的自由形式字符串

**通过测试强制:**

- `test/secret-sink-harness.test.ts` 用播种的 secret 跑每个 secret-handling bin,并断言 seed 不会出现在任何捕获的通道中(stdout、stderr、`$HOME` 下的文件、遥测 JSONL)。每个 seed 四个匹配规则:精确、URL-decoded、前 12 字符前缀、base64。
- 同一测试文件中的正面控制故意在每个覆盖的通道中泄漏 seed,并断言 harness 捕获每个。如果不是正面控制,一个静默少报 harness 会与正常工作的 harness 看起来一样。

**你仍然可能泄漏什么**(v1 的诚实限制):

- 如果你把 secret 粘贴到 `read -s` 之外的普通聊天消息中,它就会出现在对话转录与任何 host 端日志中
- 泄漏 harness 不会转储子进程环境——一个 `env >> ~/.log` 的 bin 会规避检测(v1 中无 bin 这样干;grep 测试阻止它)
- 你的 shell 自己的 `HISTFILE` 行为属于你的 shell,不是我们的——我们从不在 argv 中传 secret,所以它们不会通过我们的代码进入那里,但没有什么能阻止你自己把一个粘到原始 `curl` 命令里

## Troubleshooting

### 安装时 "PATH SHADOWING DETECTED"

另一个 `gbrain` 二进制在 PATH 中比安装器刚刚链接的更早。安装器的版本检查捕获了它。修一个:

- `rm $(which gbrain)` 如果你不需要另一个
- 在你的 shell rc 中把 `~/.bun/bin` 拼到 PATH 前,让链接的二进制胜出
- 设置 `GBRAIN_INSTALL_DIR` 为 shadowing 二进制的安装目录,然后重跑

然后重跑 `/setup-gbrain`。

### "rejected direct-connection URL"

你粘贴了一个 `db.<ref>.supabase.co:5432` URL。那些是仅 IPv6 的,在多数环境下会失败。改用 Session Pooler URL:Supabase dashboard → Settings → Database → Connection Pooler → **Session** → 复制 URI(端口 6543)。

### 自动 provision 在 180 秒处超时

Supabase 项目仍在初始化。你的 ref 已打印在 exit 消息中。等一分钟,然后:

```bash
/setup-gbrain --resume-provision <ref>
```

该 skill 重新收集一个 PAT,跳过项目创建,resume 轮询。

### "Another `/setup-gbrain` instance is running"

你有一个过期的锁目录。如果你确定没有其他实例在跑:

```bash
rm -rf ~/.gstack/.setup-gbrain.lock.d
```

然后重跑。

### 策略文件上 "No cross-model tension"

你手撸了 `~/.gstack/gbrain-repo-policy.json` 并带遗留的 `allow` 值?没问题。在下次读取时,gstack 自动把 `allow` 迁移到 `read-write` 并加 `_schema_version: 2`。stderr 上一行日志,幂等,确定性。

### `gbrain doctor` 说 "warnings"

`/health` 把那个视为黄灯而不是红灯。用 `gbrain doctor --json | jq .checks` 查看哪个子检查是 warning。典型原因:resolver MECE 重叠(skill 名冲突)或 DB 连接尚未配置。

### `/sync-gbrain` 报 `OK` 但 `gbrain search` 没有返回任何语义结果

Embedding 可能在 import 期间失败了。符号查询(`code-def`、`code-refs`)仍能工作,因为它们不需要 embedding,但 `gbrain search "<terms>"` 回落到一个降级的 BM25 路径。在同步输出中查找这样的行:

```
[gbrain] embedding failed for code file <name>: OpenAI embedding requires OPENAI_API_KEY
```

修复方法是在重新跑前把一个 provider API key 放进进程 env。`VOYAGE_API_KEY` 在代码上更优(gstack 在设置时把 PGLite 默认成 `voyage-code-3`);否则 `OPENAI_API_KEY` 回落为 `text-embedding-3-large`。在裸 Mac shell 上,在调用前从 `~/.zshrc` 源出 key。在 Conductor 中,`lib/conductor-env-shim.ts` shim 自动把 `GSTACK_ANTHROPIC_API_KEY` / `GSTACK_OPENAI_API_KEY` 提升为它们的规范名;对于 `VOYAGE_API_KEY`,在 Conductor workspace env 中直接设置。重跑 `/sync-gbrain --code-only` 来回填已导入页面的 embedding。

### `gbrain sync` 在 commit hash 处卡住 —— `FILE_TOO_LARGE`

你的树中有一个文件超过 gbrain 的 5 MB 硬限制(`gbrain/src/core/import-file.ts` 中的 `MAX_FILE_SIZE`)。常见元凶:响应回放缓存、捕获的截图、大型 JSON fixtures。Gbrain 不为代码同步尊重 `.gitignore` 风格的排除列表;唯一的旋钮是承认失败:

```bash
gbrain sync --source <source-id> --skip-failed
```

水印会越过那个出问题的 commit 推进。如果该文件再次变化,会再次失败;再次 skip。

### ZeroEntropy embedding 在 2026 年 9 月 4 日之后停止工作

ZeroEntropy 被 Notion 收购,将于 **2026 年 9 月 4 日** 停用其托管 API(新注册已禁用)。一个用 `zeroentropyai` embedding recipe 配置的 gbrain 在该日期之后继续导入页面,但 embedding 静默失败——页面以结构化形式落地,无语义搜索。wireup helper 在你的 `~/.gbrain/config.json` 命名该 recipe 时警告;在截止日期前迁移到另一 provider(通过 `VOYAGE_API_KEY` 的 Voyage,或通过 `OPENAI_API_KEY` 的 OpenAI)。细节、自托管注意事项与迁移讨论:[garrytan/gstack#2365](https://github.com/garrytan/gstack/issues/2365)。

### PGLite → Supabase 切换卡住

一个兄弟 Conductor workspace 中的另一个 gstack 会话可能正通过其序章中的 `gstack-brain-sync` 调用持有一个锁在你的本地 PGLite 文件上。关闭其它 workspace,重跑 `/setup-gbrain --switch`。超时限定在 180 秒,所以你永远实际上不会等太久。

## Why this design

**为什么是按远程信任三位一体,而不是二元的 allow/deny?** 多客户咨询师需要搜索而不回写。一个咨询师上午在客户 A、下午在客户 B,不能让 A 的代码洞见泄漏到客户 B 能搜索的 brain 中。Read-only 干净地解决了这个问题。

**为什么不把 gbrain 捆绑进 gstack?** Gbrain 是一个独立的、积极开发中的项目,有自己的发布节奏、schema 迁移与 MCP 面。捆绑意味着 gstack 必须 gate gbrain 的更新,这会减慢 gbrain 改进触达用户的速度。分开但集成让各自按自己的节奏发布。

**为什么是 `gbrain init --non-interactive` 通过 env var,而不是一个 flag?** 连接字符串包含数据库密码。把它们作为 argv 传会让密码进入 `ps`、shell 历史与进程列表。Env-var 交接让 secret 只在进程内存中。Gbrain 同时支持 `GBRAIN_DATABASE_URL` 与 `DATABASE_URL`;我们用前者是为了避免与非 gbrain 工具冲突。

**为什么在 PATH shadowing 上 fail-hard 而不是 warn-and-continue?** 被 shadow 的 `gbrain` 意味着每个后续命令调用的都是一个与我们刚安装的不同的二进制。这是一个静默的版本漂移 bug,在几周后才以神秘的特性缺失形式浮出水面。Setup skill 只有一个工作——搭建一个能用的环境。拒绝安装到一个坏环境是 setup-skill 正确的行为。

**为什么不自动导入每个仓库?** 隐私 + 噪声。一个自动导入序章 hook 会: (a) 未经同意就把工作代码泄漏进共享 brain,(b) 用一次性仓库堵塞搜索。按远程策略让摄入成为一个显式的、每仓库的决定。`/setup-gbrain` 今天没有安装任何自动导入 hook——但策略存储对之后的一个向前兼容。

## Related skills + next steps

- `/health` —— 在其 0-10 综合分中包含一个 GBrain 维度(doctor 状态、同步队列深度、上次推送年龄)。该维度在 gbrain 未安装时被省略;在非 gbrain 机器上跑 `/health` 不会因为这个选择而扣分。
- `/gstack-upgrade` —— 让 gstack 本身保持最新。**不**独立升级 gbrain。gbrain 默认按最新 HEAD 安装;要刷新,在你的 gbrain clone 中 `git pull`(默认 `~/gbrain`)并重跑 `/setup-gstack`。用 `gstack-gbrain-install --pinned-commit <sha>` 钉住特定 commit,如果需要可复现性。低于最低测试版本的安装会被拒绝。
- `/retro` —— 每周复盘从你的 gbrain 拉取 learnings 与 plans(当 memory sync 开启时),让复盘能引用跨机器历史。

跑 `/setup-gbrain` 看看哪些能落地。
