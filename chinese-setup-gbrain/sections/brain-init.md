<!-- AUTO-GENERATED from brain-init.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
Path-specific。仅在 Step 2 中选择的路径(或当 Step  为引擎迁移选择 Switch 流程时)运行下面的子节。

### Path 1 (Supabase,existing URL)

源代码 secret-read 助手,通过 `read -s` + 模糊预览收集 URL:

```bash
. ~/.claude/skills/gstack/bin/gstack-gbrain-lib.sh
read_secret_to_env GBRAIN_POOLER_URL "Paste Session Pooler URL: " \
  --echo-redacted 's#://[^@]*@#://***@#'
```

然后从结构上验证:

```bash
printf '%s' "$GBRAIN_POOLER_URL" | ~/.claude/skills/gstack/bin/gstack-gbrain-supabase-verify -
```

如果验证退出码是 3(direct-connection URL),验证器自己的消息解释修复;表面它并重新提示 Session Pooler URL。

成功后,通过环境变量移交给 gbrain(永不通过 argv):

```bash
GBRAIN_DATABASE_URL="$GBRAIN_POOLER_URL" gbrain init --non-interactive --json
```

然后立即 `unset GBRAIN_POOLER_URL GBRAIN_DATABASE_URL`。该 URL 现在由 gbrain 本身以模式 0600 持久化在 `~/.gbrain/config.json` 中。

### Path 2a (Supabase,auto-provision)

在收集令牌之前按字面显示此 PAT 范围披露:

> *This Supabase Personal Access Token grants full read/write/delete access
> to every project in your Supabase account, not just the `gbrain` one we're
> about to create. Supabase doesn't currently support scoped tokens. We use
> this PAT only to: create one project, poll it until healthy, read the
> Session Pooler URL; then discard it from process memory. The token
> remains valid on Supabase's side until you manually revoke it at
> https://supabase.com/dashboard/account/tokens; we recommend revoking
> immediately after setup completes.*

然后:

```bash
. ~/.claude/skills/gstack/bin/gstack-gbrain-lib.sh
read_secret_to_env SUPABASE_ACCESS_TOKEN "Paste PAT: "
```

通过 AskUserQuestion 询问层问题: "Which Supabase tier?" 展示 Free(2 项目限制,7 天不活动后暂停)与 Pro($25/月,无暂停,推荐用于实际使用)。说明层是**组织级别的**(根据管理 API 合约);用户根据其当前层选择他们的组织。Pro 可能要求他们首先在 supabase.com 上升级组织。

列出组织,选择一个(如果多个则使用 AskUser 提示):

```bash
orgs=$(~/.claude/skills/gstack/bin/gstack-gbrain-supabase-provision list-orgs --json)
```

如果 `.orgs` 数组为空,则表面: "Your Supabase account has no organizations. Create one at https://supabase.com/dashboard, then re-run `/setup-gbrain`." 停止。

询问用户的区域(默认 `us-east-1`;有效值是 Supabase 管理 API 中的 18 个枚举值 — 列出一些常见的,让他们选择 "Other" 以获取完整列表)。

生成 DB 密码(永不向用户显示):

```bash
export DB_PASS=$(openssl rand -base64 24)
```

设置 SIGINT trap,以便可以恢复或删除中断的配置:

```bash
trap 'echo ""; echo "gstack-gbrain: interrupted. In-flight ref: $INFLIGHT_REF"; \
      echo "Resume: /setup-gbrain --resume-provision $INFLIGHT_REF"; \
      echo "Delete: https://supabase.com/dashboard/project/$INFLIGHT_REF"; \
      unset SUPABASE_ACCESS_TOKEN DB_PASS; exit 130' INT TERM
```

创建 + 等待 + 获取:

```bash
result=$(~/.claude/skills/gstack/bin/gstack-gbrain-supabase-provision \
  create gbrain "$REGION" "$ORG_SLUG" --json)
INFLIGHT_REF=$(echo "$result" | jq -r .ref)
~/.claude/skills/gstack/bin/gstack-gbrain-supabase-provision wait "$INFLIGHT_REF" --json
pooler=$(~/.claude/skills/gstack/bin/gstack-gbrain-supabase-provision \
  pooler-url "$INFLIGHT_REF" --json)
GBRAIN_DATABASE_URL=$(echo "$pooler" | jq -r .pooler_url)
export GBRAIN_DATABASE_URL
gbrain init --non-interactive --json
unset SUPABASE_ACCESS_TOKEN DB_PASS GBRAIN_DATABASE_URL INFLIGHT_REF
trap - INT TERM
```

