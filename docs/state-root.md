# gstack 将其状态保存在哪里

gstack 将其记住的所有内容 — 配置、分析、会话、项目、learnings、评审日志、出口账本、信任策略存储、hook 错误日志 — 保存在一个目录下,即 **state root**。默认情况下是 `~/.gstack`。设置 `GSTACK_HOME` 将其放在其他位置。

一条规则选择根,每个 gstack 脚本、hook 和 skill 都使用它。该规则存在于两个由奇偶校验测试保持相同的所有者文件中:`bin/gstack-state-root.sh`(bash)和 `lib/state-root.ts`(TypeScript)。

## 根的选择方式

此列表中具有非空值的第一个变量胜出。空值计为未设置。

| 顺序 | 来源 | 何时使用它 |
|------:|--------|----------------|
| 1 | `GSTACK_STATE_ROOT` | `gstack-paths` 自己的输出。它也作为输入被尊重,因此在同一环境中重新读取它返回相同的根。不要手动设置它。 |
| 2 | `GSTACK_HOME` | **要设置的唯一变量** 当你希望你的状态在 `~/.gstack` 之外的位置时。 |
| 3 | `GSTACK_STATE_DIR` | 用于兼容性的传统别名。 |
| 4 | `CLAUDE_PLUGIN_DATA` | 仅当 `CLAUDE_PLUGIN_ROOT` 包含 `gstack`(任何大小写)时,以便另一个插件的数据目录永远不会捕获 gstack 状态。 |
| 5 | `$HOME/.gstack` | 默认。在 Windows shell 上,当 `HOME` 未设置时,`USERPROFILE` 替代。 |
| 6 | `.gstack` | 当根本没有主目录时的最后手段(某些容器)。 |

## 查看正在使用的根

```bash
~/.claude/skills/gstack/bin/gstack-paths --explain
```

使用 `GSTACK_HOME` 设置而 `~/.gstack` 仍保存较旧状态时的真实输出:

```
state root: /home/you/work-state (selected by GSTACK_HOME)
chain (first non-empty wins):
  GSTACK_STATE_ROOT    unset
  GSTACK_HOME          /home/you/work-state  selected
  GSTACK_STATE_DIR     unset
  CLAUDE_PLUGIN_DATA   unset
  default              /home/you/.gstack  ignored
default root /home/you/.gstack also holds gstack state: yes
merged privacy keys (most restrictive value across roots wins):
  telemetry:        off (from /home/you/.gstack/config.yaml)
  memorable_recall: not set (default applies)
  codex_reviews:    not set (default applies)
  update_check:     not set (default applies)
docs: https://github.com/garrytan/gstack/blob/main/docs/state-root.md
```

`--explain` 仅读取;它不写入任何内容并退出 0。当 `GSTACK_STATE_ROOT`、`GSTACK_HOME` 和 `GSTACK_STATE_DIR` 命名不同的目录时,`gstack-config list` 还会打印一行注释。

## 移动你的状态

gstack 从不为你移动状态。要重新定位它:

```bash
cp -a ~/.gstack /new/place/gstack-state
export GSTACK_HOME=/new/place/gstack-state   # add this to your shell profile
~/.claude/skills/gstack/bin/gstack-paths --explain   # confirm: selected by GSTACK_HOME
```

在确认新根有效后,保留或删除旧的 `~/.gstack`。在删除它之前,其隐私设置仍然有效(下一节)。

## 移动根时隐私设置永远不会变得不那么严格

四个配置键是选择退出:`telemetry`(off < anonymous < community)、`memorable_recall`(off < on)、`codex_reviews`(disabled < enabled) 和 `update_check`(false < true)。gstack 从解析的根**和** `~/.gstack` 读取每个,并使用最严格的值。因此,在 `~/.gstack` 中关闭遥测一次会在你设置 `GSTACK_HOME` 后保持关闭。其他每个键,包括 `proactive` 和 `founder_resources`,仅读取解析的根。

当 `gstack-config set` 写入合并的键但另一个根仍保存更严格的值时,它会告诉你哪个根覆盖它并打印更改该根的命令,例如:

```bash
GSTACK_STATE_ROOT='/home/you/.gstack' ~/.claude/skills/gstack/bin/gstack-config set codex_reviews enabled
```

`gstack-config list` 显示合并键的获胜根,例如 `telemetry: off (set, /home/you/.gstack)`。

信任策略拒绝 tier 以相同的方式合并:`~/.gstack/gbrain-repo-policy.json` 中的 `deny` 或 `read-only` 条目在解析的根在其他地方时仍然适用。其他任何内容都不会写入其他根。

## 项目桶

每个项目状态(检查点、learnings、时间线、问题偏好)存在于 `projects/<slug>/` 中。`bin/gstack-slug` 从 git `origin` 远程选择 slug:

- 2 段远程(`github.com/owner/repo`)归档在 `owner-repo` 下。
- 具有 3 个或更多路径段的远程(GitLab 嵌套组,例如 `gitlab.com/customer-a/product/repo`、Azure DevOps)归档在 `product-repo-<16 hex>` 下。hex 是规范远程的摘要,因此 `customer-a/product/repo` 和 `customer-b/product/repo` 获得单独的桶,并且一个仓库的 ssh、https 和带端口的 ssh 拼写共享一个。
- 没有远程时,使用项目根的目录名称。

gstack 在目录中解析的第一个 slug 粘住(它在 `slug-cache/` 下被缓存),因此项目在稍后添加远程时保留其桶。

