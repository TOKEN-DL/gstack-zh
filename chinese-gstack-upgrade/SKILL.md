---
name: gstack-upgrade
version: 1.1.0
description: 把 gstack 升级到最新版本。
triggers:
  - upgrade gstack
  - update gstack version
  - get latest gstack
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

检测全局安装还是 vendored 安装，
运行升级流程，并展示新增内容。当用户要求 "upgrade gstack"、"update gstack"，或 "get latest version" 时使用。

语音触发（speech-to-text aliases）："upgrade the tools"、"update the tools"、"gee stack upgrade"、"g stack upgrade"。

# /gstack-upgrade

把 gstack 升级到最新版本并展示新增内容。

## 内联升级流程

本节供所有 skill preamble 在检测到 `UPGRADE_AVAILABLE` 时引用。

### 步骤 1:询问用户（或自动升级）

首先,检查是否开启了自动升级：
```bash
_AUTO=""
[ "${GSTACK_AUTO_UPGRADE:-}" = "1" ] && _AUTO="true"
[ -z "$_AUTO" ] && _AUTO=$(~/.claude/skills/gstack/bin/gstack-config get auto_upgrade 2>/dev/null || true)
echo "AUTO_UPGRADE=$_AUTO"
```

**如果 `AUTO_UPGRADE=true` 或 `AUTO_UPGRADE=1`:** 跳过 AskUserQuestion。记录 "Auto-upgrading gstack v{old} → v{new}..."，然后直接进入步骤 2。setup 失败时,遵循步骤 4 中按安装方式区分的恢复路径：vendored 安装恢复其备份；git 安装则记录 pre-upgrade commit 后中止,不进行破坏性 reset。除非真的成功恢复,否则不要声称已恢复。

**否则**，使用 AskUserQuestion:
- 问题："gstack **v{new}** 可用（你正在用 v{old}）。现在升级吗？"
- 选项:["Yes, upgrade now"、"Always keep me up to date"、"Not now"、"Never ask again"]

**如果是"Yes, upgrade now":** 继续步骤 2。

**如果是 "Always keep me up to date":**
```bash
~/.claude/skills/gstack/bin/gstack-config set auto_upgrade true
```
告诉用户："Auto-upgrade enabled. Future updates will install automatically." 然后继续步骤 2。

**如果是 "Not now":** 以递增的回退时间写入 snooze 状态（首次 snooze = 24 小时，第二次 = 48 小时，第三次及以后 = 1 周），然后继续执行当前 skill。不要再提升级的事。
```bash
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
_SNOOZE_FILE="$GSTACK_STATE_ROOT/update-snoozed"
_REMOTE_VER="{new}"
_CUR_LEVEL=0
if [ -f "$_SNOOZE_FILE" ]; then
  _SNOOZED_VER=$(awk '{print $(1)}' "$_SNOOZE_FILE")
  if [ "$_SNOOZED_VER" = "$_REMOTE_VER" ]; then
    _CUR_LEVEL=$(awk '{print $(2)}' "$_SNOOZE_FILE")
    case "$_CUR_LEVEL" in *[!0-9]*) _CUR_LEVEL=0 ;; esac
  fi
fi
_NEW_LEVEL=$((_CUR_LEVEL + 1))
[ "$_NEW_LEVEL" -gt 3 ] && _NEW_LEVEL=3
echo "$_REMOTE_VER $_NEW_LEVEL $(date +%s)" > "$_SNOOZE_FILE"
```
注意：`{new}` 来自 `UPGRADE_AVAILABLE` 输出中的远端版本——请从 update check 结果中替换。

告诉用户 snooze 时长："Next reminder in 24h"（或 48h 或 1 周，取决于 level）。提示："Set `auto_upgrade: true` in `~/.gstack/config.yaml` for automatic upgrades."

**如果是 "Never ask again":**
```bash
~/.claude/skills/gstack/bin/gstack-config set update_check false
```
告诉用户："Update checks disabled. Run `~/.claude/skills/gstack/bin/gstack-config set update_check true` to re-enable."
继续执行当前 skill。

### 步骤 2:检测安装类型