成功后,发出 PAT 撤销提醒:

> "Setup complete. Revoke the PAT you pasted at
> https://supabase.com/dashboard/account/tokens; we've already discarded
> it from memory and don't need it again. The gbrain project will continue
> working because it uses its own embedded database password."

### Path 2b (Supabase,manual)

引导用户完成 supabase.com 步骤:
1. 登录 https://supabase.com/dashboard
2. 点击 "New Project",命名为 `gbrain`,选择一个区域,并复制生成的数据库密码 — 步骤 4 中的 pooler URL 必须携带它(粘贴前替换任何 `[YOUR-PASSWORD]` 占位符)
3. 等待约 2 分钟以初始化项目
4. Settings → Database → Connection Pooler → Session → 复制 URL(端口 6543)

然后遵循与 Path 1 相同的 secret-read + verify + init 流程。

### Path 3 (PGLite local)

```bash
# gstack 默认: voyage-code-3 (1024d) 当设置 VOYAGE_API_KEY 时 — 在实际代码查询上,代码检索击败通用嵌入(经过 A/B 验证)。没有密钥时,gbrain 自动选择(可用时为 OpenAI 1536d)。
# 永不为新大脑选择 gbrain 的旧版 zeroentropyai 配方:其托管 API 的 sunset date(2026 年 9 月 4 日)已过;wireup 助手警告现有安装。
set --  # 标志乘坐位置参数 — 无引号的 $VAR 在 zsh 单词拆分下中断
if [ -n "${VOYAGE_API_KEY:-}" ]; then
  set -- --embedding-model voyage:voyage-code-3 --embedding-dimensions 1024
fi
gbrain init --pglite --json "$@"
```

完成。无网络,无秘密(除了同步期间的 Voyage 嵌入 API 调用,如果设置了 `VOYAGE_API_KEY` — 每 1M 令牌约 0.18 美元,每个仓库几美分)。

### Path 4 (Remote gbrain MCP — 带承载令牌的 HTTP 传输)

对于其大脑在另一台机器上的用户(Tailscale、ngrok、内部 LAN 或队友的服务器)。无本地 gbrain CLI 安装,无本地数据库。此技能注册远程 MCP 并停止;摄入 + 索引发生在 brain 主机上。

**4a. 收集 MCP URL。** 提示用户:

```
Paste your gbrain MCP URL (e.g. https://wintermute.tail554574.ts.net:3131/mcp):
```

使用普通 `read -r` 读取(不需要秘密卫生 — URL 本身不是凭证)。验证它以 `https://` 开头(对任何非环回主机要求 TLS);对非 localhost 拒绝 `http://`。

**4b. 通过 secret-read 助手收集承载令牌(永不通过 argv)。**

```bash
. ~/.claude/skills/gstack/bin/gstack-gbrain-lib.sh
read_secret_to_env GBRAIN_MCP_TOKEN "Paste bearer token: " \
  --echo-redacted 's/.\{6\}$/***REDACTED***/'
```

**4c. 通过 gstack-gbrain-mcp-verify 验证。** 运行助手;捕获分类的 JSON 输出:

```bash
verify_json=$(GBRAIN_MCP_TOKEN="$GBRAIN_MCP_TOKEN" \
  ~/.claude/skills/gstack/bin/gstack-gbrain-mcp-verify "$MCP_URL")
status=$(echo "$verify_json" | jq -r .status)
```

如果 `status != "success"`,助手已将失败分类为 NETWORK / AUTH / MALFORMED,并发出一行修复提示。在来自 `error_text` 的原始错误之上显示提示,然后**停止**,并显示 "fix and re-run /setup-gbrain" 消息。不要继续运行认证以在失败的 verify 上进入 Step 5a — 部分注册会让用户处于半损坏状态。

从 verify 输出中捕获两个值以供下游步骤使用:
- `SERVER_VERSION`(例如,`0.27.1`) — 写入 Step 8 中的 CLAUDE.md 块。
- `URL_FORM_SUPPORTED`(`true|false`) — 在 Step 7 中传递给 `gstack-artifacts-init` 以控制打印 brain-admin 挂钩命令的哪种形式。

**4d. (Path 4) 为代码搜索提供本地 PGLite。** 询问:

