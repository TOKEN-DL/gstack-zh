# 测试组合:覆盖所有权和重复工作

审计源:`06ed920a974809ebedc6bcbbe402fb81f5944598`,2026 年 9 月 24 日。这是覆盖清单和有针对性的重构,而不是声明每个测试断言都可以与关于同一 skill 的另一个断言互换。

## 每种证据的一个所有者

测试可以共享设置或检查同一公共捕获。当合约需要独立试验时,它们不得将一次捕获计两次。特别是,免费的解析器测试、提示质量分数和完成的实时工作流即使提到同一 skill,也证明不同的事情。

| 责任 | 所有者 | 此证据不替代的内容 |
| --- | --- | --- |
| 选择、预算、流程所有权、清理、原生事件解析和记录的失败控件 | 免费 runner 和 fixture 回归 | 选择或完成正确工作流的实时 model |
| 生成的文件、主机奇偶性、section 清单和完整的输入身份 | 免费生成和结构测试 | 实际加载所需部分的智能体 |
| 提示清晰度和评分标准质量 | 该完整提示的注册质量 judge | 工具执行、原生确认或完成的报告 |
| 原生第一个问题的格式和内容 | 实时 SDK/PTY 问题捕获 | 已回答的问题或已完成的工作流;收据明确表示 `workflowCompleted: false` |
| 随机一致性和冗长/雕刻比较 | 独立捕获,具有单独稳定性和 A/B 或 oracle | 一个成功的样本被重用为三次试验,或一个提示版本代表另一个 |
| 决策、发现和报告完成 | 每个 skill 的原生工作流 fixture | 单独的第一个问题、没有原生证据的屏幕文本或通用问题计数 |
| 离线部署和金丝雀报告构建 | 显式模拟的工作流 fixture | 真实的 GitHub 合并、部署、回滚或生产健康检查 |
| 多阶段排序和交接 | 生产阶段发布 hook,由免费 `test/autoplan-publication-guard.test.ts` 固定;自 2026-09 审计以来没有付费链评估(TODOS:"没有付费评估运行完整的 /autoplan 链") | 完成 CEO → Design → DX → Eng 的实时 model |
| 外部评审、其他 model provider、浏览器引擎和平台行为 | 它们各自的实时集成 fixture | 提示奇偶性或模拟传输 |

覆盖功效实验保留其完整的 fixture/model/arm/trial 矩阵。安全用例保留其源、路径、套接字、进程和租约标识。这些是不同的场景维度,而不是要删除的重复工作。

## 检测器所有者测试

捕获的付费失败成为其检测器的所有者测试中的一行(`describe` 块或表条目),而不是新的每事件文件;`test/test-of-test-ratchet.test.ts` 强制执行此。2026-09 审计后的所有者([证据](test-audit-2026-09.md)):

| 检测器 | 所有者测试 |
| --- | --- |
| `hasStaleFillRaceFinding` | `test/ceo-section-loading-fixture.test.ts` |
| `generateModelOverlay` / `resolveModel`(覆盖短语) | `test/model-overlays.test.ts` |
| `coverageAuditVerdict` / `coverageAuditReadEvidence` | `test/coverage-audit-evidence.test.ts` |
| Autoplan 阶段完成(`autoplanPhaseCompletions`) | `test/autoplan-phase-observer.test.ts` |
| `findNativeAutoDecision` 和自动决策状态 | `test/native-auto-decide.test.ts` |
| `claudeOutsideExecutions` | `test/outside-voice-evidence.test.ts` |
| `engStep0Boundary` / `engSetupAUQ` / `engFirstReviewAUQ` | `test/eng-first-review.test.ts` |
| `hasNativePlanTerminal`(完成和交接) | `test/plan-count-completion.test.ts` |
| `createPlanCountPermissionGuard` | `test/plan-count-file-permission.test.ts` |
| CEO 模式选项解析(`ceo-mode-option`) | `test/ceo-mode-option.test.ts` |
| 计划范围选择(`plan-scope-selection`) | `test/plan-scope-selection.test.ts` |
| `planCountPrerequisitePick` | `test/plan-count-prerequisite.test.ts` |

