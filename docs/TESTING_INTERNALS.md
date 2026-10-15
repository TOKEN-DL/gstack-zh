# 测试内部:环境键、密封的 E2E

逐字从 CLAUDE.md 移出(减少 token 加载)。在编写或调试 E2E 测试、将 `env:` 传递给 runner 或触及 `test/helpers/hermetic-env.ts` 之前,请阅读此内容。

**Conductor 工作区中的环境键。** `GSTACK_*` 环境 shim(v1.39.2.0+,`lib/conductor-env-shim.ts`)将 `GSTACK_ANTHROPIC_API_KEY` / `GSTACK_OPENAI_API_KEY` 提升到 gstack 的 TS 二进制文件中的规范名称。通过 gstack 入口点运行的测试会自动继承此提升。不要将密钥值回显到 stdout、日志或 shell 历史记录。历史的"永远不要将 `env:` 传递给 `runAgentSdkTest`"规则已弃用:失败是部分环境替换(SDK 的 `Options.env` 替换子项的整个环境,因此没有密钥的对象破坏了身份验证)。Runner 现在始终传递带有每个测试 `env:` 最后合并的完整 hermetic 环境,因此每个测试覆盖是安全的;环境 `process.env.ANTHROPIC_API_KEY` 变更也仍然有效(环境构建器在调用时读取 process.env)。

**Hermetic 本地 E2E(默认)。** 每个 E2E runner(claude -p、PTY、Agent SDK、codex、gemini)都通过 `test/helpers/hermetic-env.ts` 生成子项:允许列表清理的环境(操作员 `CONDUCTOR_*`、`CLAUDE_*`、`GSTACK_*`、`MCP_*`、`GBRAIN_*` 和凭据(如 `GH_TOKEN`)永远不会到达子项)、新的播种的 `CLAUDE_CONFIG_DIR`(没有操作员 `~/.claude` CLAUDE.md / MCP 服务器 / skill)、临时 `GSTACK_HOME` 和 `--strict-mcp-config`。本地评估信号匹配 CI。使用 `EVALS_HERMETIC=0` 针对真实操作员状态进行调试(恢复旧环境并删除 strict-MCP 标志)。每个测试 `env:` 覆盖最后合并,因此故意的污染(`CONDUCTOR_WORKSPACE_PATH`、每个测试 `GSTACK_HOME`)继续有效。Hermetic 配置目录默认不播种任何 skill;键入 `/skill` slash 命令的 PTY 测试必须将 `seedSkills: true` 传递给 PTY runner,这会将子项的 `CLAUDE_CONFIG_DIR` 指向 `hermeticSkillsConfigDir()` — 一个播种的注册表,该注册表符号链接 LIVE 工作树的 SKILL.md 文件(设计:技能是测试对象;快照会测量陈旧的副本)。连接由 `test/hermetic-wiring.test.ts`(静态 tripwire)、`test/skill-e2e-hermetic-canary.test.ts` 中的两个 gate-tier canary 以及 `test/hermetic-skills-seeding.test.ts` / `test/pty-skill-seeding-wiring.test.ts` 中的播种 tripwire 固定。

播种的计划会话还会通过 `test/helpers/hermetic-skill-runtime.ts` 接收隔离的运行时主目录,因此绝对的 lazy-section 路径解析为正在测试的工作树。显式的每个测试主目录覆盖保持不变。Autoplan 从其自己的已安装主机注册表中解析每个评审 skill。

**交互式计划证据。** 原生计划评审计数驱动程序使用 `observeScreen: true` 并在选择输入之前等待 `currentScreen()`。现有的 xterm 依赖项解释光标移动和擦除;原始流中的旧菜单无法建立当前提示。快照分别保留 `terminal.raw.log`、`terminal.visible.log` 和 `terminal.screen.log`。设置 `EVALS_RUN_ID` 或 `GSTACK_EVAL_DIR` 保留这些快照;没有运行 ID 的输出目录为该写入器获得稳定的、唯一的本地 ID。已完成的原生脚本调用建立问题计数和阶段覆盖率。报告感知的计数测试还需要新鲜的、完整的报告以及接受完成标题的原生完成证据。

