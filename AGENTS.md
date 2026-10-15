# gstack — AI Engineering Workflow

gstack 是一组 SKILL.md 文件,赋予 AI 代理结构化的软件工程角色。每个 skill 是一位专家:CEO 评审、工程经理、设计师、QA 负责人、发布工程师、调试者,以及更多。

## Available skills

技能位于 `.agents/skills/`(或在 Claude Code 上的 `~/.claude/skills/gstack/`)。通过名称调用(例如,`/office-hours`)。

### Plan-mode reviews

| Skill | What it does |
|-------|-------------|
| `/office-hours` | 从这里开始。在你写代码之前重塑你的产品创意。 |
| `/plan-ceo-review` | CEO 级评审:在请求中找到 10 颗星的产品。 |
| `/plan-eng-review` | 锁定架构、数据流、边界情况与测试。 |
| `/plan-design-review` | 对每个设计维度打 0-10 分,解释 10 分长什么样。 |
| `/plan-devex-review` | DX 模式评审:TTHW、魔幻时刻、摩擦点、persona traces。 |
| `/plan-tune` | 自我调优每个问题对应的 AskUserQuestion 敏感度。 |
| `/autoplan` | 一条命令跑完 CEO → design → DX → eng 评审(eng 永远在最后)。 |
| `/design-consultation` | 从零开始构建一个完整的设计系统。 |
| `/spec` | 把模糊的意图转成精确、可执行的 spec,分五个阶段。归档一个 GitHub issue,可选地在新 worktree 中孵化一个 Claude Code 代理,并让 `/ship` 在合并时关闭源 issue。 |

### Implementation + review

| Skill | What it does |
|-------|-------------|
| `/review` | 落地前 PR 评审。找到那些通过 CI 但在生产中崩溃的 bug;列出每次 gate 编辑以保证 gate 完整性。 |
| `/deslop-shared-libs` | 在最近的工作中寻找有价值的共享代码抽取。仅推荐。 |
| `/test-audit` | 扫一遍现有测试,找出低价值、与实现耦合或重复的测试。仅报告,除非你批准一批。 |
| `/codex` | 通过 OpenAI Codex 获得第二意见。Review、challenge 或 consult 模式。也可在 Codex 沙箱外使用。 |
| `/claude-code` | 通过 Claude Code 获得第二意见。Review、challenge 或 consult 模式。也可在 Claude Code 沙箱外使用。 |
| `/investigate` | 系统化的根因调试。没有调查就没有修复。 |
| `/design-review` | 在线站点的可视化审计 + 修复循环,带原子提交。 |
| `/design-shotgun` | 生成多种 AI 设计变体、对比板、迭代。 |
| `/design-html` | 生成生产质量的 Pretext-native HTML/CSS。 |
| `/devex-review` | 在线开发者体验审计(TTHW 与真实流程对照)。 |
| `/qa` | 测试浏览器、API、CLI、job、worker 与 webhook 行为;复现 bug,修复并重新验证。 |
| `/qa-only` | 同样的 surface-aware QA,产出发现与提议的测试,不修改产品代码或测试。 |
| `/scrape` | 在你的 Aside 浏览器中、用你真实登录态,从网页拉取数据。只读。在回退浏览器上,一个固化(browser-skill)的浏览器技能可在约 200ms 内回答重复的意图。 |
| `/skillify` | 把最近一次成功的 `/scrape` 流程固化为永久的 browser-skill(仅回退浏览器)。 |

### Release + deploy

| Skill | What it does |
|-------|-------------|
| `/ship` | 跑测试、评审、gate 完整性、推送、开 PR。工作区感知的版本队列。 |
| `/land-and-deploy` | 合并 PR,等待 CI 与部署,验证生产健康。 |
| `/canary` | 部署后监控循环(在你的 Aside 浏览器中,或当 Aside 缺席时在 gstack 自家浏览器中)。 |
| `/landing-report` | 工作区感知的 ship 队列只读仪表板。 |
| `/document-release` | 在每次 ship 之前审计相关文档,支持独立文档更新。 |
| `/document-generate` | 从代码生成 Diataxis 文档(tutorial / how-to / reference / explanation)。 |
| `/setup-deploy` | 一次性部署平台检测(Fly.io、Render、Vercel 等)。 |
| `/gstack-upgrade` | 将 gstack 升级到最新版本。 |

### Operational + memory

