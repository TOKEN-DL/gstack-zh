# gstack development

## Commands

```bash
bun install          # install dependencies
bun run dev <cmd>    # run CLI in dev mode, e.g. bun run dev goto https://example.com
bun run typecheck    # strict tsc over product code (zero errors required)
bun run typecheck:test  # test-code type-debt ratchet
bun run build        # gen docs + compile binaries
bun run gen:skill-docs  # regenerate SKILL.md files from templates
bun run skill:check  # health dashboard for all skills
bun run dev:skill    # watch mode: auto-regen + validate on change
bun run slop          # full slop-scan report (all files)
bun run slop:diff     # slop findings in files changed on this branch only
bun run audit:manifest  # file slices for /claude-api prompt-audit; rerun it at each frontier-model release (CONTRIBUTING.md)
```

测试与评估命令、它们的成本和所需内容,集中在一张表里:
[Which command do I run?](CONTRIBUTING.md#which-command-do-i-run). 你最需要的是 `bun run test`(免费验收)和 `bun run eval:bg:pr`(付费、变更覆盖)。已退役的名字如 `test:evals` 和 `test:e2e` 会打印其替代命令并以 exit 1 退出。

付费评估需要 `ANTHROPIC_API_KEY`。Codex E2E 测试(`test/codex-e2e.test.ts`、`test/codex-e2e-sol-scope.test.ts`)使用 Codex 自己的鉴权——hermetic runner 仅从 `${CODEX_HOME:-~/.codex}` 复制 `auth.json` 并在子进程 env 中钉住 `CODEX_HOME`,无需 `OPENAI_API_KEY` 环境变量。

**Hermetic E2E + 环境键:** 每个 E2E runner 通过 `test/helpers/hermetic-env.ts` 派生子进程(allowlist 净化的环境、新鲜播种的 `CLAUDE_CONFIG_DIR`、临时 `GSTACK_HOME`、`--strict-mcp-config`);per-test `env:` 覆盖合并到最后,作用于一个完整的 hermetic env,因此是安全的。需要键入 `/skill` 命令的 PTY 测试必须传 `seedSkills: true`。用真实运维者状态调试,设 `EVALS_HERMETIC=0`。完整细节(env shim、seeding tripwires、wiring 测试):
[docs/TESTING_INTERNALS.md](docs/TESTING_INTERNALS.md)。

**测试选档与层级(tier):** 分片付费 runner(`test:pr`、`eval:bg:pr`、`test:gate:sharded`)通过 `test/helpers/touchfiles.ts`(`EVALS_ALL=1` 强制全跑;`eval:select` 预览 PR profile,`--profile full` 显示普通 touchfile 选择)中的依赖列表,从 `git diff` 中选择测试。在 `E2E_TIERS` 中为每个新的 E2E 测试分类:安全护栏或确定性功能测试 → `gate`;质量基准、Opus 模型测试、非确定性或外部服务(Codex、Gemini) → `periodic`。`test/e2e-tier-alignment.test.ts` 强制层级对齐。CI lane 与 periodic 排除:[docs/TESTING_INTERNALS.md](docs/TESTING_INTERNALS.md#test-selection-and-tiers)。

## Testing

```bash
bun run test         # final full free acceptance after focused repairs and source freeze
bun run eval:bg:pr   # required changed PR coverage, with explicit deferrals
```

遵循 [Validation discipline in AGENTS.md](AGENTS.md#validation-discipline):用聚焦检查证明修复,完成必需的选定评估,然后在最终整合代码上跑一次完整免费套件。在修复期间,聚焦检查在每次提交前替代完整套件运行。

`bun run test` 走 `scripts/test-free-shards.ts`,其严格的 per-shard 分类在缺失 bun 摘要行时让 shard 失败。`TREE_MUTATING` 列出仍在自己尾部串行 shard 中运行的文件(目前只有 `test/bootstrap-retention.test.ts`)。永远不要为整个套件键入裸 `bun test`:它会遍历整个仓库、加载付费评估文件、并且错过严格分类器。`bun run test:pr` 跑选定的短实时行为和质量评估,并报告已延后的广覆盖。完整免费验收与必需的 PR 检查必须在发布前通过。命令与衡量目标见 [testing policy](CONTRIBUTING.md#test-tiers)。需要 Aside 本身的测试会在 Mac 装了 Aside app 的情况下才自跳。分片打包、评估复用与引擎跳过:
[docs/TESTING_INTERNALS.md](docs/TESTING_INTERNALS.md#free-suite-runner-judge-reuse-and-engine-skips)。

新增或变更的测试遵循 [test value bar](docs/test-value-bar.md):每一条都在保护一个真实回归会破坏的行为,且契约测试(SKILL.md goldens 与下面定义的提示字节契约)保留。bar 的源在 `scripts/resolvers/test-value.ts`。

在本仓库中,提示字节仅在以下情况下才是契约:软件会读取它们(解析器、hook、评分器或另一个 skill 消费确切的文本),或有记录在案的评估显示措辞确实重要。模板与生成 SKILL.md 上的测试钉住结构、步骤顺序、路由表、机器可读标记以及安全相关行;对安全行做不区分大小写的关键字共现检查,关注意义而非大写。行为留给 E2E 用例与裁判。不要钉住强调、大小写、issue 编号或行为检查已经覆盖的句子;当一次重写改变了被钉住的句子时,用结构或意义级检查替换钉子,而不是钉住新句子。这些定义在 [docs/test-value-bar.md](docs/test-value-bar.md)("Prompt-byte contract"、"Sentence pin")中。模板/SKILL.md 检查用 `test/helpers/prompt-structure.ts`(`between`、`expectTokens`、`expectAbsent`、`expectOrdered`、`expectMentions`):token 和顺序精确,安全规则作为一个句子内的不区分大小写的关键字共现。

项目用 CLAUDE.md 中可选的 `## Test Coverage` 键来调优 `/ship` 的覆盖门,默认全部缺席:`Minimum:`、`Target:`、`Generation cap:`(默认 5)、`Base control:`(`auto` 或 `off`)、`Base control budget:`(秒,默认 90)以及 `Star rating:`(`auto` 或 `off`)。

## Project structure

完整带注释的目录树:[docs/PROJECT_STRUCTURE.md](docs/PROJECT_STRUCTURE.md)。速查地图:`browse/` gstack 自己的无头浏览器 CLI(回退引擎)加上 `/browse` skill;`design/` 设计二进制;`make-pdf/` PDF 二进制;`hosts/` 类型化 host 配置;`scripts/` 构建+DX 工具(gen-skill-docs、resolvers——`resolvers/aside.ts` 是 Aside 契约);`test/` 验证+评估;`lib/` 共享库(`aside-render.ts` 通过 Aside 渲染本地 HTML,引擎缺失时回退;`design-catalog.ts` 是每个设计 skill 渲染所用的类型化设计反模式目录);`bin/` CLI 工具(`gstack-render.ts` 是 skill 调用的渲染 CLI;`gstack-design-detect.ts` 与 `gstack-design-md.ts` 是设计检测器与 open-DESIGN.md 工具);`extension/` Chrome extension;每个 skill 一个目录(`ship/`、`review/`、`qa/`、...);`.github/` CI;`contrib/` 贡献者工具;`docs/designs/` 设计文档。

## SKILL.md workflow

SKILL.md 文件由 `.tmpl` 模板**生成**。要更新文档:

1. 编辑 `.tmpl` 文件(例如 `SKILL.md.tmpl` 或 `browse/SKILL.md.tmpl`)
2. 运行 `bun run gen:skill-docs`(或 `bun run build` 会自动完成)
3. 同时提交 `.tmpl` 和生成的 `.md` 文件

同一次 `gen:skill-docs` 运行会从 `lib/` 再生成两个文件:`review/design-checklist.md`(来自 `lib/design-catalog.ts`,通过 `scripts/resolvers/design-checklist.ts`)以及 `lib/dom-dump.js`(来自 `lib/dom-dump-script.ts`)。编辑目录或脚本源,重新生成,然后同时提交两者;永远不要编辑生成的文件(`test/design-checklist-sync.test.ts` 在漂移时失败)。

生成使用每个 host 的 `defaultModel`(大多数 host 是 `claude`,Codex 是 `gpt`,Antigravity 的 `agy` 是 `gemini`),除非 `--model` 显式指定。Codex 安装还另外从 `${CODEX_HOME:-~/.codex}/config.toml` 读取顶层模型;改完那个模型后,重跑 `./setup --host codex`。注意:`bun run build` 与裸 `gen-skill-docs --host codex` 会渲染 host 默认模型(gpt)——如果你的 Codex config.toml 钉住了不同的模型,之后请重跑 `./setup --host codex` 以恢复你的 profile(单所有者持久化在 TODOS.md 中跟进)。

技能中的浏览器步骤是 `scripts/resolvers/aside.ts` 中的 `aside repl` 脚本,每个都有回退引擎的 `$B` 等价物。要新增 browse 命令:加到 `browse/src/commands.ts` 并重新构建。要新增 snapshot 标志:加到 `browse/src/snapshot.ts` 的 `SNAPSHOT_FLAGS` 并重新构建。技能模板中的本地 HTML 渲染调用 `bun run ~/.claude/skills/gstack/bin/gstack-render.ts`;新渲染选项加到 `lib/aside-render.ts`(它处理回退),不要加到 skill 自己的 bash 里。

**尺寸预算:** 生成的 SKILL.md 文件超过 160KB(约 40K token)会发出警告;`test/catalog-budget.test.ts` 将始终加载的技能目录上限设为 `CATALOG_BUDGET_TOKEN_EQUIVALENTS`(目前 1,197;每加一个 skill 都会 ratchet 上调);`test/context-budget-ratchet.test.ts` 钉住每个 skill 的 token 上限,与 `test/fixtures/context-budget.json` 对比(对于合理的增长或已落地的缩减,重跑 `bun test/helpers/capture-context-budget.ts` 并在同一 commit 中提交刷新的 fixture)。原理与流程:
[docs/CONTRIBUTOR_REFERENCE.md](docs/CONTRIBUTOR_REFERENCE.md#skill-size-budgets)。

**SKILL.md 文件的合并冲突:** 永远不要通过接受任何一方来解决生成的 SKILL.md 文件的冲突。正确流程:(1) 解决 `.tmpl` 模板和 `scripts/gen-skill-docs.ts`(真相之源)的冲突,(2) 运行 `bun run gen:skill-docs` 重新生成所有 SKILL.md 文件,(3) stage 重新生成的文件。接受一方的生成输出会静默地丢掉另一方的模板变更。

## Platform-agnostic design

技能绝不能硬编码特定框架的命令、文件模式或目录结构。正确做法是:

1. **阅读 CLAUDE.md** 以获取项目特定配置(测试命令、评估命令等)
2. **如果缺失,AskUserQuestion** ——让用户告诉你,或让 gstack 搜索仓库
3. **把答案持久化到 CLAUDE.md**,这样我们永远不用再问

这适用于测试命令、评估命令、部署命令,以及任何其他项目特定行为。项目拥有自己的配置;gstack 读取它。

## Writing SKILL templates

SKILL.md.tmpl 文件是**供 Claude 读取的提示模板**,不是 bash 脚本。每个 bash 代码块在独立的 shell 中运行——变量不会在块之间持久化。

规则:
- **用自然语言表达逻辑和状态。** 不要用 shell 变量在代码块之间传递状态。改为告诉 Claude 要记住什么,并在散文中引用它(例如,「在 Step 0 检测到的基分支」)。
- **不要硬编码分支名。** 通过 `gh pr view` 或 `gh repo view` 动态检测 `main`/`master`/等。对 PR 目标 skill 使用 `{{BASE_BRANCH_DETECT}}`。在散文中用「the base branch」,在代码块占位符中用 `<base>`。
- **保持 bash 块自包含。** 每个代码块应能独立工作。如果一个块需要上一步的上下文,在上方的散文中重新陈述它。
- **把条件分支表达为英语。** 不要在 bash 中写嵌套的 `if/elif/else`,而是写编号的决策步骤:「1. 如果 X,做 Y。2. 否则,做 Z。」

## Writing style (V1)

每个 tier-≥2 的技能默认输出遵循 `scripts/resolvers/preamble/generate-writing-style.ts` 中的 Writing Style 部分:首次出现术语时加注释(`scripts/jargon-list.json` 中的精选列表,skill 在运行时遇到第一个术语时 Read 它),问题以结果形式("what breaks for your users if...")而非实现形式措辞,短句,决策以用户影响收尾。希望更紧凑的 V0 文体的资深用户可设 `gstack-config set explain_level terse`(二进制开关,无中间模式)。完整设计原理见 `docs/designs/PLAN_TUNING_V1.md`。原本想与 writing-style 并肩推进的 review 节奏整改被抽出为 V1.1——见 `docs/designs/PACING_UPDATES_V0.md`。

## Browser interaction

gstack 优先驱动 Aside AI 浏览器(macOS 15+),当 Aside 缺失时回退到自己的浏览器引擎。当你需要与浏览器交互(QA、dogfooding、检查页面)时,使用 `/browse` skill:它探查 Aside,当 `READY` 时驱动它——用户的真实浏览器与真实 session——通过遵循 `scripts/resolvers/aside.ts`(`{{ASIDE_SETUP}}`)中契约的 `aside repl` 脚本。每个浏览器 skill(`/qa`、`/qa-only`、`/design-review`、`/canary`、`/benchmark`、`/scrape`)都做同样动作,技能中的 web research 走 Aside 的 agent(`{{ASIDE_RESEARCH}}`),在 WebSearch 工具之前。当探查返回 `NEEDS_ASIDE` 或 `ASIDE_NOT_RUNNING`(Linux、Windows、Aside app 关闭)时,skill 解析 `$B`(按 `{{BROWSE_FALLBACK}}`)并改跑 browse 二进制——`$B <command>` 在该上下文中是合法工具,cookie 导入、GStack Browser headed 模式、`/pair-agent`、browser-skills/`/skillify` 都属于它。skill 自己生成的本地 HTML(make-pdf、diagram、design previews)通过 `bin/gstack-render.ts` / `lib/aside-render.ts` 渲染,该工具在 loopback 上服务文件并在 Aside 中打印或截图,或当 Aside 缺席时在引擎中——永远不要把渲染器指向一个网站。永远不要使用 `mcp__claude-in-chrome__*` 工具——它们慢、不可靠,并且不是本项目所使用的。

**Server / sidebar / extension internals:** 在编辑 `browse/src/server.ts`、`extension/`、sidebar PTY、任何 SSE 端点或 CDP session 代码之前,先读 [docs/BROWSER_INTERNALS.md](docs/BROWSER_INTERNALS.md)——sidebar 消息流、WebSocket 鉴权、隧道双监听规则、出口处的 Unicode 净化、SSE/CDP 辅助、setup 符号链接硬化,以及 sidebar 安全栈都在那里,每条都被一个 CI tripwire 钉住。

**每个离机出口的出账回执(v1.63.0.0+)。** 每个 gstack 主动发起的离机发送必须在发送前写一条回执:TypeScript 调用方使用 `lib/egress-receipt.ts` 中的 `writeReceipt`;shell 脚本 source `bin/gstack-egress-lib.sh` 并使用 `_receipted_curl` / `_receipted_git`。`test/egress-receipt-wiring.test.ts` 在出现未回执的新 sink 时让 CI 失败。失败极性、扫描器豁免与 `_aside_exec` 包装器:
[docs/CONTRIBUTOR_REFERENCE.md](docs/CONTRIBUTOR_REFERENCE.md#egress-receipts)。

## Dev symlink awareness

在开发 gstack 时,`.claude/skills/gstack` 可能是一个指回本工作目录的符号链接(gitignored)。这意味着 skill 的变更**立即生效**,有利于快速迭代,但在大型重构中风险较高——写了一半的 skill 可能会破坏同时使用 gstack 的其他 Claude Code 会话。

**每个会话检查一次:** 运行 `ls -la .claude/skills/gstack` 看看它是符号链接还是真实副本。如果它是指向你工作目录的符号链接,请注意:
- 模板变更 + `bun run gen:skill-docs` 会立即影响所有 gstack 调用
- 对 SKILL.md.tmpl 文件的破坏性变更可能破坏并发的 gstack 会话
- 在大型重构期间,删除该符号链接(`rm .claude/skills/gstack`),改用 `~/.claude/skills/gstack/` 的全局安装

**Skill 链接与所有权(#2119):** setup 将每个 skill 安装为顶层目录(简短的 `qa` 或带命名空间的 `gstack-qa`,依 `skill_prefix`),`setup` 和 `bin/gstack-relink` 仅删除或链接能证明是 gstack 拥有的条目;保持这两份规则同步。证明规则、清理行为与钉住测试:
[docs/CONTRIBUTOR_REFERENCE.md](docs/CONTRIBUTOR_REFERENCE.md#skill-linking-and-ownership)。

**注意:** 将 gstack 作为 vendored 副本塞进项目仓库已被弃用。改用全局安装 + `./setup --team`。参见 README.md 中关于 team mode 的说明。

**对于 plan review:** 当审阅修改 skill 模板或 gen-skill-docs 流水线的计划时,考虑这些变更是否应先在隔离环境中测试(尤其当用户正在其他窗口中活跃使用 gstack 时)。

**升级迁移:** 当一项变更以可能破坏既有用户安装的方式修改了磁盘上的状态(目录结构、配置格式、过期文件),请在 `gstack-upgrade/migrations/` 中添加迁移脚本。格式与测试要求请阅读 CONTRIBUTING.md 的「Upgrade migrations」一节。升级 skill 在 `/gstack-upgrade` 期间于 `./setup` 之后自动运行这些迁移。

## Compiled binaries — never commit browse/dist/, design/dist/, or make-pdf/dist/

`browse/dist/`、`design/dist/` 与 `make-pdf/dist/` 目录包含编译后的 Bun 二进制(`browse`、`find-browse`、`design`,每个约 62MB)。这些是 Mach-O arm64 only——它们无法在 Linux、Windows 或 Intel Mac 上运行。`./setup` 脚本会为每个平台从源码构建。

这些目录**未被追踪并被 gitignore**(`.gitignore:3-6`;`browse/dist/` 二进制在 `64d5a3e4`、v0.11.16.0 未被追踪;其它则从未被追踪)。它们不会出现在 `git status` 中。如果某个 dist 二进制出现在 `git status` 中,说明有人强制添加了它(`git add -f`)——不要提交它;撤销 stage 并查明它是怎么进来的。

在 stage 文件时,始终使用具体文件名(`git add file1 file2`)——不要用 `git add .` 或 `git add -A`,它们会把构建产物和垃圾也卷进去。

## Redaction guard (PII / secrets / legal content)

`lib/redact-patterns.ts` + `lib/redact-engine.ts`(CLI `bin/gstack-redact`、opt-in hook `bin/gstack-redact-prepush`)在任何外部 sink 之前捕获凭据、PII 与法律内容。它是**护栏,不是密不透风的强制**;永远不要声称它能阻止一个决心泄漏的人。始终扫描将要发送的确切字节,并从 `scripts/resolvers/redact-doc.ts` 渲染技能文档。分层、可见性、围栏与配置键:
[docs/CONTRIBUTOR_REFERENCE.md](docs/CONTRIBUTOR_REFERENCE.md#redaction-guard)。

## Commit style

**始终 bisect 提交。** 每个 commit 应当是一个逻辑变更。当你做了多处变更(例如,重命名 + 重写 + 新测试),将它们拆成独立的 commit 再推送。每个 commit 应当可独立理解与回退。

良好 bisect 的例子:
- 重命名/移动 与行为变更分开
- 测试基础设施(touchfiles、helpers) 与测试实现分开
- 模板变更 与生成文件的重新生成分开
- 机械性重构 与新功能分开

当用户说「bisect commit」或「bisect and push」,将已 stage 或未 stage 的变更拆成逻辑 commit 然后推送。

## Slop-scan: AI code quality, not AI code hiding

我们使用 [slop-scan](https://github.com/benvinegar/slop-scan) 来捕获「AI 生成的代码确实比人类写的差」的模式。我们**不是**要冒充人类代码。我们是 AI 编写的并以此为荣。目标是代码质量。

```bash
npx slop-scan scan .          # human-readable report
npx slop-scan scan . --json   # machine-readable for diffing
```

配置:仓库根的 `slop-scan.config.json`(目前排除 `**/vendor/**`)。

在修复任何发现之前,先读 [docs/SLOP_SCAN.md](docs/SLOP_SCAN.md):它将真正的质量修复(文件操作周围的空 catch → `safeUnlink()`、进程 kill → `safeKill()`)与我们拒绝的「让 linter 过关」操作(字符串匹配错误信息、放宽尽力而为的清理)区分开。工具位于 `lib/error-handling.ts`。不要追逐分数。

## Community PR guardrails

在评审或合并社区 PR 时,**始终 AskUserQuestion** 在接受任何属于下列的 commit 之前:

1. **触及 ETHOS.md** ——该文件是 Garry 的个人构建者哲学。外部贡献者或 AI 代理的任何编辑一律不行,没有例外。
2. **删除或软化推广性材料** ——YC 引用、founder 视角、产品声音都是刻意的。将它们框定为「不必要」或「过于推广」的 PR 必须被拒绝。
3. **改变 Garry 的声音** ——技能模板、CHANGELOG 与文档中的语气、幽默、直接性、视角不是泛泛的。将声音重写得更「中性」或「专业」的 PR 必须被拒绝。

即使代理强烈相信某项变更能改进项目,这三类仍需通过 AskUserQuestion 获得显式用户批准。没有例外。不自动合并。不接受「我就清理一下」。

## Checking out PRs from garrytan-agents

Fork PR(例如来自 `garrytan-agents/gstack`)拿不到基仓库的 secrets,所以 eval CI 会因空 env 鉴权失败。除非用户说要让它保持 fork PR 状态,否则将分支推到 `garrytan/gstack` 并从那里重新指向 PR:
[docs/CONTRIBUTOR_REFERENCE.md](docs/CONTRIBUTOR_REFERENCE.md#checking-out-prs-from-garrytan-agents)。

## CHANGELOG + VERSION style

**自主选择版本;默认 PATCH。** Garry 将发布版本决策委托给代理。不要请他选择或批准版本,包括当一个已经批准的版本与另一个 PR 冲突时。该策略覆盖 `/ship` 与 `/document-release` 中通用的版本审批提示。

对普通发布(包括修复、新增、重构、测试基础设施与协调的多文件工作)优先选择 **PATCH (X.Y.Z+1.0)**。仅差异大小本身不是选择 MINOR 的理由。仅当把一个发布称为 patch 会明显误导时,才选 **MINOR (X.Y+1.0.0)** 或 **MAJOR (X+1.0.0.0)**,例如不兼容的公开接口变更或真正新的产品级发布。做出该判断时不需要再问一次。

发布前用 `bin/gstack-next-version` 检查活跃发布队列。如果一个 slot 已被认领,推进到所选 bump 级别的下一个可用版本,并用 `bin/gstack-version-bump` 同步发布元数据。基版本更高本身不需要 MINOR bump。让 PR 保持可合并状态,等待 Garry merge;自主版本决策不授权合并、部署或跳过必需的验证。

**VERSION 和 CHANGELOG 是分支范围的。** 每个发布分支获得自己的 bump 与一个条目,在 `/ship` 时写入,描述**这个**分支相对于 main 加了什么,面向用户,没有分支内部版本或开发叙事。VERSION 保持 4 位;package.json 携带 `bin/gstack-version-bump` 拥有的 3 位 npm 翻译,永远不要手动编辑它。在写条目之前请阅读 [docs/CHANGELOG_STYLE.md](docs/CHANGELOG_STYLE.md):它包含版本不变性、merge-main 检查、不被纳入的内容以及条目格式。始终以 `Contributed by @username` 标注社区贡献。

## AI effort compression

当估算或讨论工作量时,始终同时展示人类团队与 CC+gstack 的时间:

| 任务类型 | 人类团队 | CC+gstack | 压缩比 |
|---------|---------|-----------|--------|
| 脚手架 / 模板代码 | 2 天 | 15 分钟 | ~100x |
| 测试编写 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| Bug 修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 研究 / 探索 | 1 天 | 3 小时 | ~3x |

完整性很便宜。当完整实现可行时,不要推荐捷径。煮沸整片海洋——完整的事才是目标;只有真正不相关的跨季度迁移才分开算,从来不是走捷径的借口。完整哲学见 skill 序章中的 Completeness Principle。

## Search before building

在设计任何涉及并发、陌生模式、基础设施或运行时/框架可能有内建能力的方案之前:

1. 搜索「{runtime} {thing} built-in」
2. 搜索「{thing} best practice {current year}」
3. 查阅官方运行时/框架文档

三层知识:久经验证(第一层)、新而流行(第二层)、第一性原理(第三层)。把第三层置于最高。完整构建者哲学见 ETHOS.md。

## Local plans

贡献者可在 `~/.gstack-dev/plans/` 中存储长期愿景文档与设计文档。这些是本地的(不入库)。在评审 TODOS.md 时,检查 `plans/` 中是否有可提升到 TODOs 或实施的候选。

## E2E eval failure blame protocol

当 E2E eval 在 `/ship` 或任何其他工作流中失败时,**未经证明永远不要声称「与我们的变更无关」**。这些系统存在隐性耦合——一段序章文本变更会影响代理行为,一个新增 helper 改变时序,重新生成的 SKILL.md 改变提示上下文。

**在将失败归因为「pre-existing」之前,需要:**
1. 在 main(或基分支)上跑同一 eval 并显示它也失败
2. 如果它在 main 上通过但在分支上失败——那**就是**你的变更。追责。
3. 如果你在 main 上跑不了,说「未验证——可能相关也可能无关」,并在 PR body 中将其标记为风险

「Pre-existing」没有证据是懒惰的声明。证明它,或别说。

## Running evals as an agent: always detach (SIGTERM-proof)

永远不要把长 eval 作为普通的 backgrounded Bash 任务运行:turn-boundary SIGTERM 会中途杀掉它。后台运行付费评估用 `bun run eval:bg:pr`(diff-selected PR gate)或 `bun run eval:bg:release`(完整 gate + periodic);`eval:bg:gate` 与 `eval:bg:periodic` 跑单个 tier。这四个都跑 `scripts/eval-bg.ts`,它会打印 `[eval-bg] backend=<dispatch|local> …` 和 `gstack-detach LOG <path>`,然后立即返回。

- **Dispatch 后端**(默认,当 HEAD 干净且推送到 garrytan/gstack 且 `gh` 能 dispatch):CI 在该确切 revision 上跑该 lane(`evals.yml` 和/或 `evals-periodic.yml`,由 `expected_sha` 验证);本地日志跟踪该运行并命名其 URL。派发的运行为验证运行:新鲜,从不写 PR 回执。先 commit 并 push;未提交的文件或未 push 的 commit 回退到 local,日志说明原因。
- **Local 后端**(脏树、未推送的 commit、fork、没有 `gh`、或 `--local`):在这台机器上的分片 runner,上限为 ceil(1.5 × 计划串行秒数 / EVALS_JOBS) + 20 分钟,最多 4 小时(`--timeout SECS` 覆盖)。先 export `ANTHROPIC_API_KEY`(永远不要在 argv 中传 key)。
- **强制:** `--dispatch` 拒绝并指出修复方法(「commit 并 push,或用 `--local` 重跑」),不回退;`--local` 或 `GSTACK_EVAL_BG_MODE=local` 始终在这里跑。
- **等待:** 轮询日志直到 `### gstack-detach EXIT=<code> ###`(0 通过,1 失败,130 取消,2 其它);沉默不等于成功。持续检查直到它出现或用户让你停止,并在每次检查时报告进度。`bun run scripts/eval-bg.ts status <log-or-run-id>` 打印 running、passed、failed、cancelled 或 incomplete(无 sentinel 死亡)。
- 两个后端都持有机器级 `gstack-evals` 锁,所以并发的 worktree 排队而不是饱和 API;日志位于 `~/.gstack-dev/eval-runs/` 下。同 revision 与基的第二次 `eval:bg:<lane>` 跟随排队、运行中或绿色的派发,而不是再次派发。

人类在自己的终端跑 eval 不需要这个。后端、上限与旋钮:
[docs/TESTING_INTERNALS.md](docs/TESTING_INTERNALS.md#running-evals-as-an-agent-detach)。

## E2E test fixtures: extract, don't copy

**永远不要把整个 SKILL.md 文件复制到 E2E 测试 fixture。** SKILL.md 文件有 1500-2000 行。当 `claude -p` 读取那么大的文件时,上下文膨胀会导致超时、turn 限制抖动,以及耗时是必要 5-10 倍的测试。

改为只提取测试实际需要的那一节:

```typescript
// BAD — agent reads 1900 lines, burns tokens on irrelevant sections
fs.copyFileSync(path.join(ROOT, 'ship', 'SKILL.md'), path.join(dir, 'ship-SKILL.md'));

// GOOD — agent reads ~60 lines, finishes in 38s instead of timing out
const full = fs.readFileSync(path.join(ROOT, 'ship', 'SKILL.md'), 'utf-8');
const start = full.indexOf('## Review Readiness Dashboard');
const end = full.indexOf('\n---\n', start);
fs.writeFileSync(path.join(dir, 'ship-SKILL.md'), full.slice(start, end > start ? end : undefined));
```

在跑定向 E2E 测试以调试失败时,同样适用:
- 在**前台**跑(`bun test ...`),不要用 `&` 与 `tee` 后台
- 永远不要 `pkill` 正在运行的 eval 进程再重启——你会丢失结果并浪费钱
- 一次干净运行胜过三次 kill-then-restart 运行

## Publishing native OpenClaw skills to ClawHub

原生 OpenClaw 技能位于 `openclaw/skills/gstack-openclaw-*/SKILL.md`。命令是 `clawhub publish`(不是 `clawhub skill publish`)——完整流程、鉴权与验证:
[docs/OPENCLAW_PUBLISHING.md](docs/OPENCLAW_PUBLISHING.md)。

## Deploying to the active skill

活跃 skill 位于 `~/.claude/skills/gstack/`。做出变更后:

1. 推送你的分支
2. 在 skill 目录中 fetch 并 reset:`cd ~/.claude/skills/gstack && git fetch origin && git reset --hard origin/main`
3. 重新构建:`cd ~/.claude/skills/gstack && bun run build`

`bun run build` 在一次从未跑过该 host 的 checkout setup 中渲染每个 host。在安装(任何带 `.gstack-installed-hosts` 的 checkout,包括用 `bin/dev-setup` 配置的、记录了 Claude 和 Codex 的安装)中,它只渲染记录的 host 并打印它们(#1694);`GSTACK_RENDER_HOSTS=all bun run build` 渲染那里所有 host,下一次 `./setup` 会把额外的渲染再次清理到 `$GSTACK_STATE_ROOT/backups/host-renders/`。

**如果你使用 gbrain:** 步骤 2 中的 `git reset --hard` 会回滚 `gstack-config gbrain-refresh` 渲染到安装中的脑感知(`GBRAIN_CONTEXT_LOAD` / `GBRAIN_SAVE_RESULTS`)块(这些生成块与 `main` 的差异是刻意的)。部署后,重跑 `gstack-config gbrain-refresh` 以恢复它们到你所有项目的 Claude 会话中。它是幂等的。

或直接复制二进制:
- `cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse`
- `cp design/dist/design ~/.claude/skills/gstack/design/dist/design`
- `cp make-pdf/dist/pdf ~/.claude/skills/gstack/make-pdf/dist/pdf`

## Skill routing

当用户请求匹配某个可用 skill 时,通过 Skill 工具调用它。只路由到会话可用 skill 列表中的 skill;对快速问题或小范围编辑直接回答。

关键路由规则:
- 产品创意/头脑风暴 → invoke /office-hours
- 战略/范围 → invoke /plan-ceo-review
- 架构 → invoke /plan-eng-review
- 设计系统/计划评审 → invoke /design-consultation 或 /plan-design-review
- 完整评审流水线 → invoke /autoplan
- Bug/错误 → invoke /investigate
- QA/测试站点行为 → invoke /qa 或 /qa-only
- 代码评审/diff 检查 → invoke /review
- 视觉打磨 → invoke /design-review
- 部署/发版/PR → invoke /ship 或 /land-and-deploy
- 保存进度 → invoke /context-save
- 恢复上下文 → invoke /context-restore

## Cross-session decision memory

耐久决策及其理由被捕获到一个 append-only、event-sourced 存储,位于 `~/.gstack/projects/<slug>/decisions.jsonl`,这样你或用户都不会在会话间重新挑起一个已定的决定或丢失「为什么」。这是可靠、仅依赖文件的路径:它在 gbrain 关闭时也能工作。(gbrain 语义召回是可选的增强层,而不是依赖。)

- **Resurface** 在重新决定之前先列出活动决策:`bin/gstack-decision-search`(`--recent N`、`--scope repo|branch|issue`、`--query KW`、`--all`、`--json`)。加 `--semantic`(与 `--query` 一起)以在 gbrain 在线时追加来自 gbrain 记忆的相关命中;它在 gbrain 关闭时静默回退到可靠的文件结果。会话开始时,Context Recovery 已经呈现 scope 相关的活动决策。如果某项决策已列出,把它当作带理由的已定决策;如果你要反转它,明确说出来。
- **Capture** 一个耐久决策,当您或用户做出一个时:`bin/gstack-decision-log '{"decision":"...","rationale":"...","scope":"repo|branch|issue","source":"user|skill|agent","confidence":1-10}'`。用 `--supersede <id>` 反转之前的决定;用 `--redact <id>` 抹除意外泄漏的 secret;用 `--compact` 将日志重写为活动集。非交互式(从不提示)、注入净化,且写入时 HIGH-secret 阻断。
- **耐久意味着:** 架构选择、范围裁剪、工具/供应商选择,或对之前决定的反转。不是回合级编辑、措辞微调,或任何平凡可重新推导的内容。Capture 在源头被策展——只记录耐久决策,否则存储会变成噪声。

## GBrain Search Guidance (configured by /sync-gbrain)
<!-- gstack-gbrain-search-guidance:start -->

GBrain 已在本机上设置并同步。代理在问题是语义性的或你尚不知道确切标识符时,应优先选择 gbrain 而非 Grep。

**本 worktree 通过仓库根的 `.gbrain-source` 文件被钉到 worktree 范围的代码源**(kubectl-style context)。从本 worktree 下任何地方调用的 `gbrain code-def`、`code-refs`、`code-callers`、`code-callees` 或 `query` 默认路由到该源,无需 `--source` 标志。同 repo 的 Conductor 兄弟 worktree 各自有自己的 pin 与自己的索引页,因此语义结果匹配本 worktree 磁盘上的实际代码。

通过 `gbrain` CLI 可用的两个已索引语料库:
- 本 worktree 的代码(通过 `.gbrain-source` 自动 pin)。
- `~/.gstack/` 策展记忆(通过现有 federation 流水线注册为 `gstack-brain-<user>` 源)。

在以下场景优先选择 gbrain:
- 「X 在哪里处理?」/ 语义意图,尚无确切字符串:
  `gbrain search "<terms>"` 或 `gbrain query "<question>"`
- 「符号 Y 在哪里定义?」/ 基于符号的代码问题:
  `gbrain code-def <symbol>` 或 `gbrain code-refs <symbol>`
- 「谁调用 Y?」/ 「Y 依赖谁?」:
  `gbrain code-callers <symbol>` / `gbrain code-callees <symbol>`
- 「上次我们决定了什么?」/ 过去的计划、retros、learnings:
  `gbrain search "<terms>" --source gstack-brain-<user>`

Grep 仍然适用于已知确切字符串、正则、多行模式、文件 glob。在有意义的代码变更后运行 `/sync-gbrain`;如需跨所有 worktree 的持续自动同步,在每台机器上运行一次 `gbrain autopilot --install`——gbrain 的守护进程按计划处理增量刷新。

安全:在 `gbrain autopilot` 活跃时不要运行 `/sync-gbrain`——调度器在检测到 autopilot 运行时拒绝破坏性源操作,以避免与之竞速(#1734)。优先用 `gbrain sources add --path <dir>`(无 `--url`)注册用户仓库:URL 管理的源会自动 reclone,这些源的代码走查需要显式的 `--allow-recline` 选项。

<!-- gstack-gbrain-search-guidance:end -->