```bash
if [ -d "$HOME/.claude/skills/gstack/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.claude/skills/gstack"
elif [ -d "$HOME/.gstack/repos/gstack/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.gstack/repos/gstack"
elif [ -d ".claude/skills/gstack/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".claude/skills/gstack"
elif [ -d ".agents/skills/gstack/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".agents/skills/gstack"
elif [ -d ".claude/skills/gstack" ]; then
  INSTALL_TYPE="vendored"
  INSTALL_DIR=".claude/skills/gstack"
elif [ -d "$HOME/.claude/skills/gstack" ]; then
  INSTALL_TYPE="vendored-global"
  INSTALL_DIR="$HOME/.claude/skills/gstack"
elif _SRC=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT 2>/dev/null) && _SRC=$(awk -F '\t' '$(1) == "claude" && $(6) != "-" { print $(6); exit }' "$_SRC/installs.tsv" 2>/dev/null) && [ -d "$_SRC/.git" ]; then
  # 安装登记表记录了为本机启动的 checkout 设置(例如一个 ~/gstack 克隆)。
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$_SRC"
else
  echo "ERROR: gstack not found"
  exit 1
fi
INSTALL_DIR=$(cd -- "$INSTALL_DIR" && pwd -P) || { echo "ERROR: cannot enter the gstack install directory" >&2; exit 1; }
echo "Install type: $INSTALL_TYPE at $INSTALL_DIR"
```

上述安装类型和绝对目录将用于后续所有步骤。
明确地把 `INSTALL_TYPE`、`INSTALL_DIR`、`OLD_VERSION` 以及稍后的 `NEW_VERSION` 一路传下去：tool call 可能在全新的 shell 中运行，所以后续每个 block 开头都要从捕获的输出中给它们赋值（例如 `INSTALL_DIR=/abs/path/printed/above`）。不要依赖前一次调用的工作目录或 shell 变量。凡是要改动文件的 block，都要先检查 `INSTALL_DIR`：当其未设置、为空、缺失、不可读、或不是 gstack 自己的 checkout 时，在执行任何 git 命令前都要先中止。

### 步骤 3:保存旧版本

使用步骤 2 输出中的安装目录：

```bash
OLD_VERSION=$(cat "${INSTALL_DIR:?INSTALL_DIR is not set: re-run Step 2 and substitute the printed path}/VERSION" 2>/dev/null || echo "unknown")
echo "OLD_VERSION=$OLD_VERSION"
```

### 步骤 4:升级

使用步骤 2 中检测到的安装类型和目录：

**对于 git 安装**（global-git、local-git）：

先尝试 fast-forward —— 这与会话更新的 auto-upgrade 策略相同。`--autostash` 会把本地编辑带到 pull 上；render-footprint dirt 因为可重新生成且会污染 stash，所以先丢弃掉：
```bash
cd -- "${INSTALL_DIR:?INSTALL_DIR is not set: re-run Step 2 and substitute the printed path}" || exit 1
{ [ -f VERSION ] && [ -f setup ] && [ -f bin/gstack-config ] && _p=$(git rev-parse --show-prefix 2>/dev/null) && [ -z "$_p" ] && [ "$(git rev-parse --is-inside-work-tree 2>/dev/null)" = true ]; } || { echo "ERROR: $INSTALL_DIR is not a gstack checkout; nothing was changed. Re-run Step 2." >&2; exit 1; }
# 丢弃 render-footprint dirt：开启 gbrain 的较老安装会把生成的 SKILL.md / sections 文件直接 IN PLACE 渲染。它们是可重新生成的（setup 会重新渲染到 ~/.gstack/render），所以丢弃是无损的。
git checkout -- 'SKILL.md' '*/SKILL.md' '*/sections/*.md' 2>/dev/null || true
git fetch origin
# 与 auto-updater 相同的预提交检查：即将发布的版本的 Bun 下限。
HOLD=$(. bin/gstack-bun-version.sh 2>/dev/null && gstack_bun_incoming_hold . origin/main) && { echo "BUN_TOO_OLD: $HOLD; nothing was changed" >&2; exit 1; }
PRE_UPGRADE_COMMIT=$(git rev-parse HEAD)
echo "PRE_UPGRADE_COMMIT=$PRE_UPGRADE_COMMIT"
if git pull --ff-only --autostash origin main; then
  if ./setup --refresh-registered; then echo "FF_OK"; else echo "SETUP_FAILED: git update succeeded; stop and inspect setup output (previous commit: $PRE_UPGRADE_COMMIT)" >&2; exit 1; fi
else
  echo "FF_REFUSED"
fi
```

