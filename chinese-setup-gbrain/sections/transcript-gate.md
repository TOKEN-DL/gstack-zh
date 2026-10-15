<!-- AUTO-GENERATED from transcript-gate.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
在 memory 同步连接之后(Step 7),但在持久化 CLAUDE.md 配置之前(Step 8),询问这台机器的编码代理会话转录是否可以进入 gbrain。无论答案如何,策划后的 `~/.gstack/` 制品(learnings、timeline、plans、designs、retros)同步;此问题仅涵盖转录。仅在用户选择 `recent`、`all` 或 `new`(只有从现在开始开始的会话)后才摄入转录。

检查用户是否已经选择。闸门按键于 `has`,因为 `get` 为从未设置的键打印 `off` 默认值:
```bash
_TIM=$(~/.claude/skills/gstack/bin/gstack-config get transcript_ingest_mode 2>/dev/null || true)
if ~/.claude/skills/gstack/bin/gstack-config has transcript_ingest_mode; then
  _T='[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]t[0-9][0-9]:[0-9][0-9]:[0-9][0-9]z'
  case "$(printf '%s' "$_TIM" | tr -d '[:space:]' | tr '[:upper:]' '[:lower:]')" in
    recent|all|off|recent+repos|all+repos|new@$_T|new@$_T+repos)
      echo "TRANSCRIPT_MODE: $_TIM (repos: $(~/.claude/skills/gstack/bin/gstack-config get transcript_repos 2>/dev/null || true))" ;;
    *) echo "TRANSCRIPT_MODE: ask (stored value '$_TIM' is not recognized by this version)" ;;
  esac
else
  echo "TRANSCRIPT_MODE: ask (not set)"
fi
```

- `TRANSCRIPT_MODE:` 带值(`recent`、`all`、`off`、`new@<time>`、任一带 `+repos`): 用户已经选择。不要再次询问;继续到 Step 8。
- `TRANSCRIPT_MODE: ask`: 在下面询问一次。对于 `SESSION_KIND: spawned` 或 `headless`,不要询问,也不要存储值。从不自动选择同意;保持键未设置(转录保持跳过),并报告下一次交互式 `/sync-gbrain` 将询问。

使用两个探测器调整问题大小。`--sources transcript` 使探测器即使尚未同意也计数转录:
```bash
bun run ~/.claude/skills/gstack/bin/gstack-memory-ingest.ts --probe --sources transcript
bun run ~/.claude/skills/gstack/bin/gstack-memory-ingest.ts --probe --sources transcript --all-history
```
读取 `Total files in window`、`Total bytes` 和来自每个的估计。

**未找到转录**(两个探测器报告 0): 改为询问是/否:

> "Ingest coding-agent sessions as they appear?"

是表示 `recent`(然后询问下面的范围问题);否存储 `off`。

**否则**,使用计数 AskUserQuestion。命名两个会话源和来自 Step 4 的目标大脑:用户的 Supabase 大脑(Path 1、2a、2b)或这台机器上的本地 PGLite 大脑(Path 3)。

> "Found <N_recent> Claude Code and Codex sessions from the last 90 days
> (<N_all> in all history, <bytes>) across the projects on this machine
> that your repo policy allows. Ingest them into your <Supabase | local
> PGLite> brain?
>
> What you get: gstack skills load recent context from your past sessions,
> so the agent finds your prior work without you describing it, and you
> can ask 'what was I doing on day X'. Per-session pages are searchable,
> taggable and deletable. Secret scanning runs before any push.
>
> What stays the same: nothing leaves this machine unless brain sync is
> enabled (Step 7). Per-repo trust policies still apply. The first full
> sync can take a while on a large history (estimate: <est>); skills stay
> usable while it runs.
>
> Multi-machine note: with brain sync enabled, transcript pages sync across
> your machines. Deleting a page later removes it from gbrain, but git
> history keeps it in earlier commits. Use `gstack-transcript-prune` to
> delete in bulk, and `git filter-repo` on the brain remote to remove it
> from history."

选项:
- A) 是,过去 90 天(`recent`)
- B) 是,所有历史(`all`)
- C) 是,仅从现在开始的新会话(`new`)
- E) 否,永远不摄入转录(`off`)

在任何 "是" 上,询问第二个独立问题: "Which repos' sessions?"
A) 您的仓库策略允许的每个项目仓库,或 B) 仅此仓库(仅当 `git remote get-url origin` 在此处成功时提供 B)。在已知两个答案之前不存储任何内容;如果用户在其中途中取消,则不存储任何内容。
然后存储范围,然后存储模式值(永远不是字母;`new` 存储 `new@` 加上当前 UTC 时间):
```bash
~/.claude/skills/gstack/bin/gstack-config set transcript_repos "$(git remote get-url origin)"  # 仅此仓库
~/.claude/skills/gstack/bin/gstack-config unset transcript_repos                                 # 每个仓库
~/.claude/skills/gstack/bin/gstack-config set transcript_ingest_mode <recent|all|off|new@$(date -u +%Y-%m-%dT%H:%M:%SZ)>
bun run ~/.claude/skills/gstack/lib/transcript-consent.ts --describe
```
告诉用户 `--describe` 行: 什么将被摄入,文字化。

对于 `recent`、`all` 或 `new@…`,现在运行第一次同步:
```bash
bun run ~/.claude/skills/gstack/bin/gstack-gbrain-sync.ts --full --no-brain-sync
```
(`--no-brain-sync` 因为 Step 7 已经连接该路径;这运行代码导入和 memory 摄入阶段,而 `all` 走遍所有历史。)
对于 `off`,在这里不运行任何内容;其他 memory 仍在下一次 `/sync-gbrain` 上同步。

新转录在下一次 `/sync-gbrain` 运行时被摄入。技能开始时没有任何内容摄入它们。

用户的参考文档: `setup-gbrain/memory.md#transcripts`。