| Skill | What it does |
|-------|-------------|
| `/context-save` | 保存工作上下文(git 状态、决策、剩余工作)。 |
| `/context-restore` | 从已保存的上下文中恢复,即使跨 Conductor 工作区。 |
| `/learn` | 管理 gstack 在跨会话中学到的东西。 |
| `/retro` | 每周复盘,带按人拆分与发布 streak。 |
| `/health` | 代码质量仪表板(类型检查、linter、测试、dead code)。 |
| `/benchmark` | 性能回归检测(页面加载、Core Web Vitals)。 |
| `/benchmark-models` | 跨模型 benchmark 技能(Claude、GPT、Gemini 并排)。 |
| `/eval-plan` | 预注册一个付费 eval(bars、arms、decoys、stop rules),以 0 美元 dry-run,试点三个定价项,然后批准花费。 |
| `/cso` | 支持的安全发现,带显式覆盖。静态评估无需 catalog profiles 即可用;含运行时/扫描器执行的 catalog 需要匹配的合格 profile,且目前没有 catalog 拥有该 profile(扫描器构建输入已审阅,等待它们的合格运行;运行时合格需要一个仍在构建中的私有评估器),所以 `/cso` 仅跑静态评估。经过运行时测试的 bundle 分别认证外部断言。项目测试完成仍是 `self_reported`,因为目标代码控制测试过程;`tested` 为未来的目标无关完成见证保留。 |
| `/setup-gbrain` | 设置 gbrain 用于跨机器会话记忆同步。 |
| `/sync-gbrain` | 让 gbrain 与本仓库代码保持同步;在 CLAUDE.md 中刷新代理搜索指引。 |

### Browser + agent integration

每个浏览器 skill 优先驱动 Aside AI 浏览器(macOS 15+,aside.com)——用户的真实浏览器,带他们的真实 session,通过 `aside repl` 脚本;gstack 永不安装它。当 Aside 未安装或未运行(Linux、Windows、关闭的 Aside app)时,同一 skill 自动回退到 gstack 自己的无头 Chromium(`$B`),`/browse` 之下的三个技能即在该处应用。

| Skill | What it does |
|-------|-------------|
| `/browse` | 驱动浏览器:打开页面、读取、点击完成一个流程、截图、控制台错误。Aside 优先;gstack 自家 Chromium(每命令约 100ms)作为回退。其他所有浏览器 skill 都基于它。 |
| `/open-gstack-browser` | 启动可见的 GStack Browser,带 sidebar + stealth——回退引擎的有头面。 |
| `/setup-browser-cookies` | 从你真实浏览器导入 cookies 到回退引擎,以便做带身份验证的测试。在 Aside 上不需要。 |
| `/pair-agent` | 让远程 AI 代理(OpenClaw、Codex 等)通过作用域隧道与 gstack 自己的浏览器配对。 |

### iOS QA — 通过 USB 或 Tailscale 驱动真实 iPhone(v1.43.0.0+)

| Skill | What it does |
|-------|-------------|
| `/ios-qa` | 通过 USB CoreDevice 隧道 + 内嵌 StateServer 的真机 iOS QA。可选地通过 Tailscale 暴露设备,让远程代理也能驱动它。 |
| `/ios-fix` | 带回归快照捕获的自治 iOS bug 修复器。 |
| `/ios-design-review` | 设计师视角下的真机 iPhone QA——10 维 Apple HIG 评分。 |
| `/ios-clean` | 便利工具:在 Release 构建之前剥离 DebugBridge + `#if DEBUG` 连线。 |
| `/ios-sync` | 重新生成 iOS debug bridge 以匹配最新上游模板。 |

伴侣 CLI(在插着设备的 Mac 上运行):

| Command | What it does |
|---------|-------------|
| `gstack-ios-qa-daemon` | Mac 端 broker。默认 loopback;`--tailnet` 添加 Tailscale-facing 监听器,带能力分级与审计日志。 |
| `gstack-ios-qa-mint` | tailnet allowlist 的所有者授权 CLI(`grant`/`revoke`/`list`)。 |
| `gstack-ios-qa-regen` | 重新生成规范的本地 DebugBridge 包与类型化访问器(`--app-source` / `--bridge-dir`)。 |

端到端走查:[docs/howto-ios-testing-with-gstack.md](docs/howto-ios-testing-with-gstack.md)。

### Safety + scoping

| Skill | What it does |
|-------|-------------|
| `/careful` | 在破坏性命令之前警告(rm -rf、DROP TABLE、force-push)。 |
| `/freeze` | 把编辑锁定到一个目录。硬阻断,不只是警告。 |
| `/guard` | 一次性同时激活 careful + freeze。 |
| `/unfreeze` | 解除目录编辑限制。 |
| `/make-pdf` | 把任何 markdown 文件转成出版物级 PDF。通过 Aside 渲染,或当 Aside 缺席时用 gstack 自家浏览器。 |
| `/diagram` | 英语进、图出:mermaid 源 + 可编辑的 .excalidraw + SVG/PNG,离线。通过 Aside 渲染,或当 Aside 缺席时用 gstack 自家浏览器。 |

