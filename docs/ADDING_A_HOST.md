# 向 gstack 添加新主机

gstack 使用声明式主机配置系统。每个支持的 AI 编码 agent(Claude、Codex、Factory、Kiro、OpenCode、Slate、Cursor、OpenClaw、Hermes、GBrain、Copilot、Antigravity)都被定义为由 `defineHost()` 工厂构建的类型化 TypeScript 配置对象。渲染新主机意味着创建一个文件并重新导出它:生成器、测试和验证会处理它。安装它不是免费的:setup 仍然为每个可安装的主机提供一个安装分支(见下面的"setup 在磁盘上的更改"),因此可安装的主机还需要一个 setup 分支、安装注册表 tier 表中的一个条目,以及在被标记为 `full` 之前的认证运行。

## 安装所有权规则

这些规则适用于每个主机、每个范围和每个升级路径(`./setup`、`./setup --refresh-registered`、`/gstack-upgrade`、team-mode 自动更新)。每次拒绝都命名保留的内容、应用的规则以及要继续的确切命令。

1. **项目本地 setup 永远不会默默替换全局安装。** 从项目 vendored 副本运行的 setup,或从全局安装链接到的 checkout 之外的任何 checkout 运行的 setup,会保留该全局安装并打印 `./setup --global` 作为故意替换它的方式。(Codex:`setup:186-207`,#2879。Claude:全局 `~/.claude/skills/gstack` 链接。)
2. **显式 `--host X` 永远不会变更另一个主机。** `--host codex` 仅写入 Codex 的 skills 目录和 checkout 自己的 `.agents/` 渲染。它不会修复、注册或删除 `settings.json` 中的 Claude Code hooks,也不会触及 Claude gbrain 渲染(#2347)。一个故意的例外:一次性 `/claude` → `/claude-code` 重命名修复(`bin/gstack-migrate-claude-code`)重命名共享该 checkout 的每个安装中的 gstack-owned 条目。
3. **升级精确刷新已注册的安装。** 每个激活的安装 setup 都记录在安装注册表中。升级运行 `./setup --refresh-registered`,它为该 checkout 的每个注册主机运行一次 setup(首先打印来源),除此之外不运行任何内容。另一个 checkout 拥有的安装和另一个项目的项目安装会列出在该处刷新它们的命令;它们永远不会被重新指向(#1925)。
4. **用户拥有的文件幸存。** Setup、`gstack-relink` 和 `gstack-uninstall` 仅在有证据证明 gstack 创建了它(#2119 所有权 gate)的情况下才删除或链接到条目;任何其他内容都会被报告并保留。

### 安装注册表

`$GSTACK_STATE_ROOT/installs.tsv`(模式 0600)是安装的唯一记录。只有 `bin/gstack-install-registry.sh` 写入它,在带有原子重命名的锁目录下。列:host、范围(`global`/`project`)、project、目的地(host 发现的 skills 目录)、install root、source realpath、version、prefix 设置、render、updated-at。一行仅在主机分支完成激活后才发布。其 install root 已消失的行由 `gstack_install_registry_reconcile` 在每次刷新和 `gstack-uninstall` 时删除,因此升级永远不会复活已卸载的主机。

`./setup --status` 为每个安装(主机、tier、范围、结果、版本、目的地、来源)打印一行,而无需构建或写入任何内容,以及它在磁盘上找到的任何预注册安装,标记为 `unregistered` 并附上注册它的命令。由项目 vendored 副本捕获的 Codex 安装(#2879)被报告为具有两个选择,而不是被重新指向。setup 摘要、`/gstack-upgrade` 摘要和 `--status` 共享一个渲染器(`gstack_install_registry_render`)。

### 迁移

`gstack-upgrade/migrations/v*.sh` 是 state-root 迁移。它们的标记是 `$GSTACK_STATE_ROOT/.last-setup-version`(为早于移动的安装从 `~/.gstack` 读取一次)。它仅在过去成功的迁移之后推进;失败的迁移停止序列,打印带有重试命令的 `failed` 行,并在下一次 `./setup` 上重新运行。安装和主机迁移由注册表行的版本按安装跟踪。

### Setup 在磁盘上的更改

按阶段列出的每个 setup 副作用的清单(行号是近似的):

| 阶段 | 写入 | 范围 |
|---|---|---|
| 构建(Build) | `browse/dist`、`design/dist`、`make-pdf/dist`、`bin/gstack-cso-*` 在源 checkout 中 | 源 |
| 主机记录(Hosts record) | 源 checkout 中的 `.gstack-installed-hosts`:从其安装的每个主机,累加(#1694) | 源 |
| 生成(Generation) | 仅渲染记录的主机(`.agents/`、`.kiro/`、`.factory/`、`.opencode/`、`.cursor/`、`.copilot/`、`.agy/`)在源 checkout 中;`gstack-patch-names` 重写 Claude `name:` 字段到位 | 源 |
| 渲染修剪(Render prune) | 记录未命名的主机的渲染:链接到 checkout 删除,生成的文件移动到 `$GSTACK_STATE_ROOT/backups/host-renders/<ts>-<id>/`,其他文件保留 | 源 / state root |
| Chromium | Playwright 缓存(`~/.cache/ms-playwright`),state root 下的锁 | 机器 |
| 配置(Config) | `$GSTACK_STATE_ROOT/config.yaml` 中的 `skill_prefix`、`timeline_stop_hook`、team-mode 键 | state root |
| Claude 分支(Claude arm) | `~/.claude/skills/gstack` 链接,每个 skill 一个目录,带有 `SKILL.md` 链接,别名副本 | 选定的主机 |
| Codex 分支(Codex arm) | `${CODEX_HOME:-~/.codex}/skills/gstack` 运行时根,`.agents/skills` 中的 `gstack-*` 链接,`.agents/skills/gstack` sidecar | 选定的主机 |
| Kiro、Factory、OpenCode、Cursor、Copilot、Antigravity 分支 | 主机的 `skills/gstack` 运行时根(在活动根旁边构建并换入)和 `gstack-*` 链接 | 选定的主机 |
| OpenCode 命令 | `~/.config/opencode/commands/gstack-*.md` 带托管标记(#2629);用户命令文件永远不会被触及 | 选定的主机 |
| 注册表(Registry) | 状态根中每个激活安装的一行 `installs.tsv` | state root |
| 迁移(Migrations) | 每个 `v*.sh` 修复的内容;state root 中的标记 | state root |
| Claude hooks | `~/.claude/settings.json`(修复、SessionStart、plan-tune、Stop)——仅当选择 Claude 时 | Claude |
| gbrain | `gbrain-detection.json`;`$GSTACK_STATE_ROOT/render/claude` 中的 Claude 渲染——仅当选择 Claude 时 | state root / Claude |

今天的激活:在 macOS 和 Linux 上,每个安装都是源 checkout 中的一组符号链接,因此 `git pull` 在 setup 运行之前更改活动有效负载,而"保持先前有效负载"意味着"保持先前 checkout"——这是升级 skill 在 vendored 路径上所做的(`.bak` 还原)并在 git 路径上记录(`PRE_UPGRADE_COMMIT`)。在 Windows 上,setup 复制而不是链接,每次复制分阶段为 `<dst>.gstack-new.<pid>` 并仅在完成时换入;中断的换入在下一次运行时修复(缺少的目的地从 `<dst>.gstack-old.<pid>` 还原)。符号链接平台的版本化有效负载目录尚未构建。

### 安装上下文渲染契约

提交的渲染命名每个主机的默认安装根。任何其他根的渲染(项目安装、`CLAUDE_CONFIG_DIR`、重命名的 checkout)使用 `gen-skill-docs --host <h> --install-root <abs path> --out-dir <tmp> --link-root <dir>` 制作:每个默认根路径变为给定的根,解析器将其视为 `TemplateContext.installRoot`(null 保留提交的字节)。每次安装的渲染存在于 `$GSTACK_STATE_ROOT/render/installs/<host>-<id>/`(`<id>` 由根的 realpath 键控),并记录在注册表的 render 列中。Setup 为 `~/.claude/skills/gstack` 之外的 Claude 安装(重命名的 checkout、项目 vendored 副本、`CLAUDE_CONFIG_DIR`)制作一个,为 `~/.codex/skills/gstack` 之外的全局 Codex 安装(`CODEX_HOME`)制作一个;仓库本地 Codex 安装在运行时解析其根。不是普通路径(空白、shell 元字符)的根通过 `<host>-<id>.root` 别名符号链接命名:worktree-isolated Claude Code 拒绝任何引号中包含空格的命令路径。

### 安装中的主机渲染(#1694)

checkout 仅渲染从其安装的主机的技能。`bin/gstack-host-renders.sh` 拥有该记录,即 checkout 中的 `.gstack-installed-hosts`:setup 添加它安装的每个主机并且从不删除一个,因此 `./setup --host codex` 保留 `--host factory` 渲染。升级后的第一次运行从 checkout 已服务的主机(注册表行、解析为它的 install root、链接到其渲染的主机技能,以及没有运行时根的传统副本)植入它。`scripts/build.sh` 渲染 claude 加记录的主机并打印哪些主机和为什么;没有记录的 checkout(开发 checkout)或 `GSTACK_RENDER_HOSTS=all` 渲染每个主机。

然后每次 setup 运行使用 `bin/gstack-relink` 的证据规则修剪未记录的主机的渲染:checkout 中的符号链接(强证据)被删除;证明生成的文件(弱证据:gen-skill-docs 横幅、生成器的确切 `openai.yaml` 形状、与 checkout 的字节同一性)被移动到 `$GSTACK_STATE_ROOT/backups/host-renders/<ts>-<id>/`;每个其他文件保留在其位置,只删除留空的目录。备份的 `prune.log` 列出每个路径。gstack-upgrade 迁移为没有 setup 运行升级的安装运行相同的修剪。仅指令主机(Hermes、OpenClaw、GBrain)永远不会被 setup 安装,因此手动在安装内渲染一个的用户首先将其名称添加到记录中。

## 主机层级和能力

每个主机声明 `tier` 和 `capabilities`(`scripts/host-config.ts`)。一个词汇到处使用:`./setup --help`、setup 和 `--status` 行、主机错误和 README 主机矩阵,所有这些都对照 `hosts/index.ts` 通过 `test/host-config.test.ts` 进行检查。

| 层级(Tier) | 含义 |
|---|---|
| `full` | 安装、渲染、通过一致性套件,并且下面有带日期的认证记录。 |
| `experimental`(实验性) | 安装并渲染并通过一致性套件;还没有认证记录。 |
| `instruction-only`(仅指令) | 无安装分支。`./setup --host X` 打印要复制的内容并且不更改任何内容。 |

仅渲染永远不会认证 `full`。

| 能力(Capability) | 值 | 用于 |
|---|---|---|
| `toolExecution` | 布尔值 | preambles 和 `bin/` helpers 运行 |
| `questions` | `native` / `prose` | 结构化问题工具 vs 纯文本问题 |
| `planMode` | 布尔值 | plan-mode 转换 |
| `delegation` | 布尔值 | 子智能体 |
| `browser` | 布尔值(需要 `toolExecution`) | gstack 的浏览器 |
| `safetyHooks` | `enforced` / `advisory` | `/careful`、`/freeze`、`/guard` 阻塞(Claude)或仅警告 |

具有 `safetyHooks: 'advisory'` 的主机在 `/careful`、`/freeze` 和 `/guard` 的顶部获得一行:"not enforced on <host>: advisory, not blocked"。

## 认证你的主机

1. 通过一致性套件:`bun test test/host-conformance.test.ts` 将每个可安装的主机安装到临时 HOME 中,检查发现目录、没有其他主机的变体、没有重复、`./setup --status` 以及从先前版本的安装布局进行升级。
2. 在主机上运行一个真实的有代表性的工作流(例如在小型 diff 上运行 `/review`),以及从现有安装进行升级。
3. 在下面添加一行,包含日期、主机版本、gstack 版本以及脚本所在的位置,然后设置 `tier: 'full'`。

| 主机 | 日期 | 主机版本 | gstack 版本 | 证据 |
|---|---|---|---|---|
| claude | 2026-10-02 | Claude Code 2.1.284(CI 镜像 pin) | v1.91.13.0 (1c54555) | gate-tier evals, real Claude Code sessions: https://github.com/garrytan/gstack/actions/runs/37033755821 |

Codex 保持 `experimental`,直到其周期性用例(`codex-discover-skill`、`codex-review`)在 CI 镜像的固定 Codex CLI 上通过并添加一行。

## GitHub Copilot CLI(#393)

作为 `experimental` 发布(`hosts/copilot.ts`,`./setup --host copilot`):`~/.copilot/skills/gstack-*` 中的技能,`~/.copilot/skills/gstack` 中的运行时根,带有 `.source-path` 标记以便 `/gstack-upgrade` 找到 checkout,名称前缀为 `gstack-`(否则 Copilot 的内置 `/review` 会胜出),敏感技能上的 `disable-model-invocation: true`,以及插入到 preamble 的 STATUS 规则之前的单段工具名称词汇表(`ask_user`、`exit_plan_mode`、`task`、`view`、`skill`)。共享目录是不够的:Copilot CLI 在 1.0.36 中停止读取 `~/.claude/skills`,gstack 从不写入 `~/.agents/skills`,而 Codex 渲染带有 Codex 路径和 Codex model profile。

社区尝试的处理:

| PR | 决定 | 原因 |
|---|---|---|
| #2323 (@andrey-esipov) | 选择性重用,并致谢 | 保留:`~/.copilot/skills` 和 `.github/skills` 根、`copilot`-binary 检测、`.source-path`、`gstack-` 前缀、1024 字符描述限制、卸载覆盖。不保留:新 schema 字段和生成器编辑(早于 `defineHost()`)、运行时根的未受保护的 `rm -rf`(#2142)、递归扫描目录内对 `setup`/`scripts`/Claude 源的链接、setup 中的仅测试安装后门,以及全面的 AskUserQuestion → `ask_user` 重写。 |
| #396 (@ridermw) | 拒绝 | Sed-将 Codex 输出重写为共享的 `.agents/skills`;其 E2E runner 将真实的 `~/.copilot` 凭据复制到临时 HOME。它的会话扫描器是以后 `/retro global` 更改的候选。 |
| #487 (@ridermw) | 拒绝 | 依赖 `~/.claude/skills`,Copilot CLI 不再读取。 |
| #1852 (@lolisaigao1234) | 拒绝 | 检测 `gh` 而不是 `copilot`,安装到 Copilot 不扫描的目录,在没有 `lib` 的情况下链接 `bin`。 |

未经真实 Copilot CLI 验证(为什么它不是 `full`):自动驾驶可能自己回答 `ask_user`,plan mode 阻塞变更 shell 命令,符号链接的 skill 目录需要 Copilot CLI 1.0.62 或更高版本,以及 `COPILOT_HOME` 而不是 `~/.copilot` 现在被拒绝。

## Antigravity CLI

作为 `experimental`(`hosts/agy.ts`,`./host agy`,别名 `--host antigravity`)为 Google 的 Antigravity CLI 发布。主机以 `agy` 二进制命名:使用较长的名称时,每个 fence 运行时序言对 400 字节预算是 408 字节(全局根很长);`agy` 正好渲染 400,因此必须使用预算更改才能将下一个字节添加该序言。针对 Google 的文档验证于 2026-10-08:

| 事实 | 值 | 来源 |
|---|---|---|
| 全局技能 | `~/.gemini/antigravity-cli/skills/<skill>/SKILL.md` | [Agent skills](https://antigravity.google/docs/skills/)(CLI 部分)、[Plugins & skills](https://antigravity.google/docs/cli/plugins/)、[Migrating from Gemini CLI](https://antigravity.google/docs/cli/gcli-migration/) |
| 工作区技能 | `.agents/skills/<skill>/SKILL.md` | 同上 |
| Frontmatter | `name`(可选,默认为目录)、`description`(必需) | [Agent skills](https://antigravity.google/docs/skills/) |
| 描述限制 | 1024 字符(Agent Skills 格式) | [What are Agent Skills?](https://cloud.google.com/discover/ai-agent-skills) |
| 工具名称 | `run_command`、`view_file`、`write_to_file`、`replace_file_content`、`multi_replace_file_content`、`grep_search`、`find_by_name`、`invoke_subagent`、`ask_question` | [Hooks: supported tools](https://antigravity.google/docs/hooks/) |
| Plan mode | `agy --mode=plan` / `/plan`,无 exit-plan 工具 | [Execution modes](https://antigravity.google/docs/cli/modes/) |
| 二进制文件 | `agy`,安装到 `~/.local/bin/agy`;配置在 `~/.gemini/antigravity-cli/` 中 | [Installation](https://antigravity.google/docs/cli/install/)、[Settings](https://antigravity.google/docs/cli/settings/) |

主机做什么:`~/.gemini/antigravity-cli/skills/gstack-*` 中的技能,`~/.gemini/antigravity-cli/skills/gstack` 中的运行时根,带有 `.source-path` 以便 `/gstack-upgrade`,`name:` 等于 `gstack-<skill>` 目录(CLI 将每个名称转换为 slash 命令,并且有 `/plan`、`/diff`、`/skills` 和其他内置),frontmatter 限制为 `name` 和 `description`,Gemini model overlay,以及在 preamble 的 STATUS 规则之前的单段工具词汇表。词汇表还告诉智能体在 `view_file` 仅返回长 SKILL.md 的一部分时继续阅读:用户报告首次读取在 800 行停止([forum](https://discuss.ai.google.dev/t/make-the-800-line-forced-first-read-in-view-file-configurable-or-opt-out/172257)),Google 的文档没有说明,因此渲染的文本不命名任何数字。

**共享 `~/.gemini`。** Antigravity CLI 和 Gemini CLI 都位于 `~/.gemini` 下。Gemini CLI 读取 `~/.gemini/skills`(以及 `~/.agents/skills`);Antigravity CLI 读取 `~/.gemini/antigravity-cli/skills`。Setup 仅在那里写入 gstack 条目,卸载仅删除那里的 gstack 管理的条目,而 `--host auto` 仅在 `PATH` 上的 `agy` 或现有的 `~/.gemini/antigravity-cli` 时选择 Antigravity,而不是裸 `~/.gemini`。`test/setup-install-registry.test.ts` 保存两个方向。Gemini CLI 作为主机不受支持。仍然存在两个重叠:workspace `.agents/skills` 也是项目本地 Codex 安装放置其 Codex 风格的 `gstack-*` 技能的地方,Antigravity 将在该仓库中列出它们,而 Antigravity 2.0 应用程序和 IDE 读取 `~/.gemini/config/skills`,此主机不安装(Google 的 codelab 表示该目录也可以访问 CLI;CLI 文档没有,所以 setup 遵循 CLI 文档)。

社区尝试的处理:

| PR | 决定 | 原因 |
|---|---|---|
| #2134 (@0xshae) | 选择性重用,并致谢 | 保留:`agy` 作为主机名称和二进制、`.agents/skills` 作为工作区根,以及与 Codex 的 `.agents` 不同的 `.agy` 渲染目录。不保留:手写配置(早于 `defineHost()`)、`~/.gemini/config/skills` 作为 CLI 根,以及没有所有权 gate 的 setup 分支。 |
| #2893 (ManchalaSashank) | 选择性重用,并致谢 | 保留:setup 和生成中的 `agy`/`antigravity` 作为主机名称和别名、Gemini overlay、卸载覆盖。不保留:`~/.gemini/config/skills`、裸 `~/.gemini`(Gemini CLI 的目录)上的自动检测、`.gemini` 作为渲染目录,以及仓库本地的 `.gemini/skills` sidecar。 |
| #2726 (@voipexpert) | 选择性重用,并致谢 | 保留:Antigravity 自己的文档中的工具名称,以及 `manage_task`(后台进程)不是待办工具。不保留:全面工具名称重写(preamble 词汇表替换它们)以及抑制通过 shell 运行的外部评审。 |
| #2244 (@KiDDarn) | 选择性重用,并致谢 | 保留:当没有链接技能时安装失败。不保留:`~/.gemini/config/plugins` 插件布局(CLI 插件位于 `~/.gemini/antigravity-cli/plugins` 下并通过 `agy plugin install` 安装)、benchmark provider 工作以及额外的 frontmatter 字段。 |
| #413 (@jnMetaCode) | 拒绝 | 安装到 `~/.gemini/skills` 并检测 `gemini`:这是 Gemini CLI。那里的一条评论(@doogiehowzer)报告了词汇表现在处理的部分 `view_file` 读取。 |
| #2079 (@FACUTUCCI10) | 拒绝 | Gemini CLI 路径(`~/.gemini/skills`)。 |
| #2187 (@NotArsal) | 拒绝 | 没有 setup 的 #2134 的副本。 |
| #1490 (@7H0M45-4N70NY) | 拒绝 | `~/.antigravity/skills`,没有 Antigravity 表面读取。 |
| #565 (@ChetanTekur) | 拒绝 | Gemini CLI `GEMINI.md` 支持;Gemini CLI 不在范围内。 |

未经真实 Antigravity CLI 验证(为什么它不是 `full`):CLI 遵循符号链接的 skill 目录,它不将运行时根的额外 markdown 文件(`ETHOS.md`、`review/*.md`)作为技能导入,长技能上 `view_file` 的部分读取行为,`ask_question` 选项处理,以及从现有安装升级。

## 仅指令主机

OpenClaw、Hermes、Slate 和 GBrain 没有安装分支。Hermes 保持仅指令,直到有人认证本地安装(#2826 在 Hermes 0.21.0 上测量了一个):`./setup --host hermes` 打印摘要路径和生成器命令,而 Hermes 渲染写入 `name: gstack-<skill>`,因此 Hermes 通过 frontmatter 名称索引技能,无法影子它们(#2825)。

## 它如何工作

```
hosts/
├── define-host.ts   # defineHost() 工厂:共享默认值 + 派生字段
├── claude.ts        # 主要主机
├── codex.ts         # OpenAI Codex CLI
├── factory.ts       # Factory Droid
├── kiro.ts          # Amazon Kiro
├── opencode.ts      # OpenCode
├── slate.ts         # Slate (Random Labs)
├── cursor.ts        # Cursor
├── openclaw.ts      # OpenClaw
├── hermes.ts        # Hermes (Nous Research)
├── gbrain.ts        # GBrain
├── copilot.ts       # GitHub Copilot CLI
├── agy.ts           # Google Antigravity CLI
└── index.ts         # 注册表:导入所有,派生 Host 类型
```

每个配置文件调用 `defineHost()` 并导出生成的 `HostConfig` 对象,它告诉生成器:
- 将生成的技能放在哪里(paths)
- 如何转换 frontmatter(允许/拒绝列表字段)
- 要重写哪些 Claude 特定的引用(路径、工具名称)
- 要检测什么二进制文件以进行自动安装
- 要抑制哪些解析器部分
- 安装时要符号链接哪些资产

生成器、setup 脚本、平台检测、卸载、健康检查、工作树副本和测试都从这些配置中读取。它们都没有每个主机的代码。

## 一步一步:添加新主机

### 1. 创建配置文件

配置是使用 `hosts/define-host.ts` 中的 `defineHost()` 工厂构建的。你只编写与常见外部主机默认值不同的字段;其他所有内容都从主机名称派生。完全默认的主机是两行。