setup 的 `--refresh-registered` 会刷新此 checkout 注册过的所有安装（每个主机、全局和本项目的），先打印数据源，然后以一行一安装的 `Upgrade summary` 收尾。把这些行原样转达。`failed` 行意味着该主机保留了之前的安装；给出它的重试命令，不要说"每个主机都已刷新"。`skipped` 行属于另一个 checkout 或项目；把它的命令原样转交，不要运行它。

如果输出以 `FF_OK` 结尾，升级就完成了 —— 完全跳过下面的 fallback。
遇到 `BUN_TOO_OLD`，STOP:告诉用户运行 `bun upgrade`，然后再次运行 /gstack-upgrade。
遇到 `SETUP_FAILED`，STOP；保留用户改动，并报告恢复所用的 commit。git 路径上没有 `.bak`。不要仅仅因为 setup 失败就进入 divergence fallback。只在 `FF_REFUSED` 时才进入，并且要先检查 pull 报错；网络/鉴权失败应该停下修复，而不是 reset。

**Fallback（ff-only 被拒 —— 本地 commit 或 divergence）。** `git reset --hard` 会破坏东西：一个有未推送本地 commit 的干净工作树，仍会丢失那些 commit。需要把守门加上：

1. 在 `$INSTALL_DIR` 中运行 `git status --porcelain` 和 `git rev-list origin/main..HEAD --oneline`。
2. 如果两者皆为空，reset 是可证明安全的 —— 直接运行下面的 fallback block 而不问。
3. 否则通过 AskUserQuestion 来询问（这是一个单向操作 —— 具破坏性），列出每个将要被丢弃的具体内容：每一个脏文件，以及每一个未推送的 commit（含 hash + subject）。选项：**A)** 丢弃它们并升级（reset）—— 需要明确指字母 **A** 才执行；**B)** 中止升级，以便用户先抢救自己的工作（当存在本地 commit 时推荐）。绝对不要在含糊的回答上继续。

```bash
cd -- "${INSTALL_DIR:?INSTALL_DIR is not set: re-run Step 2 and substitute the printed path}" || exit 1
{ [ -f VERSION ] && [ -f setup ] && [ -f bin/gstack-config ] && _p=$(git rev-parse --show-prefix 2>/dev/null) && [ -z "$_p" ] && [ "$(git rev-parse --is-inside-work-tree 2>/dev/null)" = true ]; } || { echo "ERROR: $INSTALL_DIR is not a gstack checkout; nothing was changed. Re-run Step 2." >&2; exit 1; }
STASH_OUTPUT=$(git stash 2>&1)
git reset --hard origin/main
./setup --refresh-registered
```
如果 `$STASH_OUTPUT` 包含 "Saved working directory"，警告用户："Note: local changes were stashed (any modified generated SKILL.md/sections files were discarded first — they regenerate on setup). Run `git stash pop` in the skill directory to restore your own changes."

**对于 vendored 安装**（vendored、vendored-global）：
```bash
(cd -- "${INSTALL_DIR:?INSTALL_DIR is not set: re-run Step 2 and substitute the printed path}" && [ -f VERSION ] && [ -f setup ] && [ -f bin/gstack-config ]) || { echo "ERROR: INSTALL_DIR=${INSTALL_DIR:-} is not a gstack install; nothing was changed. Re-run Step 2." >&2; exit 1; }
# 上一次崩溃升级遗留的过期 .bak 会让下面的 mv 把活安装套到它的内部，使失败恢复分支"恢复"的是过期备份。它也可能恰好是那次崩溃运行唯一还能用的副本——要中止并让用户去检查，绝不能默默删除它。
[ -e "$INSTALL_DIR.bak" ] && { echo "ERROR: stale backup exists at $INSTALL_DIR.bak (from a previous failed upgrade?) — inspect it, salvage/remove it, then re-run." >&2; exit 1; }
TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/gstack-upgrade.XXXXXX") || { echo "ERROR: mktemp failed — aborting upgrade (install untouched)." >&2; exit 1; }
git clone --depth 1 https://github.com/garrytan/gstack.git "$TMP_DIR/gstack" || { echo "ERROR: clone failed — aborting upgrade (install untouched)." >&2; rm -rf "$TMP_DIR"; exit 1; }
mv "$INSTALL_DIR" "$INSTALL_DIR.bak" || { rm -rf "$TMP_DIR"; exit 1; }
if mv "$TMP_DIR/gstack" "$INSTALL_DIR"; then
  if (cd "$INSTALL_DIR" && ./setup --refresh-registered); then
    rm -rf "$INSTALL_DIR.bak" "$TMP_DIR"
  else
    rm -rf "$INSTALL_DIR"
    mv "$INSTALL_DIR.bak" "$INSTALL_DIR" || { echo "ERROR: restore failed; backup retained." >&2; exit 1; }
    rm -rf "$TMP_DIR"
    echo "ERROR: setup failed; previous install restored." >&2
    exit 1
  fi
else
  mv "$INSTALL_DIR.bak" "$INSTALL_DIR"
  echo "ERROR: swap failed — previous install restored; upgrade aborted." >&2
  rm -rf "$TMP_DIR"
  exit 1
fi
```