**从使用一个桶进行嵌套组的版本升级。** 较早的版本将每个 `<group>/product/repo` 归档在 `product-repo` 下。该桶保留在其位置;gstack 既不移动也不删除它。gstack 第一次解析新 slug 时,它会打印一个通知,命名两个桶,而 `/context-restore` 在旧桶仍保存检查点时报告它。要将较早的文件带过来,请在项目中运行:

```bash
~/.claude/skills/gstack/bin/gstack-slug --adopt-legacy
```

它列出旧桶中的每个文件。检查点标记为匹配此项目、另一个项目(永远不复制)或未知(在检查点记录其项目之前保存)。没有项目标识的文件(问题偏好、learnings、时间线)可能属于共享该桶的任何项目,因此它们会列出供你决定。在终端上,它会询问每个文件;否则使用 `gstack-slug --adopt-legacy --copy <path> [<path>...]` 复制选定的文件。没有你的确认,不会复制任何内容,并且新桶中的任何内容都不会被覆盖。

**Pin slug。** 设置 `GSTACK_PROJECT_SLUG=<name>` 以自行选择桶,例如在项目的 `.claude/settings.local.json`(`"env": {"GSTACK_PROJECT_SLUG": "<name>"}`)中。当 ssh 主机别名(`git@work-gitlab:group/sub/repo`)提供与 https 远程不同的主机,或者对于应该是其自己项目的 vendored 子仓库时使用它。pin 按运行应用,从不缓存。

## 临时机器

Capy 云机器、CI runner 或一次性 VM 在运行结束时丢失其磁盘,以及 gstack 在默认根下写入的每个 learnings、决策、评审日志行和计时记录。gstack 不会从路径猜测这一点(tmpfs 和主目录看起来相似);主机通过一个环境标记这样说:

```
GSTACK_EPHEMERAL=1
```

标记更改的内容:

- `gstack-doctor` 和 `gstack-doctor --check` 将 `state root` 行报告为 `durable=no`(仪表板发出警告,检查仍然通过:临时根是事实,而不是损坏的安装)。
- Unattended 会话打印 `learnings: skipped (state root is ephemeral; set GSTACK_STATE_ROOT)` 而不是默默地写入将消失的磁盘。

持久根是选择加入的:所有者将 `GSTACK_STATE_ROOT`(或 `GSTACK_HOME`)设置为寿命超过机器的目录。安装程序从不为你选择一个。在 Capy 上,驱动器受 Capy 自己的驱动器策略约束,该策略不包括内存和验证证据,因此 `bin/gstack-capy-install` 仅打印选项且从不写入那里。

子 shell 的 `export` 无法配置父级稍后的命令(`lib/state-root.ts` 读取每个进程自己的环境),因此安装程序以父级提供给每个它启动的 gstack 命令的一行机器可读的行结束:

```
GSTACK_LAUNCH: GSTACK_SESSION_KIND=unattended GSTACK_STATE_ROOT=<path> GSTACK_EPHEMERAL=1
```

为每个命令加上这些赋值的前缀(或在运行它们的 shell 中导出它们)。参见 [docs/capy-install.md](capy-install.md)。

## 卸载

`gstack-uninstall` 仅在 `~/.gstack` 处删除状态。

- 当 `~/.gstack` 解析 — 遵循符号链接 — 为 `/`、你的主目录或其父目录之一、gstack checkout、当前 git 仓库或其中任何一个的父目录时,它首先拒绝(退出 2)。该消息命名原因和 `fix:` 行。
- 当解析的根在其他地方时(例如,你设置了 `GSTACK_HOME`),卸载保留它,并打印 `left in place: <path> (selected by <VAR>=<value>)` 以及在你检查后要运行的确切 `rm -rf -- '<path>'` 命令。
- `--keep-state` 从不删除任何状态。`--force` 仅跳过提示。

## 如果 gstack-paths 缺失

Skill 块使用以下方式解析根

```bash
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
```

(没有 `eval`:worktree-isolated Claude Code 会话拒绝它。)

后半部分在解析器损坏时停止该块而不是在 `/` 下写入。如果你看到 `gstack-paths failed` 或 `cannot resolve the gstack state root`,请重新安装:在 gstack checkout 中运行 `./setup`,或 `/gstack-upgrade`。

## Claude Code 插件安装

gstack 今天不发布官方 Claude Code 插件分发(主分支上从未有过 `.claude-plugin` 清单;一些 fork 会发布一个),并且遥测不报告插件模式安装。当 gstack 确实作为插件运行时,`CLAUDE_PLUGIN_ROOT` 命名 gstack,状态存在于 `CLAUDE_PLUGIN_DATA` 中,而插件会话外的终端运行使用 `~/.gstack`。这些是两个不同的根;在每个环境中运行 `gstack-paths --explain` 以查看哪个根是活动的。跨环境支持(从 `~/.gstack` 到插件根的指针)被推迟,直到存在插件分发。

## 对于贡献者

- Bash:可执行文件在它们旁边的 `bin/gstack-state-root.sh` 并调用 `gstack_state_root_select`(设置 `$_gstack_sr_root`),或使用上面的 guard eval gstack-paths。Hooks 必须 source 双胞胎;永远不要从 hook spawn gstack-paths。
- TypeScript:`lib/state-root.ts` 中的 `resolveStateRoot()` 和 `readConfigKey()`。
- `test/state-root-ratchet.test.ts` 在手工制作的链、可执行文件 `~/.gstack` 在模板 bash 块中、未受保护的 gstack-paths eval 或散文中的 `export GSTACK_STATE_ROOT` 上失败。其允许列表(`test/state-root-ratchet.allowlist.json`)需要每个条目的原因。