> D# — Want symbol-aware code search on this machine?
> Project/branch/task: <one-sentence grounding using detected slug + branch>
> ELI10: The remote brain at `<MCP_URL>` is great for cross-machine knowledge,
> but symbol queries like `gbrain code-def` / `code-refs` / `code-callers` need
> a local index of THIS machine's code. We can spin up a tiny isolated PGLite
> database (~30 seconds, no accounts, ~120 MB disk) just for code, separate
> from your remote brain. Transcripts and artifacts continue routing through
> the artifacts repo to the remote brain; local PGLite stays code-only.
> Stakes: without it, semantic code search in this repo's worktrees falls
> back to Grep.
> Recommendation: A — 30 seconds, no ongoing cost, unlocks the symbol tools.
> Completeness: A=10/10 (full split-engine), B=7/10 (remote-only).
> A) Yes, set up local PGLite for code (recommended)
>   ✅ Unlocks `gbrain code-def`, `code-refs`, `code-callers` per worktree
>   ✅ Independent engine; won't disturb remote brain or share transcripts
> B) No, remote MCP only
>   ✅ Zero local state; only `~/.claude.json` MCP registration
>   ❌ Symbol code queries fall back to Grep in this repo's worktrees
> Net: A = full split-engine; B = remote-only.

**如果选 A(是)**: 安装 + 初始化本地 PGLite,具有回滚安全语义:

```bash
~/.claude/skills/gstack/bin/gstack-gbrain-install || exit $?
# 此时本地 gbrain CLI 在 PATH 上。初始化 PGLite,但首先备份任何现有的 ~/.gbrain/config.json(如果 init 失败则回滚)。
if [ -f "$HOME/.gbrain/config.json" ]; then
  BACKUP="$HOME/.gbrain/config.json.gstack-bak-$(date +%s)"
  mv "$HOME/.gbrain/config.json" "$BACKUP"
fi
# gstack 本地代码搜索 PGLite 的默认值:voyage-code-3 (1024d) 当设置 VOYAGE_API_KEY 时。它在此代码库的符号查询上胜过 voyage-4-large 和 OpenAI text-embedding-3-large 的 A/B。密钥不存在时回退到 gbrain 的自动选择提供方。
set --  # 标志乘坐位置参数 — 无引号的 $VAR 在 zsh 单词拆分下中断
if [ -n "${VOYAGE_API_KEY:-}" ]; then
  set -- --embedding-model voyage:voyage-code-3 --embedding-dimensions 1024
fi
if ! gbrain init --pglite --json "$@"; then
  if [ -n "${BACKUP:-}" ] && [ -f "$BACKUP" ]; then mv "$BACKUP" "$HOME/.gbrain/config.json"; fi
  echo "gbrain init failed. Existing config (if any) was restored. PGLite at ~/.gbrain/pglite/ may be in a partial state; \`rm -rf ~/.gbrain/pglite\` to reset." >&2
  echo "Continuing setup without local code search; you can re-run /setup-gbrain to retry." >&2
fi
```

然后继续到 Step 5a。5a 中的 remote-http MCP 注册不变地运行;本地 PGLite 独立于 MCP 注册(Claude Code 通过 MCP 与远程大脑通信以进行查询;`gbrain` CLI 与本地 PGLite 通信以进行 code-def/refs/callers)。

**如果选 B(否)**: 跳过安装 + 初始化。本地引擎保持缺席。
`gbrain_local_status` 将是 `missing-config`(如果未安装 gbrain 则为 `no-cli`)。`/sync-gbrain` 将干净地跳过代码阶段。

**4e. 当选择 B 时跳过 Step 3、4(其他路径)和 5(本地 doctor)。**
当选择 A 时,Step 3 已经通过 gstack-gbrain-install 运行,Step 4 已经通过 `gbrain init --pglite` 运行;直接跳到 Step 5a。当选择 B 时,Steps 3/4/5 是无操作;还要跳过 Step 7.5(转录摄入),因为在 remote-http 模式下,memory-stage 通过制品管道路由到 brain。

承载令牌(`GBRAIN_MCP_TOKEN`)在 Step 5a 的 `claude mcp add --header` 消耗之前一直保留在进程环境中;然后立即 `unset GBRAIN_MCP_TOKEN`。在 `setup-gbrain/memory.md` 中记录令牌安全权衡: `claude mcp add` 期间的简短 argv 暴露,静止状态在 `~/.claude.json` 模式 0600 中。

### Switch(从检测的现有引擎状态)

```bash
# PGLite → Supabase,首先收集 URL(Path 1 流程),然后:
timeout 180s gbrain migrate --to supabase --url "$URL" --json
# Supabase → PGLite:
timeout 180s gbrain migrate --to pglite --json
```

如果 `timeout` 返回 124(超时的退出码):表面此消息("Migration didn't complete in 3 minutes; another gstack session may be holding a lock on the source brain. Close other workspaces and re-run `/setup-gbrain --switch`. Your original brain is untouched.")。停止。