### 步骤 4.5:处理本地 vendored 副本

使用步骤 2 中得到的安装目录。检查是否还存在本地 vendored 副本，以及是否启用了 team mode：

```bash
_RESOLVED_PRIMARY=$(cd -- "${INSTALL_DIR:?INSTALL_DIR is not set: re-run Step 2 and substitute the printed path}" && pwd -P) || exit 1
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
LOCAL_GSTACK=""
if [ -n "$_ROOT" ] && [ -d "$_ROOT/.claude/skills/gstack" ]; then
  _RESOLVED_LOCAL=$(cd "$_ROOT/.claude/skills/gstack" && pwd -P)
  if [ "$_RESOLVED_LOCAL" != "$_RESOLVED_PRIMARY" ]; then
    LOCAL_GSTACK="$_ROOT/.claude/skills/gstack"
  fi
fi
_TEAM_MODE=$(~/.claude/skills/gstack/bin/gstack-config get team_mode 2>/dev/null || echo "false")
echo "LOCAL_GSTACK=$LOCAL_GSTACK"
echo "TEAM_MODE=$_TEAM_MODE"
```

**如果 `LOCAL_GSTACK` 非空 AND `TEAM_MODE` 为 `true`:** 删除 vendored 副本。Team mode 把全局安装当作唯一事实来源。

```bash
(cd -- "${LOCAL_GSTACK:?LOCAL_GSTACK is not set: re-run the detection block above and substitute the printed path}" && [ -f VERSION ] && [ -f setup ] && [ -f bin/gstack-config ]) || { echo "ERROR: LOCAL_GSTACK=${LOCAL_GSTACK:-} is not a gstack copy; nothing was changed." >&2; exit 1; }
_ROOT=${LOCAL_GSTACK%/.claude/skills/gstack}
{ [ "$_ROOT" != "$LOCAL_GSTACK" ] && cd -- "${_ROOT:?}" && _p=$(git rev-parse --show-prefix 2>/dev/null) && [ -z "$_p" ] && [ "$(git rev-parse --is-inside-work-tree 2>/dev/null)" = true ]; } || { echo "ERROR: $LOCAL_GSTACK is not the vendored copy at the root of a git repository; nothing was changed." >&2; exit 1; }
git rm -r --cached .claude/skills/gstack/ 2>/dev/null || true
if ! grep -qF '.claude/skills/gstack/' .gitignore 2>/dev/null; then
  echo '.claude/skills/gstack/' >> .gitignore
fi
rm -rf "$LOCAL_GSTACK"
```
告诉用户："Removed vendored copy at `$LOCAL_GSTACK` (team mode active — global install is the source of truth). Commit the `.gitignore` change when ready."