## 功能 QA 合约映射

下面的确定性所有者保护失败边界;他们的实时伙伴证明智能体遵循它。共享的 fixture 或捕获的事件不会取代独立的实时试验。所有免费所有者在 `bun run test` 中运行;快速资格取决于测量的持续时间或显式的 `QUICK_CORE` 条目,而此表。

| 合约 | 确定性所有者 | 必要的实时边界 | 主机和 lane |
| --- | --- | --- | --- |
| 没有浏览器设置的 CLI/API/webhook QA | `qa-functional-fixture`、`qa-functional-evidence`、`qa-lazy-sections` | `skill-e2e-qa-functional`:CLI 和 webhook 报告会话 | Linux/macOS 免费;选定的 PR gate;仅在策划时的 Windows |
| 仅报告保留本地和远程权限 | `qa-only-capability`、`qa-functional-observer`、`qa-functional-observer-atomic`、`qa-caller-authority` | 具有合成拥有的端点/身份验证的独立仅报告会话 | Linux 内核观察;免费回调控件加上选定的 PR gate |
| 修复重现缺陷、添加失败的回归并重新检查相邻行为 | `qa-fix-loop-fixture`、`qa-functional-evidence` | `skill-e2e-qa-functional-fix`:CLI 和 webhook 修复会话 | 免费控件加上选定的 PR gate |
| 评审和 Ship 实际探索 | `qa-exploratory-callers`、`qa-caller-report-observer`、`qa-checkpoint-evidence` | `skill-e2e-qa-callers`:实际的 Review/Ship 调用者 | 免费捕获加上选定的 PR gate |
| 烟雾过期保留必需的计划检查 | `qa-deadline`、`qa-deadline-selection`、`qa-browser-deadline-evidence` | `ship-exploratory-plan-checks` | 免费截止日期/调度控件加上选定的 PR gate |
| 后期更改使受影响的结果无效 | `qa-caller-freshness-order`、`qa-deadline-publication-observer`、`shared-libs-revalidation-prompt` | `ship-exploratory-late-input` 和现有的后期输入文档交接 | 免费陈旧输入控件加上选定的 PR gate |
| 文档在发布之前完成并尊重受保护的文件 | `docsync-authority`、`docsync-atomic-writes`、`docsync-report-interface`、`docsync-lifecycle-interface` | `skill-e2e-ship-docsync`、`skill-e2e-docsync-spawned` | 免费状态/权限控件;注册的 gate/周期场景保留其 tier |
| 取消在另一次尝试之前排空拥有的工工作 | `shared-libs-cancellation`、`session-runner-stream-lifecycle`、`agent-sdk-runner`、`paid-shard-settlement` | 现有的实际共享库/SDK 调用者场景 | 免费真实回调/进程控件;注册的实时 gate/周期试验保持独立 |
| 缺少工具或不完整的结果永远不会成为经过验证的覆盖 | `qa-probe-gates`、`qa-supervision-selection`、`test-free-shards`、`test-free-shards-capture`、`paid-shards` | `ship-exploratory-unavailable` 和现有的报告边界会话 | 免费负面控件加上选定的 PR gate;不受支持的主机保持未执行 |

没有后缀的名称引用 `test/<name>.test.ts`。使缺失、过时、重复、已选择但未启动、格式错误/截断和观察者溢出控件与合法的空选择保持不同。文件还原不能替代写入观察,干净的本地树无法证明外部请求没有进行变更。Fixture 端点和凭据必须是合成的和拥有的;明确授权的功能请求仍然允许。

