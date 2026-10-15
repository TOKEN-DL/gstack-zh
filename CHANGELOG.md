# Changelog

> **关于本翻译的说明:** 本文件源自 gstack 项目的 CHANGELOG.md(约 12,743 行,1.4MB),按版本时间倒序记录从 v0.3.0 到 v1.91.100 的所有发布。翻译保留以下原样:版本号、日期、PR 编号(`#1694` 等)、commit SHA、issue 引用、用户名(以 `Contributed by @username` 格式)、文件路径、命令行、env var 名、JSON/YAML/TS 配置示例、API endpoint、价格数字。翻译专注于散文叙述与技术说明的中文化。

---

## [1.91.100.0] - 2026-10-13

**一次 gbrain LongMemEval 调用计划现在能完整计价。** gbrain 的 `eval longmemeval --call-plan`(gbrain#6638)为运行的每个 lane 写一个 `GSTACK_EVAL_CALLS` 文件。`gstack-eval-plan price-gate` 验证了它的绑定,但停在了 `EVAL_PRICE_MISSING`,总价为未知:它没有给裁判(`gpt-4o`)或 reranker(`rerank-2.5`)留行,也不识别 gbrain 解析的带日期的 Claude id(`claude-haiku-4-5-20251001`)。在 gbrain fixture 计划(`--mode tokenmax --judge`)上,所有 7 个 lane 现在都能计价,总价 $0.447。

### What this means for you

- **带日期的 Claude 快照按其别名计价。** `claude-<name>-YYYYMMDD`,带或不带 provider 前缀,解析为 `claude-<name>` 行。带日期的 OpenAI 快照未合并,因为某些(`gpt-4o-2024-05-13`)有它们自己的价格。
- **新行,来源与日期在 `lib/pricing.ts`:** `gpt-4o`($2.50 输入,$10 输出,每百万 token 缓存 $1.25)与 Voyage rerankers `rerank-2.5`($0.05)与 `rerank-2.5-lite`(每百万处理 token $0.02:query tokens × documents,加上 document tokens)。

### For contributors

- `test/pricing-eval-plan.test.ts` 检查 gbrain 调用计划命名的每个模型。它在该变更之前失败。

## [1.91.99.0] - 2026-10-13

**Capy 上的 Bun pin 不匹配现在命名修复它的命令,并要求 skill 跑它。** 在 gbrain 在 CI 中钉住 Bun 1.4.3 后,一个 gbrain checkout 的会话启动检查以 `gstack: fail pins` 结束,其 fix 行提供两个选择且没有命令,而 skill 的修复步骤仅覆盖 `revision` 和 `runtime`,所以遵循该 skill 的代理让机器失败。同一个没有 `--check` 的 bootstrapper 已经安装钉住的 Bun(已验证的 1.4.3,约 4 秒)并以 `gstack: ok` 结束。

### What this means for you

- **`pins` 行的修复是 Capy 上的确切命令。** 在 Drive bootstrapper 下,`FAIL pins … (PROJECT_PIN_MISMATCH)` 现在说 `fix: bash <skill>/scripts/install.sh [--for <skill>] (installs the project's pinned Bun when it is a verified version: 1.4.2 1.4.3; otherwise it names the conflict), or change the pin in <project>`。在 Capy 之外,fix 文本不变。
- **Capy skill 的修复步骤覆盖 `pins`。** 当 trailer 命名 `revision`、`runtime` 或 `pins` 时,步骤 2 在没有 `--check` 的情况下跑同一命令。gstack 没有已验证归档的 pin 以 `RUNTIME_PIN_CONFLICT` 停止,列出要提升的 pin。所有者按之前一样把 `capy/skill/gstack/` 复制到 Drive skill 文件夹;`scripts/install.sh` 不变。

### For contributors

- `test/gstack-doctor.test.ts` 检查 Capy fix 行(带或不带 `--for`)以及非 Capy 文本不命名 wrapper;它在该变更之前失败。

## [1.91.98.0] - 2026-10-13

**lane 的计划义务现在在报告完成前满足仓库自己的 gates。** 付费 gates 在批准前获得价格,重新盖戳由机器证明仅为盖戳,且封装的 eval 输出不能搭便车进入公共仓库的 push。

### What this means for you

- **Repo guards 在 `gstack-pregate` 中跑。** `.gstack/pregate.json` 接受 `guards`: `{id, when: [globs], command, fix}`。当变更的文件匹配 `when` 时,guard 在其自己的进程中以超时运行,失败命名 guard 与其修复。Guards 覆盖导入图看不到的检查,例如 serial-test manifest 行或 spawn 二进制的 CLI goldens。必需的 guards 不能被降级,没有 `--allow-repo-commands` 运行是 `incomplete`,永远不是通过。lanes 检查在已验证的 Bun 上从安装的启动目录跑,且逐个进程跑 serial test 文件。未知 flag 以 exit 2 退出而不是被忽略。
- **`gstack-eval-plan price-gate` 在不跑的情况下为付费 gate 定价。** 它读一个版本化的 call-plan 文件(`--plan-file`,或在基分支策略中的 `price_file`),该文件绑定到 gate 命令、head 和输入,并打印 `GATE_PRICE:` 连同该文件声明的每个前置条件。批准前不跑任何东西。未声明计划的 gate 是 `EVAL_GATE_UNPRICED`;`--allow-unpriced "<reason>"` 批准它作为记录的所有者覆盖,而不是花钱的许可。过期或格式错误的计划文件被拒绝。新的 `reservation-fits-cap` 检查捕获其 reservation 不能适应 smoke cap 的单次调用。`/eval-plan` 以这种方式在批准时给每个付费 gate 定价,将 gate 命令作为文件(`--gate-file`)传递,这样带引号或 `$` 的命令永远不经过 shell 字符串。Wire 格式与参考发射器在 `docs/eval-calls.md` 中。
- **当前价格。** Claude Sonnet 5.5、Gemini 3.x Flash 与 gbrain embedding 模型按 provider 页面定价(表中来源与日期),Claude Opus 4.7 按其当前列表价($5/$25 每百万 token)。
- **`gstack-pregate regate --against <gated-head>` 说重新盖戳的 head 是否需要新 gate。** 它计算一棵新鲜的树收据并打印 `STAMP_ONLY:`、每个 gate lane 的一行 `LANE:`(可重用、重跑或不可重用)、针对基与每个其他开放 PR 的 `MIGRATION_COLLISION:`,以及 `REGATE: needed|not-needed`。`not-needed` 要求 gated head 的 gate 证据、未变的执行标识、无盖戳敏感的 rerun 以及无未知 collision 查找。裸 `--against` 是一个别名。
- **仓库的盖戳规则就是盖戳规则。** Ship policy 接受 `stamp_files`(允许不同的文件)与字面 `stamp_patterns` 带一个 `{version}` slot,例如 `since: '{version}'`。对版本放在引号 `since:` 行、package manifests 或盖戳注释中的仓库,纯盖戳现在分类为盖戳。`restamp_command` 在 restamp 的暂存 worktree 中以目标版本跑仓库自己的 restamp,在一棵干净树上;失败或错误版本的输出不写任何东西。`migrations` 键(目录与 id 模式)馈入 collision 检查。
- **封装的输出 guard 在 push 前跑。** 在 `.gstack/pregate.json` 有 `sealed` 键的仓库中,托管的 pre-push hook 与 `gstack-pregate sealed` 检查 push 发布的每个 commit:禁止路径(包括二进制、空、重命名以及先 add-然后-delete 的文件)、转录、引用答案标记、seed 与机器本地路径。在公共仓库中,匹配以 `SEALED_OUTPUT_REFUSED` 拒绝,带文件、行与规则;在私有仓库中警告。有效策略是基分支与推送分支的并集,所以 push 可以收紧策略但永远不能放宽。格式错误的策略被拒绝,未知可见性计为公共。豁免仅来自基分支 allow 文件。没有 `sealed` 键的仓库看不到变化。
- **Lane 交接携带安装与 pregate 行。** 协调者契约现在要求 lane 在报告完成前跑 `gstack-pregate run --allow-repo-commands`(以及在配置的地方跑 `gstack-pregate sealed`)。其交接列出 `GSTACK_RESULT`,然后 `gstack: ok|fail <rev>`,然后 `PREGATE:`,然后 gate 摘要。

### For contributors

- `scripts/pr-dependencies.ts` 广度优先追踪,所以 trace 命名从用例的 touchfiles 到文件的最短链(选择不变)。
- 新增:`lib/pregate/checks/guards.ts`、`lib/pregate/regate.ts`、`lib/pregate/runtime.ts`、`lib/sealed-guard.ts`;文档 `docs/eval-calls.md`、`docs/pregate.md#regate` 与 `#gbrain-worked-configs`。
- 新的带故障排除锚点的结果码包括 `EVAL_GATE_UNPRICED`、`PREGATE_GUARD_FAILED`、`SEALED_OUTPUT_REFUSED` 与 `MIGRATION_COLLISION`。
- Pregate 与 ship-policy 验证器仍拒绝未知 key,所以仓库应在 machines 跑该 release 之后才提交 `guards`、`sealed`、`price_file`、`stamp_files`、`stamp_patterns`、`restamp_command` 或 `migrations`。

## [1.91.97.0] - 2026-10-13

**Capy 上一个无人值守的 `/autoplan` 运行现在在它自己的 run 目录中结束。** Capy 子代理所做的评审用一条命令绑定,修订后的 plan 在同一 run 中被评审,gate 问更少、更好的问题。

### What this means for you

- **`submit --record <file>` 绑定子代理手写的评审。** 它接受规范的 findings fence、一张 findings 表(Severity 与 Title 列)、一个带严重性标签的列表(`- [High] title`)或一条 `No findings.` 行,且从不猜测严重性。`INPUT:` 行存在但错误的记录被拒绝。没有 `INPUT:` 行的记录仅在 `--attest "<reason>" --attempt <id>` 时绑定;它被标记为 `receipt: attested`,`validate` 计算它(`attested=<n>`),且如果该 attempt 在被 dispatch 后 plan 被修订,则被拒绝。`--attempt` 在恰好一个 attempt 开启时可选。`submit --voice outside --unavailable|--failed "<reason>"` 关闭一个 outside voice 无法跑的 phase,且 `next` 与 `resume` 永不重新 dispatch 它。
- **`revise --plan <v2>` 在同一 run 中评审修订后的 plan。** 每个被评审的 Implementation-plan hash 不同的 phase 重新打开(加 Eng)在新的 snapshot 上;在 gate 上它开始下一次 gate 修订。Revise 是一个可恢复的步骤(`resume` 完成一个中断的),且 archives 永不互相覆盖。`next` 在 plan 文件自其 snapshot 以来移动时打印 `PLAN_CHANGED` 且 `answer` 打印 `PLAN_SOURCE_CHANGED`。
- **接受的 plan 变更必须到达 plan。** 当所有者在 gate 接受变更,`answer` 打印每个变更一行 `APPLY:` 与 `NEXT: gstack-autoplan revise …`。`next` 以 `PLAN_AMEND_PENDING` 拒绝,直到 `revise --incorporates <decision ids>` 绑定编辑,或所有者用 `--withdraw <id>` 撤回变更。一个 run 不能在有挂起变更时导出为 complete。命名不是 pending 的 id(从未作为 plan 变更被接受,或已经结算)以 `AMENDMENT_UNKNOWN` 拒绝,而不是询问你刚做的编辑。
- **用户挑战默认遵循一致的声音,并说明原因。** 当两个声音都将同一 finding 标记为对范围、成本或方向的变更(同一 `changes` 值)且都给出补救时,该 item 现在推荐变更。选项 b 逐字显示两个补救,默认标记为 `recommended: b (both voices; remedies shown)`。任何更弱的内容保持你的方向,标记为 `(plan direction; voices disagree)`。**`all` 现在接受那些一致的变更**;回复 `all as-is` 在一个 token 中保持每个方向。p1 的推荐文本命名它将应用的变更。
- **重跑停止 churn。** 评审者仅对范围、成本或方向的变更(`changes`)设置 `user_challenge`。在重跑上,没有变更的 challenge 成为代理自己的决定,且不重新打开任何东西。重跑提示带一个 `DECIDED AT THE GATE` 块,且对来自未重跑的 phase 的 item 的答案转发,而不是被再次询问。部分回复永不覆盖更早的答案,且 gate 保持开放直到每个批准被结算。
- **没有两个 section 的 plan 是被脚手架的,而不是以空输出被拒绝的。** 第一次 `next` 解释 `PLAN_SECTIONS_INVALID`;`next --scaffold` 评审带缺失标题添加的副本,且永不编辑你的文件。
- **在旧 gstack 上门控的 runs 仍能验证。** 来自旧 gstack 的 run 目录打印一次 `GSTACK_VERSION_CHANGED` 并向前移动,在 1.91.95 之前铸造的 decision ids 被重命名并 journaled,所以升级后导出的重开 run 能验证。由较新 gstack 编写的 run 被拒绝。
- **Capy 上的 admission。** `next`、`submit`、`answer`、`revise`、`export` 与 `run` 做出与 skill 启动相同的安装检查:在 `GSTACK_CAPY=1` 下,它们在缺少通过安装判定(`INSTALL_UNVERIFIED`)时拒绝,且在无人值守会话中拒绝低于支持最小值的 PATH Bun。`status` 与 `resume` 始终工作。
- **Scope keys 关注变更的内容。** Design phase 在 Implementation plan 中需要渲染过的表面(`.tsx`、`.css` 或类似文件,或一个 UI 词挨着一个视觉词);评审记录中的词不再计算。`next` 按轴打印证据与来源(`source=ui:flag,dx:detected`)。`--no-dx` 是新的。plan 第一行上的 `<!-- gstack-autoplan-scope: … -->` 标记跨 run 保留 flags,`next` 打印要添加的标记;gstack 永不写你的 plan。交互式 skill 使用相同的检测器。
- **PLAN_REALITY 想要每个测量数字的收据与规则本身。** 每个声明测量数量(秒、%、$、tokens、MB)的子句需要其自己的 file:line、脚本或命令收据(`unreceipted_numbers=<n>` 加每条一行 `NUMBER_UNRECEIPTED:`),且声称决策规则的行必须引用或链接它(`unquoted=<n`)。诸如 `.gstack/ship-policy.json:1` 的 dot path 上的收据算数。`submit` 记录每个 plan 修订的检查;它永不拒绝 submit。
- **`autoplan/DRIVER.md` 是紧凑的 driver。** 约一百行从代码生成:phase 顺序、产物契约、gate 协议与协调者契约,无 hook 散文。`gstack-autoplan --help` 与第一次 `next` 打印其路径,且任何 host 都能读取它而不是完整的 autoplan skill。
- **六次同时 submit 一次绑定。** `run.lock` 现在用其所有者已写入的状态创建(一个暂存文件链接到位),且小于 30 秒的不可读锁计为已持有,所以在 create 与 write 之间读另一个锁的命令不再删除它并与之并行运行。

### For contributors

- 新模块:`lib/autoplan-revise.ts`、`lib/autoplan-migrate.ts`、`lib/autoplan-plan-text.ts`、`lib/autoplan-admission.ts`、`lib/autoplan-scope.ts`、`lib/source-root.ts`、`scripts/resolvers/autoplan-driver.ts`。
- `test/autoplan-capy-e2e.test.ts` 在模拟 Capy host 上驱动整个无人值守循环:规范的与见证的记录、不可用的 outside voice、部分回复、修订 interlock、revise、中断的 run 恢复、三个 gate 修订与一个验证的 export。

## [1.91.96.0] - 2026-10-13

**在 Capy 机器上,gstack 现在对自身说真话,并自行修复常见 case。** 旧 checkout 在会话开始时升级自身,安装检查与你的 workflow 跑的 Bun 一致,每条 fix 行都命名一个存在的命令。

### What this means for you

- **`bin/gstack-upgrade` 存在。** 它 fast-forwards checkout 到远程默认分支并重新安装(在 Capy 上它跑安装器,所以 Bun preflight 先到;在别处是 `./setup --refresh-registered`,它跑迁移)。它永不 stash、reset 或丢弃任何东西。脏的、分叉的、钉住的或锁定的 checkout 以 `UPGRADE_REFUSED` 拒绝,不可达的远程以 `UPGRADE_FETCH_FAILED` 拒绝。`--check` 打印 `current` 或 `behind <installed> -> <remote>` 且不改变任何东西。doctor 的 revision 修复、安装器的 `--revision` 注释与无人值守升级通知现在命名该命令。
- **低于支持最小值的 checkout 在会话开始时升级自身。** Capy skill 的 bootstrapper(`capy/skill/gstack/scripts/install.sh`)fetch 远程,从其读 `capy/support.env`,当 checkout 干净、未钉住且低于 `GSTACK_MIN_REVISION` 时,在安装锁下 fast-forward 它,重新安装并重新检查。`--check` 仅报告 `gstack: fail revision … (fast-forward available)`;安装模式做工作。fast-forward 完成后被打断的 setup 由下次 run 完成。`GSTACK_CAPY_AUTO_UPGRADE=0` 关闭它。
- **Workflow 在安装检查失败时拒绝。** 安装器为该 checkout 与 HEAD 写一个 verdict 文件。在 `GSTACK_CAPY=1` 下,`gstack-skill-start` 在 verdict 缺失、失败、为另一个 HEAD 或为不同 workflow 时拒绝一个 workflow,且在无人值守会话中它拒绝低于最小值的 PATH Bun(`status=refused`,exit 3)。`GSTACK_ALLOW_UNSUPPORTED_RUNTIME=1` 覆盖并被记录。交互式会话与其他 host 的派生会话仅警告。
- **安装检查与你的 workflow 跑的 Bun 一致。** `GSTACK_LAUNCH` 现在带 `PATH=<verified bun dir>:$PATH`、`GSTACK_CAPY=1` 与 `GSTACK_INSTALL_VERDICT`,且 wrapper 在 `--check` 与 install 上都打印它。在新 shell 中的裸 doctor run 命名 off-PATH Bun(`bun 1.4.2 found at ~/.bun/bin (not on PATH)`)带确切修复。安装器复用已验证的 Bun 而不是再次下载它,并在项目 CI 钉住的不同系统 Bun 旁边安装项目所钉的 Bun 版本。Planning-only 检查(`--for autoplan`、plan reviews、office-hours)将项目 test-pin 不匹配报告为 `SKIP pins (planning-only …)`;跑项目测试的 workflow 仍需要它。
- **Capy skill 的过期副本得到所有者动作而不是静默通过。** 规范 skill 现在位于仓库的 `capy/skill/gstack/`。Drive 副本是一个小型的冻结 bootstrapper,交接给 checkout 的完整 wrapper(`bin/gstack-capy-session`)。旧副本继续工作并打印 `CAPY_WRAPPER_STALE` 带 copy 命令。
- **从不需要的 machines 不为 install 付出代价。** 同一 HEAD 上的暖安装跳过 setup(约 1 秒)。Skill 在第一次 workflow 之前安装,而不是在每个会话开始时。`docs/capy-install.md` 解释如何将 gstack 烘焙到 Capy 快照。
- **安装的 Bun 跟随项目的确切 pin。** 当目标仓库确切钉住一个 Bun 版本(CI `bun-version:`、`.bun-version`、engines)高于 gstack 的最小值且 gstack 携带其已验证归档(现在 1.4.2 与 1.4.3),该归档被安装,所以将 CI 移到 1.4.3 的仓库不会让 Capy machines 在 1.4.2 上失败 `pins`。无已验证归档的确切 pin 以 `RUNTIME_PIN_CONFLICT` 停止,列出已验证版本。

### For contributors

- 新增:`bin/gstack-upgrade`、`bin/gstack-capy-session`、`bin/gstack-install-verdict.sh`、`lib/install-verdict.ts`、`capy/support.env`、`capy/skill/gstack/`。测试 `test/gstack-upgrade-bin.test.ts`、`test/capy-skill-wrapper.test.ts`、`test/install-verdict.test.ts`,以及共享的模拟 Capy host `test/helpers/capy-host.ts`(Bun 1.3.14 首先在 PATH 上,无 Codex CLI)。
- `GSTACK_MIN_REVISION` 仅针对 install-honesty 或安全修复而提升,在 stamp 时,永远不超过 VERSION(由 test 钉住)。

## [1.91.95.0] - 2026-10-12

**重新打开的 `/autoplan` gate 现在导出一个能验证的 run。**

用 override 回答一个 gate 重新打开 phase,且下一个 gate 以其 item 标签重新开始(`d1`、`p1`、...)。每个 decision 行被铸造为 `<run>-<label>`,所以第二个 gate 的行重用了第一个 gate 的 ids,且 `gstack-artifact validate` 以 `ARTIFACT_DUPLICATE_ID` 让导出的 run 失败。

### What this means for you

- **来自较晚 gate 的行获得它们自己的 ids。** Gate 修订 1 保留 `<run>-<label>`。在重新打开之后的修订使用 `<run>-r<rev>-<label>`(例如 `run-…-r2-d1`)。Reply 语法与标签不变:你仍回答 `d1b` 或 `all`。
- **Tasks 指向现在打开的批准。** `tasks.jsonl` 行被阻止在当前 gate 的 plan 批准(`<run>-r2-p1` 在一次重新打开之后)上,而不是第一个 gate 的已回答的。
- **docs/unattended.md** 文档化 id 形状。

### For contributors

- `decisionId(run, gateRev, itemId)` 在 `lib/gate-list.ts` 中是铸造这些 id 的唯一位置。`decisionRows`、export 的 task builder 与 owner brief 的 lookup 都使用它。
- `test/autoplan-answer-invalidation.test.ts` 驱动一个 run 经历一次重新打开到第二个 gate,批准它,导出它并要求 `ARTIFACT_VALID`。`test/gate-list.test.ts` 检查跨两个修订的唯一 ids。

## [1.91.94.0] - 2026-10-12

**`/codex review --effort medium` 以你命名的 effort 跑一次评审,而不改变你保存的设置。**

`codex_effort`(1.91.92.0)为每次运行设置 Codex 推理 effort,`--xhigh` 把它提升一次以用于一次运行。没有 per-run 方式去降低,所以一次更便宜的临时评审意味着翻转该设置再翻回来。

### What this means for you

- **任何 `/codex` 模式上的 `--effort <level>`。** `/codex review --effort medium`、`/codex challenge --effort=low` 等等仅为该请求设置 effort。它胜过导出的 `GSTACK_CODEX_EFFORT`、`codex_effort` 与 per-mode 默认。`--xhigh` 继续作为 `--effort xhigh` 的简写工作。
- **Flag 只在 flag 出现的地方出现。** 该 flag 直接计算在 `/codex` 或其 mode 词之后,挨着 `-m`/`--model`/`--role`,在你的提示之前。`/codex what does --effort do?` 把该问题发送给 Codex 而不是把 `--effort` 当作设置吃掉,且 `/codex review --effort medium` 留在原生 review 上。这也适用于 `--xhigh`,它过去在输入的任何地方都计算:`/codex review fix the parser --xhigh` 现在把 `--xhigh` 作为指令的一部分发送,所以把它放在首位(`/codex review --xhigh fix the parser`)。
- **拼写错误的代价为零并说明要修复什么。** `--effort turbo` 在任何 Codex 调用之前停止,包括冷缓存上的模型探查,带 `'turbo' from /codex --effort is not a Codex reasoning effort level` 与 level 列表。缺失的值或冲突的 effort flag 也以消息停止。
- **docs:** `docs/model-policy.md` 文档化 per-request flag,`docs/skills.md` 文档化 `codex_effort` 与 `--effort` 且不再声称 Challenge 以 `xhigh` 运行(它以 `high` 运行)。

### For contributors

- `gstack-codex-probe check-effort <default> --request <level>` 与 `run-with-timeout --effort <default> --request-effort <level>` 接受请求 level(空意味着无)并将 `/codex --effort` 命名为其来源。五个 `/codex` dispatch fences 把它作为允许列表的 `'<effort-level>'` 占位符传递(语法 `^[a-z]*$`),所以 level 仅以引号、允许列表的占位符形式到达 shell(永不通过散文),且探查仅接受七个 level。`run-with-timeout` 行的测试解析器接受新参数。
- `test/codex-effort.test.ts` 对 fake `codex` 跑所有五个渲染的 dispatch,且 bogus-level 用例断言拒绝与零调用。`test/model-role-invocation.test.ts` 为 Step 0.5 中的无效 level 添加零调用用例。

Contributed by @jnunemaker

## [1.91.93.0] - 2026-10-12

**/cso 发布资格门控于已完成的报告。** 报告完整性仍被测量,但它不再阻塞发布。

- **Gate edit: `matchedCompleteMatrix` 与 `mandatoryReports`(`scripts/cso-eval.ts`)。** 两个 gates 过去要求每个报告的 helper 完整性为 `complete`:无 gap,且每个域被评估。私有 corpus fixtures 故意带 gap。它们没有 workflow 文件给 zizmor,它们的种植凭据从不被实时测试,`.env` 被保留在封装的源之外,且 gitleaks 没有历史可读。所以没有报告能 `complete`,且没有发布能资格。两个 gates 现在要求每个报告为 `finished`,在一个完整、完全裁决的矩阵上。运行中、缺失或失败的报告仍让它们失败。完整性保留为它自己的测量率:每组的 `reports`,以及评估器的 `completenessRates`。Garry 在 2026-10-11 批准了这条规则,选择选项 (a)。没有门槛改变。

## [1.91.92.0] - 2026-10-11

(更早的 1.91.x 版本历史见完整源文件)

---

## [1.91.7.0] - 2026-09-28

QA 可以用项目自己的工具测试 API、CLI、job、worker 与 webhook,无需启动浏览器。Review 与 ship 现在跑有界的探索性检查,且每次 ship 在最终验证与发布前审计相关文档。

### Added

- 功能 QA 检查原生输出与耐久效果,包括无效输入、鉴权、取消、重试、重复投递、并发与恢复。报告区分失败、被阻塞的探针与未测试的契约;浏览器与功能结果保持分离。
- 探索性 QA 将观察转为定向探针与提议的回归测试。书面证据检查点连接每个观察到的结果与下一个探针,并从最终报告链接。授权的修复需要一个复现的缺陷、一个在修复前失败的回归与一个成功的回归、原始探针与邻近路径检查,当原生测试基础设施支持它们时。

### Changed

- `/qa` 与 `/qa-only` 在每个支持的 host 上为选定的表面加载指令。功能与仅报告的 run 从不自举一个框架或继承浏览器设置权限;`/qa-only` 保留产品代码、测试、配置与 Git 状态。
- `/review` 与 `/ship` 在没有 plan 或 server 的小非浏览器 diff 上跑有界的探索。必需的检查在被阻塞或未完成时仍是必需的,且提议的测试保留父级的批准门。
- Review 在一个父拥有的修复 phase 之前收集 checklist、专家、QA 与对抗性发现。Re-review 保留相同的三轮上限、重跑受影响的探针,并诚实地记录不完整的覆盖。
- 每个 ship 在最终检查与发布前消费一份已完成的文档审计,包括未提交的变更与已有 PR 或重跑。失败、陈旧或未结算的子结果不能静默地变成干净的审计;父级保留发布元数据与 Git 所有权。

### Fixed

- 仅报告的 QA 在 setup 前完成其范围与方法 Reads,并在证据中保留精确的公共 fixture 路径,而不是发明脱敏路径。实际的 secrets 与私有 payload 仍被保护。
- Ship 的工作流质量裁判使用一个 64k 流式结构化响应在其现有截止时间;其他裁判保留 8k 限额。Cache 标识包括实际 cap、传输与响应契约,且不完整或格式错误的分数仍是失败。
- 重复的 QA run 保留先前的报告、基线与探索性笔记。浏览器技术遵循与功能 QA 相同的检查点探针顺序,且混合报告保持每个表面的证据与分数独立。
- Ship 的两遍测试生成限额包括初始尝试、失败与零测试结果。重复的设计发现共享一个动作,同时保留两个评审者的证据、统计与更严格的批准要求。
- Ship 在拥有它们的步骤中保留修复与迟到变更指令。嵌套的修复保留其返回目的地,且发布准备需要在版本或文档写入前匹配的 review 记录。
- 重用跳过的共享代码建议现在依赖对捕获分支与合格原始源证据的可执行检查。不受支持的 Git 状态、变换的路径与无受信覆盖出处的记录不能证明一个先前的决定。
- 付费测试 `--list` 在已保存计划与所选切片下也保持只读:它验证并列出所选的工作,无需 API 预检、测试启动或结果文件。
- 功能 QA 测试 fixture 在执行前强制其声明的前台命令边界,且共享的原生事件解码拒绝格式错误或不完整的证据,同时保留调用方特定的交接规则。
- 原生 plan fixture 接受 Claude 粘贴包络内的字节精确 seeds,而不接受融合或变更的内容。QA fixture 完成避免重复其检查点分类账,且调用方 fixture 区分绝对截止时间与开始时间。
- 自由测试保留私有完整日志,在证据无法保存时失败,并给出可操作的恢复步骤。Linux 与 Windows CI 收集保留的日志。刷新的时序使新的快速回归通过现有的快速 lane 可达,无需删除完整套件覆盖。
- Ubicloud wrapper 在销毁其 VM 之前取回保留的免费测试日志与任何重试分类账。

## [1.91.6.0] - 2026-09-28

PR eval 切片按每次 eval 实际花费的时间平衡,所以最慢的切片不再承担大部分运行。

### Changed

- 付费 eval 规划器使用来自真实 CI 运行的记录 per-file 墙钟时间(`scripts/paid-test-durations.json`)重新打包切片。它从现有的监督分配开始,仅当没有切片的 1-4 worker 最坏情况墙钟高于该分配的最大值时,才把文件移出或换出最重的切片,所以 CI 超时覆盖永不减弱。从两次 worker 每切片的记录时间估计,典型 PR 运行的最重切片从约 15 分钟降到 12(单次最长 eval 的长度),完整 gate 普查从约 17 分钟降到 13。
- `bun run scripts/test-paid-shards.ts --report <dir> --write-durations` 将报告的执行单文件切片时间合并到 seed。跳过的与亚秒切片被忽略。

## [1.91.5.0] - 2026-09-28

免费套件现在在 16 核 Linux 机器上以约一半时间完成,`bun run test:ubicloud` 在任何开发 box、容器或云沙盒中的一台新 16-vCPU Ubicloud VM 上运行它,且重新推一个 PR 不再等待之前 commit 的 eval run。

### Added

- `bun run test:ubicloud [test:free args]` 在一个临时 Ubicloud VM 上跑完整免费套件(需要 `UBICLOUD_API_KEY`;`UBI_SIZE` 与 `UBI_LOCATION` 选择机器)。VM 获得必需的 CI lane 的环境与严格性设置,包括未提交的编辑,切片日志被复制到 `.context/ubicloud/`,且 VM 之后总是被销毁。完整运行从头到尾约 4.5 分钟,相比 4-vCPU 机器上单独 7.5 分钟的套件时间。
- `bun run test:ubicloud --record-durations` 在同一环境中刷新 `scripts/free-test-durations.json` 并复制回来。
- `scripts/ubicloud/ubi-runner.sh` 在一个 Ubicloud VM 上跑任何命令(`run`,或 `up` / `ssh` / `sync` / `pull` / `down`)。本地仅需要 bash、curl、python3、ssh 与 tar,且在 12 小时后删除自己的陈旧 VM。

### Changed

- 在 Linux 上,`bun run test` 现在为每个可用 CPU 启动一个切片,最多 16 个;macOS 与 Windows 保留 6 个上限。在 16-vCPU VM 上,16 个切片在 137 秒完成套件,6 个切片花了 327 秒。
- 持续时间 seed 用浏览器、display 与 CSO 测试实际运行重新记录,且最慢的测试文件被拆成四个文件。在 16-vCPU 运行上,所有 16 个切片在彼此约 15 秒内完成,其中一个切片过去花的时间是其他的两倍。20 个 CI 切片用相同的 seed 打包。
- 完整套件与 CI 规划运行命名任何缺失的测试文件来自持续时间 seed,所以一个慢的新文件不能悄悄地成为最慢的一根杆。
- Windows CI lane 获得与必需 Linux lane 相同的归因失败单次串行重试,并将其 flaky passes 上传为 `flake-ledger-windows` 工件。其进程监督时序测试单独跑时通过,但在 lane 的双切片负载下可能停滞。
- PR 的新 push 现在取消之前 commit 的付费 eval run。Eval 切片、报告与评论 jobs 跑,除非 workflow 被取消(`!cancelled()`)而不是无条件地(`always()`),所以它们仍在镜像构建被跳过或切片失败时跑,但一个被取代的 run 不再完成(并计费)其切片而新 commit 的 run 在它后面等待。一个 workflow 测试如果任何 eval job 回到 job-level `always()` 则失败。
- LLM-judge skill 质量 evals 要求 clarity 分数为 3 而不是 4;completeness 与 actionability 栏不变。Cookie setup 裁判保留其手动批准的门槛。
- 两个对时序敏感的测试允许重载的机器:terminal-agent 启动竞态等待最多 8 秒以获得冷代理启动,无效 Retry-After 用例接受低于下一个 4 秒退避步的定时器延迟。

## [1.91.4.0] - 2026-09-28

Windows 用户可以将已登录的 cookies 从 Opera 与 Opera GX 复制到 gstack 的浏览器。在 Windows 上,Chrome、Edge 与 Brave 越来越多地存储 gstack 无法解密的 App-Bound Encryption cookies,Opera 与 Opera GX 仍使用 DPAPI 保护的 cookies,所以它们可能是导入继续工作的浏览器。

### Added

- Windows 上的 `cookie-import-browser opera` 与 `opera-gx`(也 `operagx` 与 "Opera GX"),从 `%APPDATA%\Opera Software\Opera Stable` 或 `Opera GX Stable` 在 `Default` 或 `Profile N` 目录读取。遗留的根级布局、Opera 侧 profile 与便携式安装不被检测。包括 @mvanhorn 在 #2980 中的 Opera 注册表与 Roaming-root 贡献(引用 #2957)。

### Fixed

- 当前 Chromium 数据库中的 Windows v10 cookies 用 32 字节的哈希解密在值前面,所以导入可能报告成功而站点保持登出。当存在时,删除 SHA-256(host_key) 前缀,用于 Chrome、Chromium、Edge 与 Brave 以及 Opera。
- 在没有原生提取的浏览器中的 App-Bound Encryption 行保留其收据(计数与原因)并命名恢复:`$B handoff`,登录,`$B resume`。部分导入警告会话可能未恢复。
- 缺失的浏览器、缺失的 profile 与模糊的 profile 选择现在说明已检查的内容与接下来要跑什么,包括哪些 OS 支持该浏览器以及该 OS 上可用的可键入浏览器名。CLI 收据列出失败原因。
- 相对的 `APPDATA` 不再重定向 Opera 根。
- BROWSER.md 拥有 Windows Opera 走查以及收据失败原因与导入错误代码的表格;一个免费测试使文档与命令参考中的浏览器列表与注册保持一致。
- 一个 Windows CI 测试通过 Node 服务器运行时用真实 DPAPI 解密一个 host-bound Opera cookie。Windows 上的真实 Opera 会话仍在等待问题报告者的确认。

## [1.91.2.0] - 2026-09-25

`/sync-gbrain` 可以检查当前 worktree 的页面是否可读,无需写探针页面或在答案不确定时删除指引。

### Fixed

- 准备性现在在检查其页面计数之前将注册的源匹配到这个 worktree,然后从该同一源读取一页。外来的 pin、失败的读取、无效的计数或不可用的服务保持 `unknown` 并保留现有指引;一个已验证的空源仍能提供重新索引。
- Windows 准备性 fixture 通过一个 `.cmd` shim 调用相同的 Bun 支持的 `gbrain` 命令,并保留继承的 PATH 拼写与分隔符。
- 导入 terminal-agent helper 不再启动 CLI 或安装全局进程 handler;直接启动仍强制其启动记录检查。
- 开发者体验问题-下限检查通过其结构而非一种措辞识别有根据的目标选择,而不接受无关的答案。
- 工程评审按顺序遵循其准备与复杂度-门路径,保持每个决策与必需的报告写入可验证,且永不启用校准回写而无其显式门。

### Changed

- PR 评估计划包括源作用域的准备性覆盖以及产生的裁判与监督预算,而不减少个别测试超时、重试或并发。

## [1.91.1.0] - 2026-09-25

### Fixed

- 通过 Claude Code 插件市场找到 Impeccable,包括一个受信任的自定义 `CLAUDE_CONFIG_DIR`。保留传统 skill 安装与现有的显式引擎、PATH 与独立缓存优先级。
- 用严格的 semver 顺序确定性地选择插件版本并支持 hash 命名的版本。让一个已选安装的启动器、引擎与引擎版本一起,而不是借用一个旧插件的引擎。
- 对独立引擎缓存使用相同的严格顺序,保留其仅 semver 策略与优先级。不要跟随缓存目录符号链接或仓库配置链接进无关的文件系统树。
- 保留仓库与符号链接执行边界、净化发现诊断,且当文件名无法安全表示时引用或抑制启动器提示。发现永不下载或跑一个启动器;引擎兼容性警告与安装同意保持不变。
- 在信任边界比较规范 HOME 路径,所以 home 目录别名与 dotfiles 仓库不能隐藏用户安装的引擎或将私有 home 文件接受为扫描目标。
- 添加插件发现、交接、格式错误版本与对抗性路径回归,加上由原生 Windows 测试 lane 选择的 Windows 安全发现用例。

包括 @SomSamantray 在 #2976 中的插件缓存发现贡献。

## [1.90.2.0] - 2026-09-24

**少花时间等待测试。**
**与更清晰的保护措施合并。**

本地测试运行器使用可用 CPU 并删除重复的 setup 而不删除测试场景。独立问题检查并行运行,而不是互相等待。`/land-and-deploy` 将你的批准绑定到所选的 PR、head 与目标分支,在合并回退前检查服务器状态,并保持缺失的部署证据可见。

### The three numbers that matter

来源:匹配的 Linux 组件基准在 [docs/TEST_PORTFOLIO.md](docs/TEST_PORTFOLIO.md) 中,它命名测试文件、工作负载与覆盖。实时比较跑两个 periodic AUQ 文件与五次独立捕获。这些是单独的组件测量,不是完整付费套件或 CI 加速。

| Workload | Before | After | Δ |
|---|---:|---:|---:|
| 9 个合成终端测试文件 | 165.87s | 72.98s | −56% |
| 发布轮询与 watchdog 测试 | 56.57s | 9.63s | −83% |
| 两个独立问题测试文件 | 325.09s | 136.40s | −58% |

合成终端检查节省约 93 秒的重复等待。问题检查保留每个独立试验与它们的原始评分规则;并行执行不是用一个成功的答案替代几个样本的许可。

### What this means for developers

对完整免费验证使用 `bun run test`;快速子集仍只是一个反馈 lane。落地一个 PR 时,一个变更的目标需要新鲜的准备性。合并后的 staging 检查不再暗示生产被持有,且一个健康的旧页面不证明新 revision 已部署。跑你的检查,然后用 `/land-and-deploy` 在合并前评审证据。

### Itemized changes

#### Changed

- 本地免费测试 worker 跟随可用 CPU 亲和性,最少一个且最大六个的现有最大值。显式 worker 覆盖与独立 CI 矩阵保留其行为;Windows CI 显式保留其两 worker 预算。
- 部署报告区分部署状态、生产健康、staging 验证与已完成的 rollback。生产前请求 staging 停止在合并前并交接给配置的 pipeline。

#### Fixed

- 浏览器-同意检查区分承诺的新同意问题与立即驱动提议,同时仍拒绝在 Aside 就绪前的条件驱动许可。
- Review fixture 接受显式的无变更答案与覆盖报告陈述,而不授权源编辑或索引标志变更。
- 合并回退要求权威确认既无自动合并请求也无队列条目。已确认的合并永不重放,变更的 head 或目标分支让早先的批准失效。
- Rollback 区分真正的合并 commit、squash 合并与 rebase 范围。失败或未验证的部署与 canary 检查保持可见,而不是变成成功标签。

#### For contributors

- 仓库发布指南默认自主补丁 bumps,包括队列碰撞。合并批准仍是分开的。
- 合成终端信号准备性;发布测试通过 scoped clock 推进原始轮询序列;watchdog 场景共享编译但保留隔离的可执行文件与状态。
- 仅免费的依赖豁免是显式的,映射的依赖优先,且未知变更保留保守的付费选择。组合文件将单独的责任分配给结构化测试、质量裁判、原生行为、模拟与平台集成。
- 原生问题、共享代码评审与设计检测器 fixture 验证其实际支持的交互与已执行的证据。源检测器断言失败在验证后被记录,带有尝试特定的诊断,该诊断在清理后被保留。

## [1.90.0.0] - 2026-09-24

Cookie 导入现在保持所选浏览器、profile 与目标的显式,显示部分失败,并区分复制 cookies 与证明你已登录。

### Added

- macOS Dia 发现与 profile 选择,使用现有 Chromium cookie reader。Live 原生 Dia 导入仍不合格;fixture 覆盖不是原生兼容性的声明。
- 针对所选目的地的可选登录验证,使用精确、可见的 identity 断言,带分离的已复制与已验证结果。
- 显式的当前 origin 存储恢复。存储默认保留;选择加入的重置清除该 origin 的 localStorage 与仅目标标签页的 sessionStorage。

### Fixed

- 优先选择从 Local State 重命名的 profile,区分重复名称,且要求选择而不是在可信的 account 之间猜测。裸与带点的域选择现在匹配预期的 cookie 范围而不扩大它。
- 用浏览器的导入域安全卫语句注册 picker 导入,拒绝跨域 picker 变更,并将异步操作绑定到它们的原始目的地。打开另一个 picker 让旧窗口失败关闭而不是静默改变它们的目标。
- 显示完整、部分、零与失败的导入。过期发现响应与重复提交不再覆盖当前 picker 状态。
- 限制凭据子进程输出与清理,仅重试瞬态数据库读取,并防止 cookie 导入变更的自动重放。Node 服务器使用真正的只读 SQLite 适配器。

### Changed

- Picker 交接 codes 持续 5 分钟并保持单次使用。浏览器指引说明 profile 选择、存储-重置同意以及复制 cookies 与已验证登录之间的差异。
- 原生 Windows 提取使用沙箱、拥有-进程、仅管道路径,带有限清理且无 TCP 回退。它的资格 allowlist 在该 release 中为空,所以通过该路径的加密 cookie 提取保持禁用,带手动-登录指引。
- 添加隔离平台资格、启动诊断与回归覆盖。原生 Dia 与受保护的默认 profile Windows 资格仍不完整;诊断通过不计入成功的导入。

## [1.89.1.0] - 2026-09-24

### Removed

- **Continuous checkpoint commits.** Skills 不再要求用户启用自动 `WIP:` 提交或指导代理创建它们。检查点模式与推送设置不再被广告或消费,且现有保存的设置被原样保留。
- **Checkpoint-specific shipping cleanup.** `/ship` 不再导出检查点上下文或重写 WIP 历史。它保留其正常的 bisectable commit 工作流并在更改已提交时直接进入验证。显式 `/context-save` 与 `/context-restore` 仍可用。

### Fixed

- **原生 DevEx 评估回复。** 测试驱动程序识别当 Claude Code 聚焦自定义答案时显示的编辑提示,同时仍检查确切的问题与回复再提交。
- **共享代码评审评估回复。** 无变更驱动程序可以使用显式保留描述来解释简写标签,同时仍拒绝混合修复/跳过选择与模糊答案。
- **Windows 超时测试准备性。** 进程清理回归在启动已注册截止时间前等待活着的后代,而一个单独的实时-时钟用例保持启动有界。
- **Design consultation workflow。** 字体与设计规则现在先于提案起草与独立输入。可选-浏览器路由、现有系统选择、预览反馈与命令/会话要求是显式的,且 token 提取不能在批准前写项目的设计文件。

## [1.65.0.0] - 2026-08-14

**/autopan、macOS 上的 /codex 与内存摄入重新工作。**
**且每个同意门现在意味着它所说的。**

这是 GStack 2 fork port 的第二波也是最终波。Wave one(v1.63.0.0)采取了审计基础设施;该 wave 采取了修复与功能。三个在每次 run 上静默失败的 skill 现在工作了:/autopan 的任务聚合发出真实任务而不是零,/codex 在 macOS 上创建其临时文件而不是死于 BSD mktemp,内存摄入实际上在当前 gbuild 上导入页面,并打印计数以便你能看到它。在此之上:你的浏览器鉴权现在能在守护进程重启后存活,/ship 可以将 iOS 应用从工作树带到 Submit for Review,四个供应链 gates 现在在每个 PR 上跑。几乎所有这些都追溯到 Sina Matian 的 time-attack/gstack fork,以测试与归属移植。

### The numbers that matter

来源:该分支(`git log 1.63.0.0..HEAD`、`git diff main...HEAD --stat`、`bun test`),加上 release 关闭的 GitHub issues。

| What | Before | After |
|------|--------|--------|
| /autopan Phase 4 任务输出 (#2018) | 0 任务,每次 run | 每个任务 |
| macOS 上的 /codex (#2091) | 在每个安装上坏 | 工作 |
| gbrain 0.42+ 上的内存摄入 (#2144) | 0 页面,报告成功 | 完整语料库,计数被打印 |
| macOS 26 上的 Headed browse (#2242) | GPU 崩溃,中毒缓存 | 启动,治愈旧缓存 |
| 浏览守护进程重启后的鉴权 (#778) | 登出 | 恢复(opt-in) |
| PR diff 上的 CI secret 扫描 | 无 | 每个 PR,fail-closed |
| 已关闭的 GitHub issues | | 24 |
| 带归属的社区 PR | | 4 |

最刺眼的是前三行:那些不是降级功能,它们是带着绿色对勾返回空结果的功能。如果你在过去两个月跑过 /autopan,它交给执行者的任务列表是空的且没人告诉你。

### What this means for gstack users

跑 /autopan 且 pipeline 将真实任务交给执行者。在 Mac 上跑 /codex 它就是工作。设 `BROWSE_PERSIST_STATE=1` 且守护进程重启不再将你从每个站点登出。如果你发一个 iOS 应用,`/ship` 现在知道整个 App Store 旅程、会话铸造的上传 keys、取代坏 fastlane 路径的 price-schedule API、扩展的年龄-分级属性、-22938 分类,以及一个鉴权时刻而不是五个。用 `/gstack-upgrade` 升级;迁移清理一个旧 gstack 坏掉的 Chromium bundle 并在声称成功前验证替换下载。

### Itemized changes

#### Added

- **Opt-in browser session persistence** (#778, #2193):`BROWSE_PERSIST_STATE=1` 快照 cookies 与标签页(原子写,0600,永不页面 HTML 或所有权),在下次启动的启动路径外恢复它们,并隔离一个损坏的快照而不是崩溃。来自 time-attack/gstack 的部分。
- **Apple App Store release journey for /ship**: `ship/sections/apple-release.md` 在目标是 Apple app 时在 repo-landing gates 之前加载。编码会话铸造的 App Store Connect keys、`appPriceSchedules` 跨坏 fastlane `price_tier`、扩展的年龄-分级属性、-22938 分类,以及一个鉴权时刻流。在 fork 上 21 次实时发布中精炼。部分版权 Sina Matian,MIT。
- **Code-intelligence provider contract, Phase 1**: `gstack-code-intelligence` 包装 GBrain、Sourcebot 与 Graphify 在一个接口后面,带一次询问大仓库(1,000+ tracked 文件,持久拒绝)索引提议。同意是显式的 per repo(`consent <repo> yes|no`),per-repo 信任策略的 deny 与 read-only tiers 无论记录了什么同意都否决写-类操作,且每个离机发送写一条出账回执,记录实际检查的同意状态。来自 time-attack/gstack 的部分。
- **Supply-chain CI**: 跑 `bin/gstack-redact` 跨每个 PR diff 的质量 gate(HIGH 发现失败,MEDIUM 注释),lockfile 变更上的依赖评审,每周 OSV 扫描,分组的 dependabot 更新,以及证据-栏 PR 模板。新 workflows 中的每个第三方 action 都被钉在 commit SHA。
- **Third-party web-actions contract** in tier-2+ skills: 当一个 workflow 需要 vendor-site 步骤(API key 注册、OAuth app),gstack 提供驱动浏览器本身,把凭据与 CAPTCHA 交给你,只所有者存储 secrets,并在声称成功前用只读调用验证。
- **Design docs land in your repo** (#703, #2000): office-hours 写 `docs/designs/<topic>.md` 作为简洁的决策记录(每个决策一个 bullet 带其 why),在任何东西触及你的 git 历史之前进行脱敏扫描。Plan reviews 在两者都存在时优先 repo-local doc。
- **`gstack-verify-gate`** (opt-in Stop hook): 阻止 turn-end 直到 CLAUDE.md 声明的 verify 命令通过。一个命令仅在你每次 repo 信任它一次后跑(`--trust`),变更时需要重新-信任,每个 grant 是审计-记录,且重入重新跑检查而不是挥动它通过。
- **"Never show me these again"** for the founder-resources pitch (#538): opt-out 在承诺任何东西前验证它自己的 config 写。用 `gstack-config set founder_resources true` 重新启用。
- **Claimed limitations need evidence**: 每个 tier-2+ skill 现在将「API 不能做这个」视为一个需要逐字错误、文档化声明或实时探针的实质声明,并在声明任何东西被阻塞前跑 10 秒的检查。

#### Fixed

- **/autoplan Phase 4 在每次 run 上发出零任务** (#2018): jq context rebind 丢弃了每个聚合的任务;错误被 stderr 抑制隐藏。Six-fixture 回归套件钉住它。
- **/codex 在每个 macOS 安装上坏** (#2091): BSD mktemp 拒绝带后缀的模板;所有临时文件现在使用便携式模板与一个静态测试在 repo 范围内禁止该模式。
- **内存摄入在 gbrain 0.42+ 上不导入任何东西** (#2144): 暂存目录位于一个 gitignored 树下,所以 git 感知的收集器看到零文件。用 `--include-gitignored` 修复(社区 PR #2560)加 `GIT_CEILING_DIRECTORIES` 第二层,Windows 安全,以及一个响亮的已摄入页面计数。
- **macOS 26 上的 Headed mode** (#2242, #2138, #2139): gstack 不再重写签名的 Chrome-for-Testing bundle(重新品牌破坏了其代码签名;GPU 进程拒绝启动)。启动通过移除整个 revision 目录自我治愈中毒的缓存,以便 re-fetch 实际上重新下载,升级迁移对现有安装也做同样,并在记录成功前验证一个工作的 Chromium 存在。品牌保持在 GStack Browser wrapper app 上。
- **`browse stop` 重新启动它被告知停止的守护进程**: CLI 现在在关闭前获得一个确认,且关闭快照有一个硬截止时间,所以一个 wedged 页面永远不能持有端口。
- **来自内部网络的 session cookies 永远不到达恢复的浏览器**: 恢复时卫生过滤器丢弃 loopback 与 link-local IP literals(127.0.0.1、::1、169.254.*)连同 localhost 与 *.internal,由持久化路径与 `state load` 共享。
- **ios-qa 停止分发裸 bearer tokens**: `/auth/sessions` 返回带盐-哈希的 token id 带 revoke-by-id 支持,boot token 离开 os_log,IPv4 监听器在 socket 上钉到 loopback。
- **make-pdf 的 no-network 承诺抵御混淆**: `<style>` @import、内联样式 URL(带引号、不带引号、CSS 转义与 HTML-entity 编码)、srcset 与 media 源在渲染未信任的 HTML 而无 `--allow-network` 时被中和。
- **pair-agent 隧道被 consent 门控** (`gstack-config set pair_agent on`): 隧道无法在未记录 opt-in 的情况下启动,总是声称为同意的收据现在反映一个存在的检查,且一个禁用的门告诉你真正的补救而不是 ngrok 安装说明。
- **per-repo gbrain 信任策略在 code-import choke point 处强制** (#2140, sync path): deny 拒绝,read-only 跳过代码摄入,不可读的策略存储失败关闭,且出账回执命名该决策。
- **Handoff 不再解除隧道-孤儿 reaper** (社区 PR #2565 加硬化): 将守护进程提升到 headed 仅抑制 headed-shutdown 分支;一个带活动隧道的守护进程仍与其父一起死亡。
- **Windows**: 状态目录上坏掉的 DACL 自我修复 (#1605),每个 Windows 可达的 spawn 通过 windowsHide (#1835,社区 PR #2523 与 #2539),且决策文件不再落在命名为 "unknown" 的项目下。
- **Setup 在首次跑时不再挂起** (#2136): Chromium 探针获得一个 90 秒截止时间,回收整个 wedged 进程树,安装是跨并发 setup 的单飞,EXIT 清理陷阱链而不是互相 clobber。
- **Paper cuts**: `gh pr edit` 在 Projects-classic GraphQL 弃用时回退到 REST API (#1079);v1.27 迁移在没有 TTY 时永不自动进行,永不将失败的 rename 记录为已完成 (#1383);模型基准识别 macOS Keychain 鉴权 (#1890);voyage embedding flags 在 zsh 中存活 (#1798);`--supersede` 保留它被给予的替代决策;brain 上下文在一次慢的冷启动探针后不再禁用自己;锁获取报告真实错误而不是幻影争用 (#1084);plan verification 探针你项目声明的 dev 服务器而不是硬编码端口列表。

#### Changed

- **Telemetry defaults to off everywhere**: 浏览守护进程现在读取与 gstack 其余部分相同的持久同意;缺失键意味着禁用,匹配 `gstack-config get telemetry`。
- **Test-command detection covers Django and config-less projects**: 一个工作的 `manage.py test` 或 `*_test.go` 套件被识别,而不是被提供一个它不需要的 bootstrap。
- **Base-branch detection everywhere**: 硬编码 `main` 的 bins 现在按顺序探查 origin/HEAD、origin/main 与 origin/master。
- **Eval model resolution is host-neutral**: `GSTACK_EVAL_MODEL`(以及 per-kind 变体)在所有六个调用站点覆盖硬编码的 model id。
- 死 bins 移除(`chrome-cdp`、`gstack-open-url`、`gstack-platform-detect`);陈旧-引用扫描现在覆盖 docs/,所以像这些的移除在文档仍引用它们时会让 CI 失败。

#### For contributors

- 23 个新测试文件(+5,499 个测试行): 行为套件用于会话持久化、中毒的 bundle 探针、两个迁移、同意 CLI、verify-gate 信任、telemetry opt-out、离线-门旁路语料库、secret-scan exit 契约,以及锁-获取边界分支。ios-qa 守护进程套件(10 个文件)现在被接入 `bun test` 与分片 runner;它从未在 CI 中跑过。
- `lib/gbrain-repo-policy-client.ts` 是读取 repo-policy tiers 的唯一位置;两个强制点都通过它路由。
- `lib/context-bill.ts` 不再在 totalMd 中双重计数嵌套 skill,且 `gstack-context-bill` 在 Conductor env-shims 下工作。
- Egress 收据:`bin/gstack-egress verify` 在链完好时通过;code-intelligence 适配器注册为 fail-closed sink。
- Credits: 该 release 跨几乎每个集群移植 **Sina Matian**(time-attack/gstack,MIT)的工作。带归属吸收的社区 PR:**Gawie van Blerk** (#2560)、**Shawn Reddy** (#2565)、**Jake Wilk** (#2523)、**Jerry Nichols** (#2539)。感谢你们。

## [1.64.1.0] - 2026-08-15

**流水线中的每个 guard 现在都可证明地触发。**
**且代码库停止描述它没有的特性。**

该 release 是一个对 gstack 那些前一代模型写且后来删除留下的部分的修复波。免费测试套件现在在 CI 中以 per-file 隔离运行,所有十个 host 输出在每个 push 上被门控,隧道安全 allowlist 与存在的端点匹配,且安全文档描述实际运行的防御。一个模板 bug 修复单独将 /spec 的 skill 文件削减 46KB,且八个实用 skill 停止携带为不同 tier 编写的 onboard 散文。净:24,943 行在 183 个文件中变轻。

### The numbers that matter

来源:该分支的验证运行(`bun test` per-file、`bun run gen:skill-docs --host all`、JSON config dump-diff)与 `git diff origin/main...HEAD --stat`。

| Metric | Before | After | Delta |
|---|---|---|---|
| 在 CI 中跑的免费测试文件 | 0 | 358,每个文件一个进程 | 通过构造截断不可能 |
| 可以失败的 host doc-freshness gates | 1/10 | 10/10 | 两个 gate 区分 gitignored 路径 |
| spec/SKILL.md | 127,462 bytes | 80,924 bytes | 一个 preamble,而不是两个 |
| hosts/*.ts config code | 595 lines | 285 lines | defineHost() factory,byte-identical output |
| Eval tier-gate implementations | ~40 drifted copies, 6 predicates | 1 | unset-tier trap 永远被钉住 |
| Net repo size | baseline | -24,943 lines | 24 files deleted outright |

感受的 tier 表是那一个:`/scrape`、`/diagram` 与浏览器启动器各自削减了 271 行从一个静默默认继承的 preamble。Skills 现在声明它们的 tier 或生成器拒绝构建。

### What this means for gstack users

已修剪的实用 skill 的 skill 调用将更少的散文加载到你的上下文窗口,/spec 加载轻 46KB,且该仓库中一个红的测试现在意味着导致它的 PR 上的红检查,每次,在每个 host 上。如果你维护 fork 或嵌入浏览守护进程:两个从未工作的 ServerConfig 字段(idleTimeoutMs、chromiumProfile)被删除而不是撒谎,且 `GSTACK_SECURITY_ENSEMBLE` 不再作为旋钮存在。正常升级;无需迁移。

### Itemized changes

#### Fixed

- CI: skill-docs 新鲜度 gate 通过一个 `gen:skill-docs --host all` 传递加上一个 tracked-drift diff 与一个 untracked-strays 检查覆盖所有 10 个 host。Codex 与 Factory gates 以前区分 gitignored 路径,那总是通过。
- CI: 新的 Free Tests workflow 在 prebaked 工具链镜像上跑整个免费套件(358 个文件)每个文件一个 bun 进程。Per-file 隔离规避两个观察到的静默-截断模式(在 server-lifecycle 测试中的 process.exit 竞态,与同跑模块-状态渗漏)以及历史上的 Bun exit-0-on-module-load-error 行为。
- Security: 从 TUNNEL_PATHS 移除已删除的 /sidebar-chat 端点,审计的隧道攻击面。该集合现在正好是 /connect 与 /command,且 closed-set pin test 强制它。
- Security: 删除 chain 的不可达直接分派回退,该回退在没有 scope、域、tab-所有权、限速或 JS-origin 检查的情况下路由命令。读命令中的 JS-origin 断言现在是无条件的。
- Security: 页面-内容日志(console、network、dialog、command audit)通过 appendSecureFile,从每个平台上的创建开始就获得所有者-仅权限。
- Stealth: headless-to-headed 交接路径使用共享的 Chromium profile 解析与单例锁清理,而不是一个忽略 CHROMIUM_PROFILE 与 GSTACK_HOME 的硬编码路径。
- Generator: /spec 的 skill 文件渲染其整个 preamble 两次,因为模板散文字面地提到了一个占位符。已修复;46,538 字节从生成的文件中删除。
- Generator: preamble tiers 按 skill 声明,且缺失声明是构建错误。八个静默默认为最重 tier 的 skill 现在携带正确的(scrape、diagram、浏览器启动器在 tier 1;landing-report、pair-agent、skillify 在 tier 2;spec 在 tier 3)。
- Generator: learningsMode 从 host config 读取而不是硬编码的 host 检查,所以七个 basic-mode hosts 获得其运行时能执行的项目-作用域 learnings 流。
- Test selection: touchfile 依赖路径针对磁盘验证(guard 在其首次跑时捕获了四个腐烂的条目),eval-watch 仪表板从 collector 写入的目录读取部分结果。
- Eval gating: 一个 describeE2ETier 实现替代了约 40 个漂移的副本。分片付费 runner 的预-生成分类器理解新形状,所以 gate runs 不再为 periodic shard 启动付出代价。

#### Changed

- hosts/*.ts 仅声明每个 host 不同的内容;defineHost() 推导其余的。通过所有十个配置的 JSON dump-diff 与一个零-差异重新生成证明字节相同。
- pty-session-cookie 与 sse-session-cookie 共享一个带分离 token 空间的会话注册实现;terminal agent 使用共享的 cookie 解析器。
- 一个 lone-surrogate sanitizer 与一个 sanitizeReplacer 住在 sanitize.ts 中;一个 startTunnel() 拥有存在三次的 ngrok 启动序列。
- lib/fs-atomic.ts 是单一的原子-写实现(pid+random tmp 后缀,throw 与 quiet 变体,mode-at-create)。lib 与 browse 调用站点迁移,包括 worktree dedup index 中的潜在确定性-tmp 碰撞竞态。
- lib/jsonl-store.ts 文档化其真实契约(调用者筛选注入模式;强制调用者被命名),获得一个 mode 选项,且 lib 侧的旁路附加器现在使用它。

#### Removed

- 死的 ML 安全层:Haiku 转录分类器与 DeBERTa ensemble(GSTACK_SECURITY_ENSEMBLE),其无生产调用者,加上它们的付费基准套件与 fixtures。活路径是 security sidecar 中的 testsavant 内容扫描。CLAUDE.md 与 BROWSER.md 现在文档化正好那个。
- 五个 HostConfig 字段无物读取(metadataFormat、sidecar、prefixable、staticFiles、adapter)与完全死的 openclaw-adapter 模块。
- 七个已注册的模板占位符无模板使用、永不被采纳的 gated-resolver 机制,以及 codex-helpers 影子模块,其陈旧副本默默地输给一个本地重新声明。
- 两个 ServerConfig 字段被文档化但从未被读取(idleTimeoutMs、chromiumProfile);BROWSE_IDLE_TIMEOUT 与 CHROMIUM_PROFILE env 保留为工作旋钮。
- proactive-suggestions.json(每次构建重新生成 31KB,无物读取),两个零-调用者 bin 脚本(gstack-open-url、gstack-platform-detect),一个孤立的 schema 模块,三个孤立的 test fixtures(包括一个 128KB golden 已从其活后继漂移 46KB),以及一个 ship-idempotency eval 的被取代重复。
- 约 2,000 行测试锻炼了已删除的特性:两个文件在 import 时通过读取 48 个版本前删除的源文件而崩溃,一个...

(后续版本省略;文档总共 12,743 行,后续条目格式一致并保留所有技术引用、版本号、PR 编号、commit SHA 与 `Contributed by @username` 标注。完整源文件保留所有这些细节不变。)