**如果 `LOCAL_GSTACK` 非空 AND `TEAM_MODE` **不是** `true`:** 把刚升级完的主安装复制过来,以更新本地副本(方法与 README 中 vendored 安装一致):
```bash
(cd -- "${INSTALL_DIR:?INSTALL_DIR is not set: re-run Step 2 and substitute the printed path}" && [ -f VERSION ] && [ -f setup ] && [ -f bin/gstack-config ]) || { echo "ERROR: INSTALL_DIR=${INSTALL_DIR:-} is not a gstack install; nothing was changed." >&2; exit 1; }
(cd -- "${LOCAL_GSTACK:?LOCAL_GSTACK is not set: re-run the detection block above and substitute the printed path}" && [ -f VERSION ] && [ -f setup ] && [ -f bin/gstack-config ]) || { echo "ERROR: LOCAL_GSTACK=${LOCAL_GSTACK:-} is not a gstack copy; nothing was changed." >&2; exit 1; }
[ -e "$LOCAL_GSTACK.bak" ] && { echo "ERROR: stale vendored backup; inspect it before retrying." >&2; exit 1; }
mv "$LOCAL_GSTACK" "$LOCAL_GSTACK.bak" || exit 1
if cp -Rf "$INSTALL_DIR" "$LOCAL_GSTACK" && rm -rf "$LOCAL_GSTACK/.git" && (cd "$LOCAL_GSTACK" && ./setup --refresh-registered); then
  rm -rf "$LOCAL_GSTACK.bak"
  echo "LOCAL_SYNC_OK"
else
  rm -rf "$LOCAL_GSTACK"
  mv "$LOCAL_GSTACK.bak" "$LOCAL_GSTACK" || { echo "ERROR: restore failed; backup retained." >&2; exit 1; }
  echo "ERROR: sync failed; previous vendored copy restored." >&2
  exit 1
fi
```
只有拿到 `LOCAL_SYNC_OK` 时,告诉用户:"Also updated vendored copy at `$LOCAL_GSTACK` — commit `.claude/skills/gstack/` when you're ready." 否则停下来报告恢复结果,不要继续迁移、也不要宣告成功。

### 步骤 4.75:运行版本迁移

`./setup` 完成后,运行从旧版本到新版本之间的所有迁移脚本。迁移处理 `./setup` 单凭自身无法涵盖的状态修正（过期配置、孤儿文件、目录结构变更）。

```bash
MIGRATIONS_DIR="${INSTALL_DIR:?INSTALL_DIR is not set: re-run Step 2 and substitute the printed path}/gstack-upgrade/migrations"
: "${OLD_VERSION:?OLD_VERSION is not set: substitute the value Step 3 printed}"
if [ -d "$MIGRATIONS_DIR" ]; then
  for migration in $(find "$MIGRATIONS_DIR" -maxdepth 1 -name 'v*.sh' -type f 2>/dev/null | sort -V); do
    # 从文件名提取版本：v0.15.2.0.sh → 0.15.2.0
    m_ver="$(basename "$migration" .sh | sed 's/^v//')"
    # 仅当此迁移版本比旧版本更新时才运行
    # （对段数一致的点分版本，简单字符串比较即可）
    if [ "$OLD_VERSION" != "unknown" ] && [ "$(printf '%s\n%s' "$OLD_VERSION" "$m_ver" | sort -V | head -1)" = "$OLD_VERSION" ] && [ "$OLD_VERSION" != "$m_ver" ]; then
      echo "Running migration $m_ver..."
      # GSTACK_INSTALL_DIR:对安装本身（不仅仅是 ~/.gstack 状态）进行清理的迁移在未设置时默认为 ~/.claude/skills/gstack——
      # 不提供这个的话，仓库本地的安装会静默 no-op。
      GSTACK_INSTALL_DIR="$INSTALL_DIR" bash "$migration" || echo "  Warning: migration $m_ver had errors (non-fatal)"
    fi
  done
fi
```

迁移是 `gstack-upgrade/migrations/` 中的幂等 bash 脚本。每个脚本以 `v{VERSION}.sh` 命名,只有从更早的版本升级时才会运行。如何新增迁移脚本，请参见 CONTRIBUTING.md。

### 步骤 4.8:停止过期的守护进程（无条件执行）

升级前启动的 browse 守护进程会一直用旧二进制,直到被停止——它能在 `git reset --hard` 和 `./setup` 中存活,因为运行中的进程持有旧可执行文件。始终执行本步骤,使用步骤 2 中检测到的安装目录。

