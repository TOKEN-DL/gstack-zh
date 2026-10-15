---
name: gstack
description: 在 Capy 和 Claude Code 中安装并使用 gstack。在会话开始时加载,以检查本机是否就绪,并用于 gstack 的规划、代码审查、调试、QA、设计、安全、文档和发布工作流。
---

# gstack for Capy and Claude Code

## 会话开始:先检查,只在需要时安装

相对本 SKILL.md 解析 `scripts/install.sh`,而不是工作仓库。它是一个冻结的引导器:它首先打印 `gstack-capy-skill: wrapper <n> (<path>)`,从 gstack 的远端学习所支持的最低版本,并交接给检出自身的包装器(`bin/gstack-capy-session`)。

1. 在会话开始时运行 `bash scripts/install.sh --check`,当你明确知道要使用的工作流时,加上 `--for <skill>`(`--for autoplan`、`--for review`、`--for qa`)。它会按组件打印一行 `PASS|FAIL|SKIP <component>`,一个 `GSTACK_LAUNCH:` 行,以及恰好一个收尾行作为最后一行:`gstack: ok <revision> for=<skill>` (退出码 0)或 `gstack: fail <ids> ...` (退出码 1)。读取退出状态并直接 grep `^gstack: `;不要单独通过 `tail`、`head` 或 `grep` 管道传输该 check。`--check` 永远不会移动检出。

2. 如果收尾行列出了 `revision`、`runtime` 或 `pins`(`gstack: fail revision ...`、`gstack: fail runtime,pins`、`gstack: fail pins`)且检出存在,运行同一命令但去掉 `--check`(900 秒超时)。对于 `pins`,当 pin 命名一个已校验的版本时,它会安装项目锁定的 Bun(约 4 秒),否则会以 `RUNTIME_PIN_CONFLICT` 命名要上报的 pin 后停止。一个干净的、未锁定的、低于支持最低版本的检出会被快进(`gstack-capy-skill: upgraded <old> -> <new>`),然后在一个锁定的步骤里重新安装并重新 check;在什么都没变的情况下约 1 秒。被锁定的(`--revision`、detached)、脏或已分叉的检出永远不会移动:收尾行命名原因和精确的命令。`GSTACK_CAPY_AUTO_UPGRADE=0` 禁用该快进。

3. 如果收尾行是 `gstack: fail not-installed ...` 且本会话不运行任何 gstack 工作流,记录那一行然后继续;不会安装任何东西。在第一个工作流之前安装:`bash scripts/install.sh --for <skill>` (冷启动约 30 秒:克隆、SHA-256 校验过的 Bun 1.4.2、两个宿主 profile,且不包含 Chromium)。

4. 任何其他 `FAIL` 行都带有一项 `fix:` 命令和一个原因代码及故障排查锚点;运行该 fix,然后再次 `--check`。`SKIP codex-cli (optional; ...)`、`SKIP browser (lazy; ...)` 和 `SKIP pins (planning-only; ...)` 不是失败。

**收尾行规则。** 本次会话里本机上没有 `^gstack: ok` 行意味着 gstack 在这里未被安装:不要运行工作流,也不要声称这些技能可用。一台机器上的安装永远不会覆盖另一台。

**子代理。** 子代理向其父代理的第一份报告应携带来自其自身机器的 `--check` 的 `gstack:` 收尾行,逐字原样,紧随其后是它的标识符。父代理把缺失的行视为 `gstack: fail not-installed`。

`gstack-capy-skill: owner action — Drive copy is wrapper <n>, minimum <m>; copy <root>/capy/skill/gstack/ (CAPY_WRAPPER_STALE)` 表示该 skill 目录比检出期望的版本要旧。运行仍然有效;告诉所有者,由所有者把该目录复制到 skill 目录之上。永远不要自己写 skill 目录。

## 用 GSTACK_LAUNCH 行启动每条 gstack 命令

每次成功运行,包括 `--check`,都会打印 `GSTACK_LAUNCH: PATH=<dir>:$PATH GSTACK_SESSION_KIND=unattended GSTACK_STATE_ROOT=<path> GSTACK_EPHEMERAL=1 GSTACK_CAPY=1 GSTACK_INSTALL_VERDICT=<file>` (仅当已校验的 Bun 不在 PATH 的首位时才有 `PATH=` 部分),然后是 `GSTACK_LAUNCH_PATH: <dir>`,供不解析 shell 语法的调用者使用。在每条 gstack 命令前加上这些赋值,或者在执行它的 shell 里 export 一次;子 shell 的 export 不能配置其父级稍后的命令。PATH 部分正是让 gstack 的 `#!/usr/bin/env bun` 命令能跑过收尾行所认证的那个 Bun;Capy 镜像的 `/usr/local/bin/bun` 是 1.3.14,低于 gstack 的最低版本。

