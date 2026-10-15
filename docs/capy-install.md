# 在 Capy 云机器上安装 gstack

三个层,每个层由 gstack 拥有:

| 层 | 位置 | 工作 |
|---|---|---|
| Bootstrapper | `capy/skill/gstack/scripts/install.sh`(复制一次到所有者的 Drive skill 文件夹;已冻结) | 缺失时克隆;获取 origin 的默认分支并从 `FETCH_HEAD:capy/support.env` 读取支持的最低版本;在安装模式下快进低于它的干净、未固定的 checkout;执行完整的 wrapper |
| 完整 wrapper | `bin/gstack-capy-session`(随 checkout 移动) | 将会话开始映射到安装程序:`--check` 认证,否则安装或修复 |
| Installer | `bin/gstack-capy-install` | Bun、两个主机配置文件、doctor、安装 verdict、启动行、尾部 |

供应商式 skill(`capy/skill/gstack/SKILL.md`)告诉 Capy 智能体如何使用它们。当被询问时,Drive 文件夹是一个镜像:所有者将 `capy/skill/gstack/` 复制到那里;没有任何内容会自动写入 Drive。

直接使用,无需 skill:

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.local/share/gstack
~/.local/share/gstack/bin/gstack-capy-install --for autoplan
```

## 会话开始

1. `bash <skill>/scripts/install.sh --check --for <skill>`(当存在 checkout 时不到一秒)。它仅获取 refs 并且永不移动 checkout。如果没有 checkout,它会打印 `gstack: fail not-installed — ...`,并且当它不运行任何 gstack 工作流时,会话记录该行并继续;在第一个工作流之前立即安装。
2. 当尾部是 `gstack: fail revision ...` 或 `gstack: fail runtime ...` 并且存在 checkout 时,运行不带 `--check` 的相同命令。低于支持的最低版本时,远程默认分支上的干净 checkout 被快进(`gstack-capy-skill: upgraded <old> -> <new>`),重新安装并作为一个事务重新检查,位于 `.gstack-capy-install.lock` 下(bootstrapper 持有它并使用 `GSTACK_CAPY_LOCK_HELD=1` 移交)。固定(`--revision`、分离的 HEAD、另一个分支)、脏或偏离的 checkout 永远不会移动;尾部命名原因和确切的命令([REVISION_BELOW_MINIMUM](troubleshooting.md#revision-below-minimum))。`GSTACK_CAPY_AUTO_UPGRADE=0` 禁用快进。
3. 当获取失败(离线)时,bootstrapper 说 `remote minimum unknown (fetch failed)` 并且仅检查本地安装;它永远不会声称为最低版本。

升级目标始终是默认分支的提示:该处的每个提交都是通过 CI 的标记发布。恢复向前运行,从不向后:在快进之后中断的安装使尾部 `fail`,下一个安装或 `bin/gstack-upgrade` 运行完成挂起的安装阶段(在快进之后永远不会跳过 setup)或拒绝并命名原因。检出较旧的提交不是受支持的回滚;它不会撤消已经运行的 setup 迁移。

在最低版本之间,`bin/gstack-upgrade --check` 说 `current <v>` 或 `behind <installed> -> <remote> (<n> commits)`,而 `bin/gstack-upgrade` 快进并重新安装(在 Capy 安装上是 `bin/gstack-capy-install`,否则是 `./setup --refresh-registered`)。它拒绝脏、偏离或固定的 checkout 以及 [UPGRADE_REFUSED](troubleshooting.md#upgrade-refused),并且永远不会存储或重置。

## 安装程序做什么

1. 在锁定下重用它所在的 checkout(或克隆到 `--root <dir>`)。它永远不会重置或拉取;`--revision <ref>` 固定新克隆,另一个修订的现有 checkout 与尾部注释重用(`(requested <ref>; run <checkout>/bin/gstack-upgrade to move)`)。
2. 预检:`git`、`curl`、`node`、`jq`,然后是 Bun。已在 `$BUN_INSTALL/bin`(默认 `~/.bun/bin`)中受支持的 Bun 被重用。项目的 pins(`--project <path>`,默认为当前目录的 git toplevel)使用 doctor 自己的解析器(`bin/gstack-runtime-pins.ts`:engines、`packageManager`、`.tool-versions`、`.bun-version` 和 CI `bun-version:` lanes 用于此平台)读取。当 PATH 上的 Bun 都没有也不满足 gstack 最低版本和 pins 时,安装程序会下载一个确切的发布存档,根据 `bin/gstack-bun-version.sh` 验证其 SHA-256,并将其安装到 `$BUN_INSTALL/bin`,旁边的任何不同系统 Bun(没有到共享 bin 目录的符号链接)。该存档是支持的最低版本(1.4.2),或项目的确切 pin 当它高于最低版本且 gstack 为其携带经验证的存档时(`GSTACK_BUN_VERIFIED`:1.4.2、1.4.3),因此 CI 固定 `bun-version: 1.4.3` 的仓库获得 1.4.3。使用确切的 pin,仅该版本满足它。任何其他确切的 pin(低于最低版本、两个不同的 pin,或没有经验证的存档),或 gstack 的最低版本不满足的范围,在 `./setup` 之前以两个约束([RUNTIME_PIN_CONFLICT](troubleshooting.md#capy-install-runtime-conflict),退出 3)停止。没有 Bun 在或高于安全底限,验证的先安装,以便可以读取 pins。
3. `./setup --host claude --no-prefix`,然后 `./setup --host codex --model gpt-6-astra`,两者都带 `--no-browser`。当此确切 HEAD 使用相同的主机、model 和浏览器标志设置时跳过(git dir 在成功设置后记录),并且 `gstack-doctor --check` 通过;温暖的重新运行大约需要一秒。
4. `gstack-doctor --check` 用于请求的工作流(`--for <skill>`、`--only <skill>` 或 `--require <id,id>`;默认 `claude,codex,patch`),使用启动 PATH 运行,因此尾部认证 Bun 工作流运行。
5. 安装 verdict,以原子方式写入 `<state root>/install-verdicts/<sha256 of the checkout realpath>.json`:checkout、HEAD、其认证的工作流和必需组件、是否跳过 pins 作为仅计划、尾部、远程最低版本、启动 PATH 和时间。

`--check` 仅运行步骤 4 和 5(无克隆、Bun 安装或 setup)。

## grep 的一行

最后一行是尾部:

```
gstack: ok 1.91.96.0 for=autoplan
gstack: fail runtime,pins
```

仅当 `gstack: ok` 时退出 0。直接读取退出状态和尾部:

```bash
out=$(bash <skill>/scripts/install.sh --check --for autoplan 2>&1); rc=$?
printf '%s\n' "$out" | grep '^gstack: '
[ "$rc" -eq 0 ]
```

永远不要单独通过 `tail`、`head` 或 `grep` 管道 `--check`(或安装程序):管道的退出状态替换 doctor 的状态,而静默失败是该安装程序存在的原因。每个 `FAIL <id>` 行都携带 `fix:` 子句和原因代码及其 [troubleshooting 锚点](troubleshooting.md#install-check-gstack-doctor---check-and-the-capy-installer)。父智能体使用此 grep 确认每个子智能体机器;一个机器的安装永远不会覆盖另一个,而子智能体的第一份报告携带其自己的 `gstack:` 行。

`SKIP pins (planning-only; repo pins differ: ..., launch bun <v>)` 不是失败:计划工作流(`--for autoplan`、计划评审、`office-hours`)从不运行目标仓库的测试。这样做(`review`、`ship`、`qa`)和 `--require` 保持 `pins` 是必需的。

当 lane 向另一个线程报告(交接、`gstack-lane-check` 结果、协调器合约)时,在消息中存储和使用完整的线程 ID(`jam_…`),而不是短的显示代码:代码是应用程序呈现的标签,id 是另一个智能体可以寻址的内容以及证据 bundle 记录的内容。`UBI_OWNER` 和 lane 所有权记录采用相同的完整 ID。

## 浏览器

默认安装会打印 `SKIP browser (lazy; gstack-browser-ensure installs on first use)` 并且不下载任何内容。第一次浏览器 skill 运行(`/browse`、`/qa`、`/design-review` 等)从其预检调用 `bin/gstack-browser-ensure`,该安装在 setup 的锁下安装 Chromium(约 115 MiB,在 Capy 机器上约 86 秒),并打印 `BROWSER_OK` 或 `BROWSER_UNAVAILABLE <reason>`。`--with-browser` 在安装期间安装它,然后安装程序分别打印仅计划的安装时间和浏览器引导时间。

## 启动行

每次成功的运行(包括 `--check`)在尾部之前打印:

```
GSTACK_LAUNCH: PATH=<dir>:$PATH GSTACK_SESSION_KIND=unattended GSTACK_STATE_ROOT=<path> GSTACK_EPHEMERAL=1 GSTACK_CAPY=1 GSTACK_INSTALL_VERDICT=<file>
GSTACK_LAUNCH_PATH: <dir>
```

`PATH=<dir>:$PATH` 和 `GSTACK_LAUNCH_PATH` 仅在认证的 Bun 不在 PATH 上时出现(Capy 映像的 `/usr/local/bin/bun` 是 1.3.14;57 个 gstack 命令以 `#!/usr/bin/env bun` 开头)。子 shell 的 `export` 无法配置父级稍后的命令,因此父级将这些分配提供给每个它启动的 gstack 命令(为命令添加前缀,或在运行它的 shell 中导出它们)。没有它们的新 shell 继续报告它真正会运行的 Bun:`FAIL runtime — bun 1.3.14 at /usr/local/bin/bun ...; verified bun 1.4.2 found at ~/.bun/bin (not on PATH); fix: export PATH="$HOME/.bun/bin:$PATH" or prefix commands with the GSTACK_LAUNCH line`。