## Validation discipline

在修复失败或准备 `/ship` 时,遵循这个顺序:

1. 列出已知失败的用例、它们的日志与源 revision、已证明的根因,以及能证明每项修复的最小检查。保留一份当前列表于 `.context/`;更新它而不是启动重叠的修复计划。把 runner 的失败总数与命名失败、未处理或模块加载错误对齐;仅命名测试的页脚并非完整清单。
2. 在编辑前解决基分支集成并为每个共享文件指派一个所有者。把修复限制在已观察到的失败与用户范围内。在 fixture 通过链接写入前,解析其目标并验证它停留在该 fixture 的临时根目录之内;live skill 注册可能指回本 checkout。当上游替换 helper API,清点所有直接调用者、模拟适配器、源快照、生成的 golden 与选择边界,再选择聚焦检查。验证抽取的测试适配器提供当前的 imports 与结果 schema;适配器失败不是生产失败的证据。独立地调度独立检查。只在检查真正需要的输入或前置条件上 gate 该检查;不相关的失败不应序列化整个验证计划。在测试 live-link 源文件时保持源固定。
3. 改代码之前先诊断。区分产品缺陷、无效的测试期望、检测器/fixture 缺陷与启动/环境失败。保留原始失败。没有证据不要称其为 pre-existing。验证被钉住的运行时工具 schema 与默认,再把遗漏字段当作模型不合规来处理。检查受限评估的 fixture 范围与自动答案是否支撑它的度量。不要让 driver 批准不相关的扩展,然后把额外工作归咎于 skill;保留必需的发现与证据限制。
4. 用最小相关测试复现。对于代理测试,在免费回归中复用已捕获的公开事件,包括负面控制,再为另一次代理运行付费。检查行为与确认;仅当那段散文就是契约时,才匹配确切散文。不要降低阈值、增加模型预算、跳过用例、或重判一个失败以制造一个通过。预注册的固定面板不是重判。对于策略或验证修复,用有代表性的本地输入练习实际注册的回调,并断言它使用了 helper 的结果。当渲染器或解析器的失败在同一边界反复出现,验证被支持的输入类与钉住的运行时是否一致。保留对抗性控制;不要为每个付费失败添加一个拼写或字形。对于工作流清晰度失败,读完整的被评估摘录与它引用的源。一次解决所有已证明的歧义:顺序、定义、所有权与批准。把密集的指令合并为可执行步骤,而不是追加更多子句。整体复审产生的工作流;仅有散文快照不能证明它是清晰的。对于每个 gate,识别它的输入何时存在,并跟踪正常、跳过/不可用与迟到变更路径,以捕获循环前置条件或被绕过的检查。
5. 在付费工作前跑必需的廉价 CI 检查,包括凭据扫描。同时跑邻近的廉价检查:生成内容新鲜度、提示大小/奇偶性限制、源断言、fixture 检查与依赖选择(如适用)。变更过的提示必须在它的 eval 之前通过这些检查。对于 skill 编辑,包含 `bun test test/parity-suite.test.ts`:它的历史 union-size 上限独立于其他提示大小与上下文预算。当工作流措辞变更时,在整个测试树中搜索被删除的子句,包括始终加载的提示卫语句。包含子进程示例的测试 fixture 必须通过 `test/spawnsync-timeout-tripwire.test.ts`;它的扫描器也检查引用代码。在读同一份变更过的提示的长行为评估之前,跑选定的质量裁判。如果一次修复取代了一次活跃运行的输入,取消该运行,保留已完成的结果,并把未完成的用例标记为 cancelled。在跑依赖检查之前检查每次编辑或 setup 命令的结果。失败的编辑不是再跑一次未变更输入的理由。
6. 在模型开始前声明 fixture actor 的支持交互。保持它的答案与权限处理在被声明的接口内。将产物所有权与传给子进程的同一隔离状态绑定;环境路径不建立所有权。针对每个用例和配置的 retry,检查整文件与 CI 监督,而不是只检查一次。预检实际的启动器:必需的二进制、隔离状态、需要时的显示、显式的测试 tier、选择与预期的已执行用例数。匹配工作流与其容器镜像钉住的运行时版本。在 runner 加入其嵌套目录后,保持带 socket 的临时路径短;在该 smoke 检查中练习那个精确的布局。把长生命周期的日志与 socket 目录分开存放。在那个启动环境下用零成本 smoke 检查验证必需的工具执行;仅有版本与鉴权不足以证明它可用。显式设置私有产物模式并保留正常的 fixture 权限。在付费工作前,证明诊断快照在最终产物目录中存活于 fixture 清理之后;原生快照需要 EVALS_RUN_ID 或 GSTACK_EVAL_DIR。绑定完整的 spool 文件名,并把 Bun 的越界 describe.skip 占位符单独分类,选中用例信用为零。把独立 Git fixture 放在另一个 checkout 之外;在解读失败之前,验证它们解析后的项目 slug 与状态根。在那里证明一次 seed commit 成功:仓库局部的作者配置不建立新 fixture 仓库可用的身份。在调用 Bun 之前拒绝缺失的显式测试文件;它可以静默地忽略不存在的文件选择器并通过剩余的文件。保留退出状态经由日志。使用文档化的 detached runner 与 eval 锁。在编辑后复审最终启动器;准备与 `--list` 模式不应启动 monitors、retainers 或测试进程。用之前/之后的进程检查来验证。在长运行期间,检查最后的公开工具结果与待处理的权限状态;在等待截止之前先诊断一个被阻塞的 actor。保留取消与测试结果分开。跳过或未开始的用例不满足覆盖;保留每次尝试。付费 eval 从不重试。每个用例的 kind(`E2E_KINDS`)在运行前固定其试验次数:`rule` 一次试验;`behavior` 一个 3 次独立试验的面板,无契约违规时 ≥ 2 通过;`judge` 对一个输出采 3 个样本,每个维度以其中位数为门槛(至少 2/3 样本)对照不变的门槛。绝不在看到结果后增加试验、样本或 dispatch,绝不为改变结果而改变 kind 而不带通过率证据,并报告每次试验。隔离遵循 `CASE_QUARANTINE` 的进入与退出规则(`EVAL_POLICY`、`docs/TESTING_INTERNALS.md`)。一次普查每个红点都被机器分类为 INFRA 或 INCOMPLETE,可能作为新运行被重新 dispatch 一次;报告两次运行。
7. 用聚焦测试证明所有已知修复,包括受影响的付费用例。仅在一次具体修复或已证明的启动修正之后重跑失败的用例。在整合代码上跑剩余必需的选定评估。不要用完整免费套件来发现可预测的邻近失败。在消耗的输入与相关环境未变时,复用通过检查。对于模型裁判,比较扩展的提示、评分标准、参数与依赖;仅一个不同的 commit 不会让结果失效。不要重新采样一个未变更的通过裁判来简化启动器配置。保留其原始来源并把结果标记为复用证据。使用实际的 prompt builders 并在证明模型输入一致性时比较完整字节;在摘录中保留字面文本并记录已消耗的输入。
8. 完成评审修复、生成、发布元数据与构建,然后再进行最终验收。冻结代码,然后在最后跑一次 `bun run test`。在修复期间,聚焦检查在每次提交前替代完整套件运行。如果最终验收意外失败,保留失败,缩小诊断范围,并在下次完整运行前报告变更过的验证计划;永远不要盲目重试它,或从一个旧 revision 声称通过。
9. 仅在必需的检查通过时发布,除非用户明确授予对已识别失败的例外。报告 revision、实际的 pass/fail/skip 计数与不完整的覆盖。一个通过的子集不是发布验收。