周期性的第一个问题矩阵使用 `test/helpers/auq-native-capture.ts` 将第一个公共 `PreToolUse` AskUserQuestion 有效负载与其当前原生显示匹配。它对该问题的确切公共字段进行评分,而不回答它或读取 model 脚本。`question_captured` 记录 `workflowCompleted: false`。使用 `GSTACK_EVAL_DIR`、`EVALS_RUN_ID` 或显式的运行 ID,`native-auq/<run-id>/<test>-<suffix>/capture.json` 在评估目录下保留公共有效负载、有界的当前视口和捕获结果。CEO 模式选择使用 `auq-sdk-capture.ts` 中的实际 SDK `AskUserQuestion` 权限回调,以及现有的 12 回合和 240 秒限制。它捕获公共问题并在不提交答案的情况下停止;其 `question_captured` 结果还记录 `workflowCompleted: false`。保留的捕获在 fixture 清理后仍然存在。Provider 拒绝和格式错误的问题仍然是失败。Section-loading 捕获保留其非交互式合约。

共享代码重新验证 fixture 将公共工具调用与其成功结果配对,以验证在完成之前检查了当前的可信开始记录。在工具输出中发现的路径计数;仅在指令或叙述中提到的路径不计数。保存的公共捕获涵盖绝对和相对路径以及发现后的读取。重新验证提示提供可信开始记录目录的路径并声明现有的回合限制。它要求智能体批量独立读取并检索完整的最终记录;每个源、批准、持久性和完成检查仍然适用。路径边界 fixture 对符号链接、子模块、忽略的文件、索引标志和遗留或过滤的证据使用相同的执行合约。他们的跳过 actor 接受明确的无更改选择;选项中的保留字,该字也批准更改,不能授权编辑。捕获的原生问题行使实际答案回调,并且原生回合限制失败即使在回答问题后仍然失败。

工程和 DX 发现 fixture 检查其播种问题的覆盖范围,而不是限制评审问题的总数。每个决策都需要一个不同的、已完成的原生问题以及提供的答案;接受、拒绝或延迟推荐都算作对其进行评审。工程的强制性遗留回归测试还需要肯定的计划或公共叙述证据。在现有时间限制内允许其他有用的问题。通用问题计数保持诊断性,并且需要新鲜的最终评审报告。

E2E 测试以实时方式流式传输进度(通过 `--output-format stream-json --verbose` 逐个工具)。结果持久化到 `~/.gstack/projects/<slug>/evals/`(遗留回退 `~/.gstack-dev/evals/`),并自动比较上一次完成的运行(飞行中的 `_partial` 文件永远不用作基线,因此运行无法针对自身进行比较)。

周期性的覆盖 fixture 使用版本化的行为 gate,其功效单独报告。有关确切的正确性要求、退役的 fanout 用例、不可变的证据以及通过结果的限制,请参见 [Overlay benchmark contract v4](OVERLAY_BENCHMARK_CONTRACT.md)。在 v4 下,已完成的错误答案是任一臂中的测量正确性结果(`taskCorrect: false` 与 `answerError`);仅覆盖-ON 错误答案会使用例失败,因此覆盖-OFF 错误答案保留在比较中。v3 verdict 保留其原始含义。

## 覆盖所有权

[测试组合审计](TEST_PORTFOLIO.md)分离确定性 harness 检查、提示 judge、实时第一个问题捕获、完成的工作流和平台集成。共享设置是可重用的;具有不同场景或独立试验要求的证据不是。它记录了保留的覆盖率和测试速度重构的测量组件节省。

## Runners:套件如何执行(2026-08 检修)

**仅 Aside E2E 测试在没有实时 Aside 时自跳;浏览器驱动测试在任何引擎上运行。** 每个打开网页的 skill 首先驱动 Aside AI 浏览器(`scripts/resolvers/aside.ts`),并在 Aside 不存在时回退到 gstack 自己的 browse 引擎(以及引导的 Chromium)。需要 Aside 本身的用例(`test/skill-e2e-aside.test.ts`,`design-review-fix` 在 `test/skill-e2e-design.test.ts` 中)调用 `asideAvailable()` 从 `test/helpers/aside-available.ts`(与 skill 在 BROWSER SETUP 中运行的相同探针),并在 `aside` CLI 或 Aside 应用程序不存在时跳过。CI runner 没有 Aside,因此这些仅在 macOS 开发机器上运行并位于周期 tier;设置 `GSTACK_SKIP_ASIDE=1` 以在本地强制跳过(它还行使回退交接)。qa E2E 文件(`test/skill-e2e-qa-workflow.test.ts`,`test/skill-e2e-qa-bugs.test.ts`)在 `asideAvailable() || fs.existsSync(browseBin)` 上设门:skill 自己的 BROWSER SETUP 选择引擎,因此在 Mac 上它们驱动 Aside,在 Linux CI 上它们驱动内置的 browse 二进制文件,仅在两者都不存在时跳过。`$B` 驱动的 E2E 用例和 `browse/test/` 在每个平台上像以前一样运行,因此 Linux CI 证明回退引擎有效。