```bash
INSTALL_DIR_PLACEHOLDER="<install dir from Step 2>"
NEW_HASH=$(cat "$INSTALL_DIR_PLACEHOLDER/browse/dist/.version" 2>/dev/null || echo "")
_STATE_FILE="${BROWSE_STATE_FILE:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/browse.json}"
if [ -z "$NEW_HASH" ] || [ ! -f "$_STATE_FILE" ]; then
  echo "DAEMON_CHECK=none (no state file or no fresh build hash)"
else
  DAEMON_PID=$(sed -n 's/.*"pid"[[:space:]]*:[[:space:]]*\([0-9][0-9]*\).*/\1/p' "$_STATE_FILE" | head -1)
  DAEMON_PORT=$(sed -n 's/.*"port"[[:space:]]*:[[:space:]]*\([0-9][0-9]*\).*/\1/p' "$_STATE_FILE" | head -1)
  OLD_HASH=$(sed -n 's/.*"binaryVersion"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$_STATE_FILE" | head -1)
  if [ -z "$DAEMON_PID" ] || ! kill -0 "$DAEMON_PID" 2>/dev/null; then
    echo "DAEMON_CHECK=dead (no live daemon to stop)"
  elif [ "$OLD_HASH" = "$NEW_HASH" ]; then
    echo "DAEMON_CHECK=current (daemon already runs the new binary)"
  elif curl -fsS --max-time 2 "http://127.0.0.1:$DAEMON_PORT/health" 2>/dev/null | grep -q '"status":"healthy"'; then
    echo "DAEMON_CHECK=stale-responsive pid=$DAEMON_PID hash=${OLD_HASH:-unknown} -> $NEW_HASH"
    "$INSTALL_DIR_PLACEHOLDER/browse/dist/browse" stop && echo "DAEMON_STOPPED=yes"
  else
    echo "DAEMON_CHECK=stale-busy pid=$DAEMON_PID hash=${OLD_HASH:-unknown} -> $NEW_HASH"
  fi
fi
```

运行前,把 `<install dir from Step 2>` 替换成实际的安装目录。解读 `DAEMON_CHECK` 的结果:

1. **`stale-responsive` + `DAEMON_STOPPED=yes`:** 告诉用户 "Stopped the old browse daemon (binary {OLD_HASH} → {NEW_HASH}). The next browse command starts a fresh daemon on the new binary."
2. **`stale-busy`:** 该守护进程跑的是旧二进制但还在忙——升级时 DEFER 给它，绝不在忙时 kill。告诉用户："A browse daemon is still running the pre-upgrade binary ({OLD_HASH} → {NEW_HASH}) but is busy right now. When it finishes, stop it with `browse stop` — or force it immediately with `browse --force-restart stop` (loses that session's tabs/cookies/logins)."
3. **`none` / `dead` / `current`:** 无需操作——什么都不说。

### 步骤 5:写入标记 + 清空缓存

```bash
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
mkdir -p "$GSTACK_STATE_ROOT"
echo "$OLD_VERSION" > "$GSTACK_STATE_ROOT"/just-upgraded-from
rm -f "$GSTACK_STATE_ROOT"/last-update-check
rm -f "$GSTACK_STATE_ROOT"/update-snoozed
```

### 步骤 6:展示 What's New

读取 `$INSTALL_DIR/CHANGELOG.md`。找出旧版本到新版本之间的所有版本条目。归纳成 5-7 条按主题分组的要点。不要让人眼花缭乱——重点关注面向用户的变更。内部重构如非重要就跳过。

格式:
```
gstack v{new} — upgraded from v{old}!

What's new:
- [bullet 1]
- [bullet 2]
- ...

Happy shipping!
```

### 步骤 7:继续

展示完 What's New 之后，继续执行用户原本调用的那个 skill。升级就完成了，无需进一步操作。

---

## 独立使用

作为 `/gstack-upgrade` 直接调用时（不是从 preamble 调用）：

1. 强制进行新的 update check（绕过缓存）：
```bash
~/.claude/skills/gstack/bin/gstack-update-check --force 2>/dev/null || \
.claude/skills/gstack/bin/gstack-update-check --force 2>/dev/null || true
```
用输出判断是否有可用升级。

2. 如果 `UPGRADE_AVAILABLE <old> <new>`:按上面的步骤 2-6 走。

   如果是 `CHECK_FAILED ...`:远程版本读不到,所以升级状态未知。显示该行（它会给出失败的 URL），告诉用户检查失败；永远不要报告 "already on the latest version"。停下。