## Build commands

```bash
bun install              # install dependencies
bun run typecheck        # strict tsc over product code; must report zero errors
bun run typecheck:test   # test-code type-debt ratchet (new diagnostics fail; --write-baseline locks in fixes)
bun run format:cso       # format lib/cso/*.ts (format:cso:check is the CI gate)
bun run test:quick       # fast measured free subset for edit feedback (not acceptance)
bun run test             # complete free suite via the strict shard runner (no API spend)
bun run eval:bg:pr       # changed live probes + selected judges; dispatches CI when HEAD is clean and pushed, else runs locally
bun run eval:bg:release  # fresh complete gate + periodic live coverage (same backend choice)
bun run test:health      # audit success metrics and weekly health from CI history (free; needs gh)
bun run build            # generate docs + compile binaries
bun run gen:skill-docs   # regenerate SKILL.md files from templates
bun run skill:check      # health dashboard for all skills
```

每个其他测试与评估命令(Ubicloud、Windows 子集、单个付费 tier 或用例、CI 中的分支验证、通过率、计划预览)及其成本与前置条件,集中在 [Which command do I run?](CONTRIBUTING.md#which-command-do-i-run)。代理轮询 `eval:bg:*` 日志以获取 `### gstack-detach EXIT=<code> ###` 哨兵
([CLAUDE.md](CLAUDE.md#running-evals-as-an-agent-always-detach-sigterm-proof))。

## Platform support

- **macOS** + **Linux**:支持完整测试套件。
- **Windows**: 策划的 Windows 安全子集在 `windows-free-tests` CI workflow 中跨六个 `windows-latest` 任务运行,按 Windows 测得的时长打包。Setup 脚本(`./setup`)目前需要 Git Bash 或 MSYS;原生 PowerShell 支持是未来的扩展。`bin/gstack-paths` 辅助函数通过 `CLAUDE_PLUGIN_DATA` / `GSTACK_HOME` 解析状态根,让插件安装在每个平台上工作。
- **CSO 原生 helper**: `/cso` 还需要 Bun 的四个 `--no-compile-autoload-*` 标志以及一个原生工具链(Linux 上的 static-capable C、macOS 上的 Xcode 命令行工具,或 Windows 上的 VS 2022 C++ Build Tools)。Setup 安装其他一切,当该可选工具链缺失时明确将 `/cso` 留作不可用。
- **Browser and renderer**: 浏览器 skill、`/make-pdf` 与 `/diagram` 优先驱动 Aside 浏览器(仅 macOS 15+)。在 Linux 与 Windows(或 Mac 但 Aside 关闭)上,准备检查会一句话说明,然后同样的 skill 改用 gstack 自家捆绑的浏览器,由 `./setup` 构建。

## Key conventions

- SKILL.md 文件**由** `.tmpl` 模板**生成**。编辑模板,而不是输出。
- 跑 `bun run gen-skill-docs --host codex` 重新生成 Codex 专属输出。
- 技能中的浏览器步骤是遵循 `scripts/resolvers/aside.ts`(Aside 优先)菜谱的 `aside repl` 脚本,每个都配一个回退引擎的 `$B` 等价物;`$B <command>` 是 browse 二进制,当 Aside 探查未打印 `READY` 时是合法工具。本地 HTML 通过 `bin/gstack-render.ts` 渲染,以同样的方式挑选。
- 安全技能(careful、freeze、guard)使用内联顾问式散文——在破坏性操作之前始终确认。
- 状态路径通过 `lib/state-root.ts` 及其来源 bash 双胞胎 `bin/gstack-state-root.sh` 拥有的单条链解析(`GSTACK_STATE_ROOT` → `GSTACK_HOME` → `GSTACK_STATE_DIR` → gstack 的 `CLAUDE_PLUGIN_DATA` → `~/.gstack`;见 docs/state-root.md)。技能散文使用 `GSTACK_STATE_ROOT=$(bin/gstack-paths --get GSTACK_STATE_ROOT)` 及其 `${GSTACK_STATE_ROOT:?…}` 卫语句(worktree 隔离的 Claude Code 会话拒绝 eval);`test/state-root-ratchet.test.ts` 拒绝手撸链。
- 浏览守护进程 HTTP 路由是 `browse/src/routes/table.ts` 中的条目(其表头展示了如何添加一个);永远不要在 `server.ts` 中按 `url.pathname` dispatch。
- 两条测试 lane 都通过 `scripts/lib/shard-engine.ts` 跑分片;免费与付费 runner 仅持有 lane 策略。PTY harness 代码位于 `test/helpers/pty/*` 之后,通过 `claude-pty-runner.ts` barrel。
- Outside-voice 失败散文(auth、timeout、empty、fallback)仅来自 `scripts/resolvers/outside-voice.ts` 中的 `outsideVoiceFailurePolicy()`。
- 每个 env-var-host 围栏都以 `scripts/resolvers/runtime-root.ts` 中的共享前奏(由其 post-render 通道插入)开头;永远不要在模板中手撸 gstack 的根。
- 每次 Codex 运行的结果都经过 `lib/outside-review-result.ts`;其原因码位于 `lib/gate-outcomes.ts`,每个都带 `docs/troubleshooting.md` 锚点。
- `test/module-size-ratchet.test.ts` 让重构过的所有者模块保持 800 行(每个函数 150 行)以内,让残余文件不增长。
- `claude` CLI 二进制通过 `lib/claude-bin.ts` 解析(从 `browse/src/claude-bin.ts` 重新导出供 browse 内部使用;`Bun.which()` + `GSTACK_CLAUDE_BIN` 覆盖)。在 Windows 上通过 WSL 跑 Claude 时,设 `GSTACK_CLAUDE_BIN=wsl` 加 `GSTACK_CLAUDE_BIN_ARGS='["claude"]'`。