功能 fixture 将其现有的封闭命令策略注册为原生 PreToolUse hook,因此在执行之前拒绝不受支持的请求。回调回归使用原生 hook 输入调用注册命令,观察隔离的变更目标,并允许拥有的 webhook 阳性控件。这是一个命令边界,而不是任意目标代码的沙箱。其私有 CLI 配置位于观察到的产品树之外,并且 fixture 的现有清理拥有两个目录。

Review/Ship 观察现在使用与 QA 检查点和文档相同的严格原生事件解码器。特定于调用者的交接/新鲜度解释保持独立。在将三个偶然的错误措辞断言替换为拒绝检查之前,运行了原始的缺失、孤立和重复调用控件;现有的阳性归因案例仍然运行,并且已完成的 ID 重用负面控件可防止不完整的证据变为绿色。

## 完整清单,而不仅仅是快速子集

在审计修订时,所有 1,124 个跟踪的 Bun 测试文件分为 1,010 个免费文件和 114 个付费文件。付费清单包含 207 个已注册的 E2E 选择 ID(84 个 gate 和 123 个周期)和 25 个主要质量 judge ID。ID、文件、model 调用、样本、尝试和已执行的 Bun 测试是不同的计数;使用运行清单和收据,而不是将一个替换为另一个。

完整的 gate 和周期清单计划跨 112 个唯一文件的 153 个文件进程。四十一个文件同时出现在两个清单中,因为它们包含混合 tier 用例;这不能建立 41 个重复的场景。大脑隐私和发布幂等性具有现有的明确排除,iOS 需要硬件,并且 spec 质量文件包含一个 TODO。iOS fixture 的原生 Swift 烟雾测试在此 Bun 文件人口普查之外,需要其平台运行时。不可用或被排除的工作不是覆盖。

现有来源仍然是权威的:

- `test/helpers/paid-test-set.ts` 拥有免费/付费文件边界。
- `test/helpers/touchfiles-data.ts` 拥有注册的依赖项和 tier 数据。
- `scripts/test-pr-profile.ts` 拥有故意部分的 PR profile。
- `test/helpers/eval-budgets.ts` 拥有监督和重试例外。
- 免费和付费分片 runner 拥有清单、调度和对账。

`test:quick` 和 `test:pr` 是反馈通道,而不是完整的发布验收。此重构不会删除用例、缩小样本、更改 tier、降低阈值、缩短生产截止日期或将检查移至以后的节奏。

## 已删除的重复工作

**合成终端启动。** 9 个文件中的 42 个假终端场景仅在安装处理程序后才发出其现有的就绪标记。它们不再花费真实 CLI 的八秒宽限时间等待已经准备好的假人。原生 CLI 宽限时间、终端几何形状、观察延迟、场景输入和所有现有断言保持不变。

**发布轮询。** 无效/不可用的日志案例仍然通过所有 40 个轮询间隔调用真实的 hook 和脚本读取器。范围序列化的假时钟删除时钟睡眠,而控件验证完整的逻辑截止日期、后期到达以及成功和失败后的恢复。真正的延迟日志和 shell 传输案例仍然使用实时。

**原生 watchdog 设置。** 每次测试文件编译一次相同的源,并将可执行文件复制到每个案例的隔离目录中。每个 watchdog 仍然执行;仅合并相同的编译器结果的重复检查。可变的工作目录和清理所有权永远不会被合并。

**独立的实时捕获。** 通过现有的三查询信号量并发启动所有三个一致性捕获以及两个 A/B 臂。在清理之前等待每个结算,保留兄弟失败,并尝试清理每个拥有的目录。一致性判断保持顺序;A/B 最多有两个同时 judge。免费回归行使实际的注册回调、原生捕获收据、信号量和 judge 请求构建器。

**Runner 所有权。** 路径规范化属于现有的共享严格输出实用程序,而不是免费 runner。付费 runner 不再导入免费 runner 来使用它。确切的仅免费豁免涵盖免费 runner 和免费 AUQ 重放 worker,具有付费导入闭包和选择控件。映射的依赖项仍然获胜;未知依赖项保留广泛的回退。不引入目录范围的豁免。