`GSTACK_EPHEMERAL=1` 告诉 gstack 磁盘随机器消失,因此 doctor 的 `state root` 行读取 `durable=no`,而 unattended 运行说 `learnings: skipped` 而不是默默地写入。持久根是选择加入的:将 `GSTACK_STATE_ROOT` 设置为寿命超过机器的目录。安装程序永远不会写入 Capy 驱动器。参见 [docs/state-root.md](state-root.md#ephemeral-machines)。

`GSTACK_CAPY=1` 打开安装准入。任何主机上的 unattended 会话都拒绝在低于支持的最低版本的 Bun 上启动工作流;使用 `GSTACK_CAPY=1`,unattended 或 `GSTACK_SESSION_KIND=spawned` 会话也需要该 checkout 的通过 verdict、其当前 HEAD 和至少与启动的工作流一样宽的工作流。拒绝打印 `INSTALL: fail ...` 和 `GSTACK_RESULT: skill=<s> status=refused run=-` 并退出 3([INSTALL_UNVERIFIED](troubleshooting.md#install-unverified))。环境生成的 spawned 会话(OpenClaw)仅警告。`GSTACK_ALLOW_UNSUPPORTED_RUNTIME=1` 仍然接受并这样说。

## 烘焙到 Capy 快照中

从不运行工作流的机器不应支付安装(冷约 30 秒)。在大多数机器上运行 gstack 的项目可以改为烘焙:项目 Setup 的初始化步骤运行

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.local/share/gstack
~/.local/share/gstack/bin/gstack-capy-install --require claude,codex,patch
```

快照捕获 `$HOME`(checkout、`~/.bun/bin`、两个主机配置文件)。然后每个会话仅支付 `--check`。烘焙的修订在低于支持的最低版本时仍会在会话开始时向前移动,并且烘焙的 `~/.bun/bin` 通过 `GSTACK_LAUNCH` 行到达工作流。项目 Setup 由项目中的每个线程共享,因此更改它是项目的决定。

## 提高支持的最低版本

`capy/support.env` 保存 `GSTACK_MIN_REVISION` 和 `GSTACK_CAPY_MIN_WRAPPER`。仅出于安装诚实性或安全修复的原因,在标记时将 `GSTACK_MIN_REVISION` 提高到正在发布的 VERSION:每个低于它的机器在其下一次会话开始时被快进。它永远不会超过 VERSION(`test/capy-skill-wrapper.test.ts`)。当 bootstrapper 本身更改时,在 bootstrapper 中将 `GSTACK_CAPY_MIN_WRAPPER` 与 `GSTACK_CAPY_WRAPPER` 一起提高;较旧的 Drive 副本继续工作并打印所有者操作([CAPY_WRAPPER_STALE](troubleshooting.md#capy-wrapper-stale))。

## 退出代码

| 退出 | 含义 |
|-----:|---------|
| 0 | `gstack: ok` — 每个必需组件通过 |
| 1 | `gstack: fail <ids>` — 必需的组件失败;每行都有修复 |
| 2 | 使用错误 |
| 3 | 预检在 `./setup` 之前停止(`INSTALL_PREFLIGHT_FAILED`、`RUNTIME_PIN_CONFLICT`);未安装任何内容 |