在 `GSTACK_CAPY=1` 下,在 unattended 或 spawned 会话中启动的工作流会检查上次运行写入的、针对此检出、HEAD 与工作流的安装 verdict,以及 PATH 上的 Bun。当任何一项失败时,它会打印一行 `INSTALL: fail ...`,附带修复命令,然后 `GSTACK_RESULT: skill=<s> status=refused run=-` 并退出码 3:停下来,运行修复,重新开始。一个 ok,但其 pins 行是 `SKIP pins (planning-only; ...)`,不允许运行仓库测试的工作流(`review`、`ship`、`qa`),verdict 未检查该工作流所需组件的也不允许;先运行 `--for <that skill>`。

## 加载正确的工作流

对于运行 GPT-6 Astra 的 Capy,读取 `${CODEX_HOME:-$HOME/.codex}/skills/gstack/SKILL.md` 以获取路由,然后读取 `${CODEX_HOME:-$HOME/.codex}/skills/gstack-<skill>/SKILL.md` 以获取所选工作流。这些是生成的 GPT-6 Astra 指令,不是 Claude 源文件。

对于 Claude Code,读取 `~/.claude/skills/gstack/SKILL.md` 以获取路由,再读取 `~/.claude/skills/<skill>/SKILL.md` 以获取所选工作流。在启动 Claude Code 会话时,告诉它加载 gstack 并命名预期的工作流。对于运行 Claude 模型的 Capy,Claude 指令文件可以直接用下面的 Capy 适配读取;这不会把 Capy 变成 Claude Code 宿主。

如果没有指定特定 skill,请使用路由器的任务-技能映射。只读取所选的工作流及其引用的文件。对于其他模型,使用最接近的可用指令 profile,但不要声称生成了或评估了特定于该模型的 profile。

## 父代理的 unattended 运行

对于从 Capy 驱动的 `/autoplan`,在检出中读取 `autoplan/DRIVER.md`(`~/.local/share/gstack/autoplan/DRIVER.md`):紧凑的阶段循环、产物契约以及 gate 协议,不包含交互式 preamble。没有该文件的检出早于它;请改用其 `autoplan/SKILL.md`。所有命令都需要用 `GSTACK_LAUNCH` 赋值运行,并将输出作为文件而非散文读取:运行目录包含 `run.json`、`decisions.jsonl`、`findings.jsonl`、`tasks.jsonl`、`timing.json`、`plan.md` 和 `review-record.md`,并且运行以 `GSTACK_RESULT: skill=autoplan status=gate_pending run=<dir>` 结束。`~/.local/share/gstack/bin/gstack-artifact validate <dir>/run.json` 是完成检查。gate 通过 `gstack-autoplan answer`(或 `gstack-gate parse`)用打印的 `gate_rev` 回答;unattended 运行从不会批准其自身的 gate。检出里的 `docs/unattended.md` 是该契约。

## 在推送或发起 PR 之前**

在具有封印配置(`.gstack/pregate.json` 中存在 `sealed` 键)的仓库中,在每次推送和发起 pull request 之前,运行 `~/.local/share/gstack/bin/gstack-pregate sealed`:Capy 的 pull-request 工具可能不会运行本地 git hooks,所以这是把封印或 eval 输出挡在公共仓库之外的检查。拒绝时会命名文件、行号和规则;在该检查通过前不要推送。

## 将特定于宿主的指令适配到 Capy

- 调用 `Skill` 的指令意味着读取并遵循所选的 SKILL.md,并使用可用的文件读写工具。不要调用不存在的 Skill 工具,也不要假设 slash 命令已在 Capy 的 UI 中注册。
- 使用 Capy 的用户可见消息与问题工具来输出和决策。普通的助手文本在 Capy 中不是已交付的消息。使用 Capy 的任务与 pull-request 工作流,而不是不可用的 Claude 或 Codex 宿主工具。
- Capy 没有原生 gstack 宿主目标。Claude 钩子不会变成被强制实施的 Capy 钩子;这里的 safety skills 是建议性的,`/autoplan` 打印一次 `autoplan guard: not enforced by this host` 以说明这一点。报告缺失的能力,而不是默默声称为同等的强制实施。
- 在 Capy 中选择 GPT-6 Astra 或 Claude 并不会安装或鉴权 Codex 或 Claude Code CLI。对于跨模型审查,使用另一个模型族的子代理是一个完整的外部声音;记录评审的模型。当另一个 runner 已评审时,不要把缺失的 Codex CLI 报告为降级的覆盖。
- 保留用户请求的作用域、决策与同意。不要代他们启用遥测或远程产物共享。安装 gstack 并不授权合并、部署、破坏性操作或绕过审批。

## 维护

`bash scripts/install.sh --check` 是健康检查;`bash scripts/install.sh` 安装或修复,并把低于支持最低版本的检出快进。在最低版本之间,`~/.local/share/gstack/bin/gstack-upgrade --check` 报告检出是否落后,`~/.local/share/gstack/bin/gstack-upgrade` 把检出快进到默认分支的 tip 并重新安装(它拒绝脏的、分叉的或被锁定的检出,且从不 stash 或 reset)。该安装器面向 Capy 的 Ubuntu 云机器,这些机器具备 Git、curl、Node.js、flock 和 timeout;未经许可,不要在个人设备上运行它。