**本地调度。** 默认的免费 worker 使用可用的 CPU 亲和性,下限为 1,上限为 6。每个分片在内部保持顺序,并且显式的 `GSTACK_FREE_JOBS` 覆盖保留其先前的含义。单独的 20 台机器 CI 计划保持不变。Windows 免费测试 CI 明确保留其双 worker 预算,而不是继承本地默认值;本地 worker 收益不是 CI 收益。

## 测量合约

在同一台机器、运行时、启动环境和工作负载上比较原始和编辑的代码。保留失败和重试记录。计算完整的用例/样本清单并分别报告跳过和不可用的平台。不要从经过时间中减去失败,也不要使用较小的选择作为完整套件变快的证据。

### 功能 QA 清理测量 — 2026 年 9 月 28 日

在相同的四 CPU Linux 机器上,使用 Bun 1.4.0、Node 22.20.0 和 Claude Code 2.1.251,现有的持续时间记录器测量所有 1,113 个免费文件。刷新的种子选择 931 个文件以获得快速反馈:90 个新包含和 20 个新排除的测量成本,净增加 70。没有文件保持未分类。所有 182 个慢速文件仍保留在完整套件中。功能命令观察器、检查点解码器和日志捕获控件是显式的快速核心案例;每个测量在两秒以下。

| 现有命令 / 尝试 | 执行范围 | 结果 | 持续时间 |
| --- | --- | --- | ---: |
| `bun run test:free --record-durations` | 1,113 个文件 | 29,175 通过,5 失败,131 跳过 | 680.64s |
| `bun run test:quick`,首次测量尝试 | 931 个文件 | 22,160 通过,2 失败,100 跳过 | 125.39s |
| `bun run test:quick`,修复尝试 | 相同的 931 个文件 | 22,162 通过,0 失败,100 跳过 | 52.43s |

该 profile 的五个失败来自机器的 Git 身份包装器覆盖合成 fixture 作者。在隔离的测试环境中使用原生 Git 运行两个受影响的文件通过所有 59 个测试,耗时 75.32s;正常的 checkout 提交保留配置的身份。两个快速尝试都使用了该更正的环境。他们的两个遥测超时使用 Bun 的同步管道输入路径;该修复重用现有的文件支持的命令捕获帮助器,而不更改命令、断言或截止日期。种子保留观察到的成本,包括失败的尝试;它是一个调度提示,而不是通过的收据。

冷依赖项安装花费 0.477s,集成构建花费 3.84s,与热测试执行分开;CLI 安装未独立计时。较早的 63.37s profile 被取消以进行解码器修复,并带有额外的范围浏览器清理,并且没有完成信用。失败、取消和修复运行是成本,而不是从工作流中删除的时间。在该机器上达到了快速目标一分钟,但这些测量既未建立跨环境加速,也未建立完整发布、实时 model 或 Windows 验收。

### 较早的组件比较

测量的组件比较:

| 工作负载 | 之前 | 之后 | 保留的覆盖率 |
| --- | --- | ---: | --- |
| 九个合成终端文件,串行聚合 | 165.87s | 72.98s | 81 个测试,1,593 个断言,42 个 PTY 场景 |
| 发布保护和 watchdog 文件,串行聚合 | 56.57s | 9.63s | 245 个原始测试;三个额外的时钟控件 |
| 两个实时周期 AUQ 文件,同一台机器和运行时 | 325.09s | 136.40s | 两个测试,五个独立捕获,所有原始评分规则 |

合成终端比较的确切选择器:

```text
test/plan-count-fixture.test.ts
test/plan-count-design-ui-recovery.test.ts
test/plan-count-native-input.test.ts
test/plan-count-empty-review.test.ts
test/plan-count-owned-permission.test.ts
test/plan-count-file-permission.test.ts
test/plan-count-truncated-question.test.ts
test/plan-count-preview-footer.test.ts
test/eng-test-plan-edit-approval.test.ts
```