**引导依赖项保留是可选的资格,而不是行为测试。** `qa-bootstrap` 仍然在 macOS 和 Linux 上运行其原始未固定的 Vitest 安装和断言,包括记录的未分片命令。只有 Linux 付费分片 runner 发出拥有的保留范围:它绑定每个实际的 fixture 和原生生命周期,保留锁、包清单和已安装的文件/链接清单,并在删除 fixture 之前确认捕获。其他平台明确报告保留不可用,并且仍然执行原生行为测试。没有 runner 发出的范围的运行不会获得保留的依赖项资格信用;需要该证据的候选验收必须使用 Linux 分片路径并验证每次尝试的完整捕获、确认和清理回退。传递的未分片或 macOS 行为测试不能替代该证据。

**渲染器以相同的方式选择,因此渲染 gate 与引擎无关。** `/make-pdf`、`/diagram` 和设计预览通过 `lib/aside-render.ts` / `bin/gstack-render.ts` 打印和截取其本地 HTML,当 `probeAside()` 说 `READY` 时在 Aside 中渲染,否则通过 browse 引擎渲染。make-pdf 的 `*-gate.test.ts` 和 `test/skill-e2e-diagram.test.ts`(周期性的、付费的)在 `browserAvailable()` 上设门(`test/helpers/browser-available.ts`:`asideAvailable() || resolveBrowseBin() !== null`)— 在 Mac 上它们通过 Aside 打印,在 Linux CI 上通过 `bun run build:gates` 编译的 browse 二进制文件,仅在两者都不存在时跳过。只有 `test/aside-render.test.ts` 的两个实时 Aside 用例(完整往返和后期就绪 `--wait-expr` 轮询)是仅限 Aside:其选项映射和生成的脚本 pin 在任何地方运行,其假可执行用例对两个引擎进行密封测试(PATH 上的假 `aside` / `browse` 脚本固定探针分类、stdout 合约、环回服务器策略、超时 kill、引擎选择和中途回退)。`test/gstack-render-cli.test.ts` 对 `bin/gstack-render.ts` 执行相同的操作,使用 `GSTACK_SKIP_ASIDE=1` 和 `GSTACK_BROWSE_BIN` 指向记录每个 argv 行的假守护进程。Linux 上的绿色 gate 证明回退引擎,而不是 Aside;Mac 运行是 Aside 证据。browse-binary leg 假定 Chromium 已引导:`resolveBrowseBin()` 仅检查二进制文件(或 `find-browse` shim)是否存在,从不检查 Chromium 是否可以启动,因此在跳过尽力而为的 Chromium 步骤(`GSTACK_SKIP_PLAYWRIGHT=1`)或失败的安装中,这些 gate 运行并在浏览器启动时失败而不是跳过。在本地运行它们之前修复引导(或将二进制文件移开);CI 始终首先安装 Chromium。`test/dom-dump-hygiene.test.ts` 是同一 leg 上的一个免费套件用例:它在真实 Chromium 页面中通过内置的 browse 二进制文件运行 `lib/dom-dump.js`(`/design-review` 交给设计检测器的渲染 DOM 转储),因此它在二进制文件不存在时自跳,并且由于冷 Chromium 启动在繁忙的开发 box 上对负载敏感,因此仅在 CI 或显式选择加入时运行(`GSTACK_DOM_DUMP_HYGIENE=1`)。

