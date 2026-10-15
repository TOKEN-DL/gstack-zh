<!-- AUTO-GENERATED from engine-remediation.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
### Step 1.5 修复:损坏引擎的 AskUserQuestion + 修复分支

用户的本地引擎无法正常工作(例如,`~/.gbrain/config.json` 指向一个死的 Postgres URL)。在 Step 2 之前触发有针对性的 AskUserQuestion:

> D# — Your local gbrain engine isn't responding. How do you want to fix it?
> Project/branch/task: <one-sentence grounding using detected slug + branch>
> ELI10: gbrain has a config at `~/.gbrain/config.json` but the engine it points
> at isn't reachable. That could be a transient outage (Postgres container
> stopped, Tailscale down) OR a stale config you want to abandon. Different
> remediation for each case.
> Stakes if we pick wrong: "Switch to PGLite" overwrites your existing config
> (one-way door if the user actually wanted the broken engine). "Retry" preserves
> existing state for transient cases.
> Recommendation: A (Retry); always try the cheap option first; if engine is
> just temporarily down it'll come back without any destructive change.
> Note: options differ in kind, not coverage; no completeness score.
> A) Retry; re-probe the engine (recommended; ~80ms)
>   ✅ Cheapest test: re-runs `gbrain sources list` to see if engine is back
>   ✅ Zero side effects; existing config preserved
>   ❌ If engine is permanently dead, retries forever; user must choose another option
> B) Switch to local PGLite (one-way; moves existing config to .bak)
>   ✅ Fastest path to a working local engine if user has abandoned the old one
>   ✅ ~30s; no accounts; private to this machine
>   ❌ Destructive; existing config moved to ~/.gbrain/config.json.gstack-bak-{ts}
> C) Switch brain mode (continue to Step 2 path picker)
>   ✅ Lets user pick Path 1/2/3/4 to re-init from scratch
>   ✅ Preserves existing config until they explicitly init the new one
>   ❌ Longer flow if user just wants to repair to PGLite
> D) Quit (do nothing)
>   ✅ No cons; this is a hard-stop choice
>   ❌ N/A
> Net: A is the right starting move; B/C are explicit destructive paths; D bails.

**如果选 A(重试)**: 重新运行 `~/.claude/skills/gstack/bin/gstack-gbrain-detect`,并设置 `GSTACK_DETECT_NO_CACHE=1`(打破 60 秒缓存)。如果新的 `gbrain_local_status` 是 `ok`,继续到 Step 2。如果仍然是 `broken-db` 或 `broken-config`,再次触发相同的 AskUserQuestion(用户再次选择)。

**如果选 B(切换到 PGLite)** — 执行回滚安全的初始化序列:

```bash
BACKUP="$HOME/.gbrain/config.json.gstack-bak-$(date +%s)"
mv "$HOME/.gbrain/config.json" "$BACKUP"
# gstack 默认: voyage-code-3 (1024d) 当设置 VOYAGE_API_KEY 时 — 最适合代码检索。没有密钥时,回退到 gbrain 自己的自动选择嵌入提供方链(当存在 OPENAI_API_KEY 时为 OpenAI 1536d 等)。
# 永不为新大脑选择 gbrain 的旧版 zeroentropyai 配方:其托管 API 的 sunset date(2026 年 9 月 4 日)已过;wireup 助手警告现有安装。
set --  # 标志乘坐位置参数 — 无引号的 $VAR 在 zsh 单词拆分下中断
if [ -n "${VOYAGE_API_KEY:-}" ]; then
  set -- --embedding-model voyage:voyage-code-3 --embedding-dimensions 1024
fi
if ! gbrain init --pglite --json "$@"; then
  # 在失败时恢复
  mv "$BACKUP" "$HOME/.gbrain/config.json"
  echo "gbrain init failed. Your previous config was restored at $HOME/.gbrain/config.json." >&2
  echo "PGLite directory at ~/.gbrain/pglite/ may be in a partial state; \`rm -rf ~/.gbrain/pglite\` if needed before retrying." >&2
  exit 1
fi
echo "Switched to local PGLite. Previous config saved at $BACKUP; review before deleting."
```

然后跳到 Step 5a(MCP 注册;新的 PGLite 引擎注册为 local-stdio)。

**如果选 C(切换大脑模式)**: 继续到 Step 2 的正常路径选择器。

**如果选 D(退出)**: 干净地停止技能。