引用的帧选择器在 2026-09 审计中被折叠到 `test/plan-count-file-permission.test.ts` 中。

发布/看门狗对是 `test/autoplan-publication-guard.test.ts` 和 `test/cso-watchdog.test.ts`。实时对是 `test/skill-e2e-auq-consistency.test.ts` 和 `test/skill-e2e-auq-verbose-vs-carved-ab.test.ts`,使用默认的三个一致性样本加上两个 A/B 捕获。

这些是单独的比较;不要将它们的百分比相加或将它们称为完整的付费套件结果。终端聚合快 56%,两个发布/看门狗文件快 83%,实时 AUQ 对快 58%。看门狗断言计数仅下降,因为相同的编译被检查一次;所有行为和安全断言仍然存在。

匹配的完整本地免费套件比较使用相同的四 CPU Linux 机器、Bun 1.4.0、Node 24.18.0、构建工件、显示和隔离的 Git 配置。两个运行都没有设置 worker 覆盖:原始默认使用两个 worker,而更改的默认使用四个。

| 完整免费套件结果 | 原始 | 性能重构 |
| --- | ---: | ---: |
| 持续时间 | 887.82s | 413.78s |
| 通过的用例 | 25,504 | 25,547 |
| 失败的用例 | 0 | 0 |
| 跳过的用例 | 60 | 60 |
| 断言 | 199,132 | 200,351 |
| 测试文件 | 1,010 | 1,012 |

这是本地减少 53.4%。用例级别的协调保留了除故意重命名的 worker-default policy 测试之外的每个原始通过标识,添加了 43 个用例,并保留了完全相同的 60 个跳过标识。此比较先于其他 fixture 和部署工作流回归用例;最终验收必须涵盖这些。它不是单独的 CI 矩阵或完整付费人口普查的测量。

确切的历史 PR #2956 diff 选择 84 个 gate ID 加上 25 个 judge,是因为免费 runner 被视为未知的付费依赖项。在该边界修复之后重放该 diff 选择预期的 26 个快速 ID 加上相同的 25 个 judge。仅免费 runner diff 选择没有付费工作。这是路由准确性,而不是新的完整人口普查运行时改进。

## 证据有效性

集成主分支 2026 年 9 月 28 日的 Ubicloud 改进后,调度种子使用上游的 CI 环境时间来共享文件,并保留先前测量的 52 个分支专用条目。这些是两台机器的调度提示,而不是匹配的性能比较或验收结果。使用 `bun run test:ubicloud --record-durations` 在测量新的共同基线时刷新整个种子;不要通过添加这些测量来推断加速。

检查每个 SDK、打印模式和终端启动器实际使用的可执行文件。在预检期间缓存的 CLI 版本无法证明 PATH 内容更改后后续会话使用的版本。使用原生会话初始化版本、终端启动捕获或进程见证,并在构建 hermetic 环境后保持运行时控件有效。

捕获的事件回归证明验证器接受有效证据并拒绝无效证据。它们不会将旧的失败 model 运行转变为通过。保留每个配置的重试尝试,并区分实际注册和过滤或超出 tier 的占位符。仅当完整的请求、评分标准、参数和相关依赖项匹配时,保留的通过 judge 才可重用;原生行为验收遵循其单独的新鲜度合约。

## 剩余工作,而不是声称的节省

使用交付修订及其实际用例清单报告最终验收。完整的全新 gate/周期和跨平台性能比较不是由上述测量建立的。

最长的不可分割的实时工作流限制了额外 worker 的好处。历史付费持续时间重放表明仅更好的调度不能将完整通道减半。后续应在共享捕获之间分配或拆分长文件之前统一可执行用例所有权/计数:保持每个 oracle、场景、重试和独立试验要求显式。主机集成和安全边界用例不得被更便宜的同类替换;退役的 Autoplan 链评估需要一个适合普通 tier 的替代品。