**免费套件(`bun run test:free`)。** `scripts/test-free-shards.ts` 运行 N 个并发分片进程(每个内部串行),每个分片严格输出分类。本地默认使用可用的 CPU 亲和性(Bun 还通过容器的 cgroup CPU 配额限制),下限为 1,在 Linux 上限为 16,在 macOS 和 Windows 上限为 6;`GSTACK_FREE_JOBS` 仍然是显式覆盖。这不会更改单独的 CI 机器计数。完整套件分片按 RECORDED 每文件时长打包(LPT,`packShardsByDuration`),当提交的种子 `scripts/free-test-durations.json` 存在时 — 使用 `bun run test:ubicloud --record-durations` 在 VM 上使用 CI lane 的环境为每个文件计时,并将种子复制回来;参见 [免费套件时长种子](#free-suite-duration-seed)。缺失的种子 → 静默哈希分片回退;损坏的种子 → 一个警告 + 回退;未知文件获得第 99 百分位悲观,并且完整套件和 `--ci-plan` 运行在 stderr 上命名它们,因此新的慢速文件不能默默地成为最长的一端。打包的分片获得时长感知墙(`max(base, predicted × 3, files × 5s)`)。旧版 `--shards N --shard i` 路径保持稳定的哈希索引。必需的 CI 使用一个时长打包的 `--ci-plan`、20 个普通的 `--ci-run` 分片加上一个单独的独占 fixture 分片,以及一个 `--ci-verify` 聚合。CI 分片仍然在独立的机器上运行,没有排序不相关的作业。公共 `TREE_MUTATING` 映射现在对独占的主机状态 fixture 进行分类;其唯一条目是 `test/bootstrap-retention.test.ts`,其同 UID 不可转储参与者影响主机范围的 procfs 权限检查。本地,此文件仅在所有并行分片结算后运行,并且取消阻止该最后阶段启动。没有选择文件、重试、预算或收据要求被删除。以前的生成器 mutator 仍然渲染到私有输出目录;此串行阶段保护进程可见性,而不是就地文档生成。

完整的子项输出保留在 `.context/free-test-logs/` 下的私有文件中,在每个分片的临时清理目录之外。Runner 在启动和完成时打印路径。丢失日志会使运行失败,即使子项成功退出。重定向的日志目录在启动子项之前被拒绝。在失败时,请先阅读该日志:恢复消息区分不完整的捕获、未确认的清理、截止日期到期和测试/模块失败。在重新运行之前修复已证明的原因。仅当每个失败都可归因于现有选定文件时,才提供有针对性的 `bun test` 命令;它证明修复,而不是原始选择的完成。共享结果时保留失败的尝试,并在共享之前检查日志中的私有数据。

在发布之前,对新的确定性回归进行分类以获得快速反馈。使用现有的记录器在固定输入上刷新计时种子;在测试运行时不要编辑源代码。当它们的反馈成本合理时,关键边界控件属于 `QUICK_CORE`。其他测量的文件在两秒或更短的时间内合格;缓慢和未测量的文件保留在快速之外,而不是完整测试之外。单独报告冷设置和热执行,同时在总成本中保留失败尝试、重试和清理时间。

**PTY fixture 计时。** Plan-count 会话在终端输出或退出时唤醒,昂贵观察之间至少间隔 250ms,以及用于不产生终端输出的脚本或 hook 更改的 2s 回退。新的输出批次在观察之前稳定 250ms,以便分割的终端重绘不能从其第一个块路由输入。输入去抖、权限保护以及真实 CLI 的 8 秒启动宽限时间保持不变。合成 CLI 可以将 `startupReadyMarker` 传递给 `runPlanSkillCounting` 并在安装其输入处理程序后发出该确切标记;缺失标记在任何命令发送之前失败。标记等待保持在现有的启动和总运行截止日期内。`test/pty-output-wake.test.ts` 涵盖输出、静默等待、退出、关闭、缺失就绪、分割重绘和连续重绘。关闭和输出等待取消其失败的截止日期,以便已完成的工作者可以立即退出。

UI-positive 设计 gate 在第一个 model 回合之前将提供的计划提交为 `review-input.md`。它要求确认的 Design 焦点/评分问题,然后是关于具体 UI 行为的已回答的原生发现;设置和外部评审选择无法满足该序列。其声明的董事会 actor 在确认之前提交反馈。最终证明使用完整的原生问题,而不是截断的诊断片段,并且提到"无 UI 范围"的提案文本不被视为退出 verdict。未知命令失败必须命名调用的 slash 命令;拒绝 `--help` 的子工具不是 skill 注册失败。周期性的播种发现分类器单独验证 fixture 拥有的发现。

**付费套件(分片 runner,本地和 CI)。** `scripts/test-paid-shards.ts` 是单一选择引擎:每个分片 1 个文件,`EVALS_JOBS` 分片进程 × `EVALS_CONCURRENCY` 在分片内,每个分片 `GSTACK_EVAL_DIR`,完整流假脱机到每个分片日志文件(在 START 和失败时打印路径),从未启动/超时分类,以及父级计算的差异选择通过 `EVALS_SELECTION_JSON` 传播到子项(失败打开:无法解析它的子项在本地重新计算,带一个警告)。付费评估从不重试;每个用例的种类在运行之前固定其试验(参见下面的"评估 verdict policy")。`CASE_SHARDED_FILES` 中的文件为每个进程运行一个已注册的用例(`<file>#<case id>`,确切的 `--test-name-pattern`,恰好执行一个用例),因此长文件的短用例跨 runner 分布,并且每个用例都有自己的 SDK 信号量。`test/carve-section-loading.test.ts` 是其中之一:它为每个雕刻的技能注册一个周期性用例(`carve-section-loading-<skill>`,每个都有自己的触摸文件和历史记录)。使用 `bun run scripts/test-paid-shards.ts --tier periodic --case carve-section-loading-<skill>` 运行一个用例。试用遥测骑乘存储:每个记录的测试携带其 1-based `attempt` 加上,在隔离的试用分片上,其 `case_id`、`kind`、`trial`、`panel` 和 `policy_version`,并且每个 lane 的报告为每个试用上传一个 `trial-outcomes` JSONL 行。`bun run eval:pass-rates` 将该历史记录转换为按用例的通过率(参见下面的"通过率历史";免费套件 flake 是单独的记录,参见 [Flake ledger](#flake-ledger))。普查完整性从免费套件强制执行:每个 `E2E_TOUCHFILES` / `LLM_JUDGE_TOUCHFILES` 键必须命名一个活动的付费测试(`test/touchfiles.test.ts` 的反向不变量),并且禁止 `git show <sha>:path` fixture — 而是供应商字节(`test/git-ref-fixture-tripwire.test.ts`)。

功能 QA 和文档验收需要明确的 `EVALS_RUN_ID`;`GSTACK_EVAL_DIR` 单独不满足其证据所有权保护。对于每个本地调用,为记录的分离 runner 提供一个新鲜的 ID:

```bash
EVALS_RUN_ID="local-$(bun -e 'console.log(crypto.randomUUID())')" bun run eval:bg:pr
```

包脚本和 `gstack-detach` 都不会发明此身份。CI 的 PR/手动切片、周期切片和每周 gate 普查提供绑定到工作流运行、尝试、作业和切片的 ID。现有切片工件保留每个分片快照;单独的始终运行 `native-captures-<EVALS_RUN_ID>` 工件将项目/遗留 `e2e-runs` 和 `evals/qa-callers` 证据保留 90 天。这些诊断工件不是收集器结果,也不建立未完成的测试已通过。

**快速 PR profile 和证据重用。** `test:pr` 在 `scripts/test-pr-profile.ts` 中选择更改的用例以及每个更改的质量 judge。`--profile full` 保留广泛的人口普查;没有用例 ID 或 tier 分配被删除。该计划记录 E2E 和 judge 选择并列出延迟的覆盖。执行器分别接收这些选择,并且报告检查实际执行的用例计数。未知的源依赖项恢复广泛的 gate(参见 [PR 付费 lane 回退](#pr-paid-lane-fallback));没有注册覆盖的未映射提示需要完整验证。已知仅广泛的提示更改被明确延迟,不计为 PR 通过。`evals-periodic.yml` 的每周/手动运行完整地执行两个完整的人口普查;手动 `evals.yml` 运行广泛的 gate。这故意将一些缺陷检测移到以后。

`scripts/eval-input-cache.ts` 仅接受完整、干净的、第一次尝试的通过,并匹配之前/之后的输入。审计的工作流-judge 适配器对实际的扩展提示、源/fixture/rubric/runner 闭包、已安装的 SDK、model 参数和运行时进行哈希。缺失/未知输入强制执行。收据的范围限定于同一仓库和 PR,在 24 小时后过期,并包含公共分数和出处,而不是提示或秘密。使用 `runWorkflowJudge` 的 17 个用例中,16 个符合条件;cookie 工作流的自定义输入与缓存适配器不匹配并保持新鲜,其他 11 个质量用例也是如此。CI 提供范围限定的缓存/运行时配置;本地运行默认是新鲜的。缓存的分数必须通过当前断言;重用的记录保留其原始来源和时间,无法续订收据。`scripts/e2e-shard-reuse.ts` 将相同的收据扩展到 PR-profile E2E 分片(付费评估从不重试,因此通过在结构上是第一次尝试):标识哈希测试的导入闭包、每个跟踪的文件

[desktop write output truncated: original_bytes=57986; if the omitted remainder is needed, continue with read using offset=320 and the same path.]
