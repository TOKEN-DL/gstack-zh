# 测试审计 2026-09:证据

测试缩减分支的证据(通过 /autoplan 批准的计划:"A, 按现状批准";UC1 解析为删除)。基础:65bfb0c (v1.91.6.0)。无论计划要求 PR 主体表还是验证项目,它都在此处解析。

## 提交

| 提交 | 工作流 |
|---|---|
| G | 测试基础设施死代码 |
| F | 伪造产品的产品测试 → 真实边界测试 |
| A | 死评估代码的测试(可达性驱动) |
| B-cleanup | 付费 lane 清理(B1–B4、B6、B7) |
| C | 弃用永不绿色的发现计数集群,修剪其 helpers |
| D | 将每事件系列折叠到检测器所有者 |
| H | plan-count 历史 PTY 的启动就绪标记 |
| E | 派生触摸文件闭包不变量(行为更改) |
| B5 | Tier-lane 跳过和普查 judges(行为更改) |
| B8 | 11 个付费评估的默认捕获 model(行为更改) |
| Guard | Ratchet、共享解析器、CONTRIBUTING、TODOS、组合、本文档 |
| Release | 持续时间刷新、CHANGELOG、VERSION、文档扫描 |

## C0 分诊(在提交 G 之前记录)

2026-09-29 在提交 G 之前记录。来源:每周周期评估运行 34812905093 (09-14,每次运行的 sha)、35567915613 (09-21,a6b3a575)、36385945043 (09-28,65bfb0c-era)。预检:`gh` 已通过身份验证(git.capy.ai 代理,账户 garrytan);`gh run download` 401,但 REST `actions/artifacts/<id>/zip` 路由有效;所有三个运行的工件都保留(未过期)。每文件证据:09-28 = 上传的每个分片 PTY 工件(observation.json + 终端日志);09-14/09-21 = eval-slices 作业日志中的每个分片失败尾部(bun 输出 + observation dump + last-3KB 终端证据)。本地副本:审计工作区:c0/。

类别:product = skill 没有按发现询问;harness = PTY/分类器/超时/启动;budget = 截止日期命中时实时 model 仍在进行。应用的协议规则:harness 和 budget 都是非产品类别;当每个工件都是 harness 或 budget 时,文件被删除(没有工件显示产品),否则保留+排除。

| 文件 | 09-14 | 09-21 | 09-28 | 类别 | 操作 |
|---|---|---|---|---|---|
| skill-e2e-autoplan-chain | harness: 会话在 13 秒内退出,没有阶段标记(启动) | harness: 观察者仅在上下文压缩后看到阶段 3 标记;脚本引用 CEO、Design 和 DX 方法论文件和"Phase 3 complete" | budget: 在有序阶段 1、2、2.5 命中后超时(2 次尝试,160 分钟) | harness/budget | 删除 |
| skill-e2e-plan-ceo-finding-count | harness: 5 发现用例在 11 秒内退出;配对用例超时 | harness: model 询问了 8 个发现决策(SQL 查找、电子邮件错误、订单加载、测试、排序、TODO…),分类器将所有 preReview 标记为 `no_review_questions` | harness: 分类器抛出"Unsupported current CEO decision; cannot exclude it from the 4–7 count"(配对用例达到 plan_ready,review=2) | harness | 删除 |
| skill-e2e-plan-eng-finding-count | harness: 按发现问题 D3 渲染;指纹是旋转器垃圾,step0=4 review=0 | harness: 退役旧 oracle "mandatory legacy regression coverage absent" 与 reviewCount=9,然后分片墙超时 | harness: model 询问了 D1–D8/D9 按发现决策,所有标记为 preReview → review=0 的截止日期 | harness | 删除 |
| skill-e2e-plan-design-finding-count | harness: 在 1.7 秒内退出 | harness: BAND FAIL 高于上限(review=8 > 5 个发现的 7;按发现加上额外询问) | budget: 在 review=4 和 review=5 超时,仍在询问 | harness/budget | 删除 |
| skill-e2e-plan-devex-finding-count | harness: 在 2.1 秒内退出 | harness: 种子分类器错过了 `missing-quickstart`,尽管 reviewCount=9 和 outcome=plan_ready;重试命中分片墙超时 | 通过 (plan_ready, review=6) | harness | 删除 |

没有工件显示产品故障,因此不打开 C3 的问题;不需要 C0-kept TODO 条目。

## 安全映射(F)

| design/test/serve.test.ts mirror 'path traversal protection' (5) | design/test/serve.test.ts real serve() reload confinement | 在 design/src/serve.ts 中删除 startsWith(allowedDir) 保护 → 测试 1 失败 |
| browse/test/terminal-agent-internal-handler.test.ts 1–3 (internalHandler/route source greps; auth gate for grant+revoke) | browse/test/terminal-agent-integration.test.ts "/internal/grant and /internal/revoke bearer auth" (no/wrong/valid × grant/revoke + state effect) | 在没有 internalHandler 的情况下重写 revoke 路由(没有 bearer 检查) → "revoke: no token…" 和 "unauthenticated revoke…" 失败 |
| server-security-surface "/health carries no security field and server.ts does not import getStatus" (#2557) | extension-token "GET /health is liveness-only" (real /health body, default + headed/pinned-origin) | 将 `security: 'protected'` 注入到 /health body → 2 失败 |
| server-security-surface "security.ts no longer exports the unfed status surface" | 相同的 /health body 检查:getStatus 的唯一使用者是 /health.security;未使用的导出没有用户可见的效果 | (由上一行覆盖) |
| server-security-surface "the sidepanel shield markup is gone" | 相同的 /health body 检查:盾牌的唯一数据源是 /health.security,现在断言不存在 | (由上一行覆盖) |
| server-security-surface:60-66 "server.ts still consumes the sidecar on the inject-scan path" (ENG-OV9) | pty-inject-scan "/pty-inject-scan — L4 sidecar verdict drives the response" (real buildFetchHandler, sidecar client mocked in a child bun test) | 将 `if (sidecarAvail.available && verdict !== 'BLOCK')` 替换为 `if (false)` 在 server.ts 中 → 失败 |
| server-security-surface:68-76 "security.ts keeps the pure combiner + canary exports" | browse/test/security.test.ts imports and exercises THRESHOLDS, combineVerdict, generateCanary, injectCanary, checkCanaryInStructure, extractDomain | 取消导出 injectCanary → SyntaxError "Export named 'injectCanary' not found",security.test.ts 失败 |
| server-security-surface "/health stays liveness-only: no token in any mode" | extension-token "GET /health never carries a token (IRON RULE)" (3, existing) + liveness-only test | 注入 `token: authToken` → 5 失败 |
| server-auth "/health never serves a token — no headed-mode or chrome-extension carve-out" | extension-token IRON RULE 测试 (headed, pinned Origin, both) | 注入 `token: authToken` → 5 失败 |
| server-auth "/health does not expose currentUrl or currentMessage"; security-audit-r2 "/health endpoint security" (2) | extension-token "GET /health is liveness-only" | 注入 `currentUrl: 'x'` → 2 失败 |
| sidebar-tabs "/health no longer surfaces agentStatus or messageQueue length" | extension-token "GET /health is liveness-only" (also asserts terminalPort survives) | 注入 `agentStatus: 'idle'` → 2 失败 |
| security-audit-r2 "Task 1: validateOutputPath uses realpathSync" source greps (4) + behavioral (5) | browse/test/path-validation.test.ts "validateOutputPath — symlink resolution" + "validateOutputPath" allow/deny cases (now importing path-security directly) | 将 validateOutputPath 中的两个 realpathSync 解析都替换为未解析的路径 → symlink 用例失败 |
| security-audit-r2 "results.push is present in the loop block"; "viewport case uses rawW/rawH" (identifier greps, not security contracts) | kept siblings: "validateOutputPath appears before page.screenshot() in the loop", "viewport case clamps width and height" | n/a — 仅标识符名称 |
| test/skill-e2e-brain-privacy-gate.test.ts (paid, never green): privacy question fires once before any artifacts egress | test/gstack-skill-start.test.ts 'artifacts-sync consent is asked before any artifacts egress, and only in interactive sessions' | 在每日拉取上去除同步模式门 → 失败(拉取戳记与同意待定一起写入);去除仅交互条件 → 失败(spawned 会话获取门) |

## 混合文件和合并清单

### F(伪造产品的产品测试)
| 文件 | 块 | 决定 |
|---|---|---|
| design/test/serve.test.ts | 整个文件(针对内联镜像服务器的 16 个测试) | 删除;由针对真实 serve() 在临时端口上运行的 2 个测试替换到位 |
| test/gbrain-init-rollback.test.ts | 运行漂移的本地 bash 副本的 3 个测试 | 删除;回滚契约移至 test/gbrain-init-voyage-code-3.test.ts 执行模板提取的块 |
| test/gbrain-init-voyage-code-3.test.ts | 本地副本 voyage 用例(4) | 移动:现在执行每个模板 init 块(3 个站点) |
| test/grain-init-voyage-code-3.test.ts | "demonstrates the #1798 collision" | 删除(测试 zsh 本身) |
| test/gbrain-init-voyage-code-3.test.ts | 模板 grep 计数测试(3) | 保留(合并为一个"template alignment"测试) |
| browse/test/browser-manager-unit.test.ts | "signature accepts an optional exitCode argument", "server.ts callback forwards exitCode…" | 删除(重言式);真实所有者:server-factory "buildFetchHandler chains cfgBrowserManager.onDisconnect" |
| browse/test/memory-command.test.ts | "12. text mode renders modificationHistory with evicted-count when > 0" | 删除(比较两个本地文字);差距:evicted-count 后缀未在所有者处测试 |
| test/ios-qa-swiftui-tap-regression.test.ts (+2 fixtures, 98 KB) | 整个文件 | 删除 |
| test/memory-ingest-no-put_page.test.ts | 整个文件 | 删除;gstack-memory-ingest.test.ts fake gbrain exits 99 on put/put_page |
| browse/test/terminal-agent-internal-handler.test.ts | 测试 1–3 | 删除;由 terminal-agent-integration "/internal/grant and /internal/revoke bearer auth" (3×2 + state effect) 替换 |
| browse/test/terminal-agent-detach-reattach.test.ts | 测试 1、4、5、6 | 删除(terminal-agent-ring-buffer-runtime 的副本) |
| browse/test/terminal-agent-detach-reattach.test.ts | 测试 2、3、7–10 | 保留 |
| browse/test/server-security-surface.test.ts | 所有 6 | 删除;参见安全映射 |
| browse/test/server-auth.test.ts | "/health never serves a token — no headed-mode or chrome-extension carve-out", "/health does not expose currentUrl or currentMessage" | 移动 → extension-token "GET /health is liveness-only" + IRON RULE |
| browse/test/security-audit-r2.test.ts | "/health endpoint security" (2) | 移动 → extension-token "GET /health is liveness-only" |
| browse/test/security-audit-r2.test.ts | Task 1 块(4 source + 5 behavioral),"results.push is present…", "viewport case uses rawW/rawH…", AGENT_SRC | 删除;path-validation 拥有 validateOutputPath |
| browse/test/security-audit-r2.test.ts | escapeRegExp behavioral test | 保留,直接导入 path-security(元命令重新导出已删除) |
| browse/test/security-audit-r2.test.ts | state-load、inbox、responsive、CSS validator ordering greps | 保留(唯一保护) |
| browse/test/sidebar-tabs.test.ts | "/health no longer surfaces agentStatus or messageQueue length" | 移动 → extension-token liveness-only(also asserts terminalPort) |
| browse/test/sidebar-tabs.test.ts | "browse/src/sidebar-agent.ts is gone", "sidebar-agent test files are gone" | 删除 |
| browse/test/sidebar-ux.test.ts | "stop button style exists", "stop button uses error color", "experimental-banner no longer uses amber…", "tool description uses system font not mono" | 删除 + 死 CSS (.stop-btn, .experimental-banner, .agent-tool, .agent-reasoning; 67 行) |
| browse/test/sidebar-ux.test.ts | "switchTab has bringToFront option" (dup of :50), "shutdown kills the terminal-agent via identity-based kill" (dup of terminal-agent-pid-identity), "quick actions toolbar has cookies button" (dup of sidebar-tabs quick-actions) | 删除 |
| test/skill-validation.test.ts | "Generated SKILL.md freshness" (3) | 删除(C14);gen-skill-docs 占位符正则表达式扩大到 \w+ |
| test/gen-skill-docs.test.ts | "generated header is present in SKILL.md", "…in browse/SKILL.md" | 删除(C14);"every skill has a generated SKILL.md with auto-generated header" covers both |
| test/post-rename-doc-regen.test.ts | "top-level SKILL.md exists and is regenerated" | 删除(C15) |
| test/static-no-legacy-writes.test.ts | "office-hours/SKILL.md uses --log-session, not raw echo append" | 删除(C15);.tmpl sibling + freshness |
| make-pdf/test/coverage-gaps.test.ts | 所有 19 个用例 | 移动 → diagram-prepass.test.ts (18) 和 render.test.ts (screenCss) |

### A(死评估代码)
可达性工具:审计工具 reach.ts(ts-morph;根 = 每个导入 test/helpers + bin/gstack-model-benchmark + outside-voice shim 的非助手文件;边 = 标识符 → 顶层助手声明;BFS 到固定点;--prune 移除未到达的声明和未使用的导入,重新运行直到 0)。65bfb0c 的基线:12 个死声明 = 10 个 A4 名称 + `execGit` (auq-sdk-capture) + `invokeAndObserve` (claude-pty-runner)。A 的测试删除后:33 个死(eng-seeded-coverage oracle 闭包 2,626 行,autoplan-artifact-permission approvers 474,matchesAutoplanDigestRows 86,上述 12 个);通过 2 → 0。

| 文件 | 块 | 决定 |
|---|---|---|
| 25 A1 纯文件 + eng-native-seed-contract.test.ts | all | 删除(所有 61 个 native-seed-contract 测试调用 evaluateEngSeedCoverage;其 3 个具有实时 pty-runner 断言的块重放 plan-eng-finding-count 回调;hasNativePlanTerminal / isQuestionlessNativePlanExit / classifyPlanCountFrame 保留所有者 plan-count-pending-exit, plan-count-empty-review, plan-count-completion) |
| eng-count-ad-v2 | "first attempt … D9 handoff", "prior successful plan Write…", "closed handoff…", "new task references…", "conditional closure…" | 删除(isEngCompletionHandoff) |
| eng-count-ad-v2 | "new first-finding and handoff paths…" | 保留 first-finding 一半(engFirstReviewAUQ);handoff 一半已删除 |
| eng-count-ad-v2 | census() | 保留,死 handoff predicate 参数已删除(retry census 未更改:administrative 0) |
| eng-count-ad-v2 | 6 个实时 + 触摸文件测试 | 保留(触摸文件测试失去 eng-completion-handoff 路径行) |
| eng-resolution-block-position | 测试 1–3 | 删除(handoff / seed oracle) |
| eng-resolution-block-position | "saved native Header and Options…" (createEngBatchingIssueCounter) | 保留 |
| eng-seeded-completion-ai | "complete native navigation preserves conflicting current states…" | 删除(handoff) |
| eng-task-pause-navigation-f359 | 所有 check()/handoff 测试 | 删除 |
| eng-task-pause-navigation-f359 | "handoff alone never supplies a native terminal…" | 保留(hasNativePlanTerminal);admin set 现在为已完成的调用签名 |
| eng-next-handoff-ah | 18 个 handoff 测试 + parser-ACK 重放 | 删除 |
| eng-next-handoff-ah | "exact final exit/report replay…", "actual pending ExitPlanMode…" | 保留(hasNativePlanTerminal, isCurrentPlanApprovalScreen) |
| eng-published-navigation | ~150 个 handoff 检查 | 删除 |
| eng-published-navigation | retry, real-completed, D19, investigation, cf74 terminal replays | 保留(hasNativePlanTerminal);内部已删除的死 handoff/phase 断言 |
| eng-seeded-coverage.test | 23 个 oracle 块 + 6 个基于 evaluateEngSeedCoverage/isEngSeedDecisionAUQ 构建的描述 | 删除 |
| eng-seeded-coverage.test | "Eng semantic native evidence boundary", 触摸文件测试, "batching caller counts…" | 保留 |
| autoplan-edit-digests-al / clipped-suffix-aq / pending-artifact | approver cases (8/8/8) | 删除 |
| same three | recorder/launcher cases | 保留 |
| 11 A2 replay files | all | 删除 |
| autoplan-permission-viewport | 27 个测试(autoplan-phase-order + pty-current-screen) | 删除;捕获的设置覆盖卡断言移至 claude-pty-runner.unit "isPermissionDialogVisible" |
| autoplan-phase-observation | 44 个测试(全部通过 phase-order helpers) | 删除 |
| eng-finding-fixture.test | seeder + legacy-auth fixture tests (5) | 删除 |
| eng-finding-fixture.test | 2 个付费 eng-finding-count 文件的 prompt-builder pin | 保留到 C(计划说"四个 prompt-builder 测试";只有 2 个) |
| ceo-paired-payment-fixture, design-ui-scope, plan-skill-completion, pty-current-screen, required-reads, transcript-section-logger tests | all | 删除 |
| plan-count-fixture | 3 design-ui-captured cases + captured-question fake plumbing | 删除 |
| autoplan-phase-handoff | readPlanSkillCompletion assertion in "captured parent text…" | 删除行;测试已保留 |
| plan-seed-submission | PtyCurrentScreen decoder | 交换到生产 createPtyScreen (58/58 通过) |
| touchfiles.test | plan-skill-completion path in "native completion changes select the Design UI gate" | 从 each-list 中删除 |

### B-cleanup(B1–B4、B6、B7)
| 文件 | 块 | 决定 |
|---|---|---|
| skill-llm-eval-spec, skill-e2e-spec-execute, gemini-e2e (+ gemini-session-runner + test), skill-e2e-ship-idempotency, 2 overlay opus-4-7 *-sonnet wrappers (+ fixture entries), skill-e2e-conductor-prose, codex-e2e-plan-format, skill-e2e-brain-privacy-gate | all | 删除(B1/B7) |
| conductor-prose-observation-ao.test.ts + fixture | all(evaluates the deleted paid caller's source) | 删除 with its paid file |
| plan-tune-cathedral-fixture.test.ts | all(evaluates the cathedral file's source under injected fakes) | 删除 — 大教堂场景现在直接在免费套件中运行(B3) |
| skill-llm-eval.test.ts | "regression vs baseline" | 删除(B2) |
| skill-llm-eval.test.ts | "command reference table", "snapshot flags reference", "browse/SKILL.md reference" | 折叠 → 一个联合 judge "browse/SKILL.md reference"(B2) |
| skill-llm-eval.test.ts | "baseline score pinning" | 折叠到联合 judge(固定 eval-baselines.json browse_skill) |
| skill-e2e-opus-47.test.ts | 3 negative routing controls | 移动 → skill-routing-e2e "journey-negatives" (same ≤1-of-3 bound);positives 已在 skill-routing-e2e 中 |
| skill-e2e-ios.test.ts | "ios-qa E2E (with device)" HAS_DEVICE stub | 删除(B3) |
| gstack-skill-start.test.ts | 新 "artifacts-sync consent is asked before any artifacts egress…" | 添加(B7:existing pins did not assert ordering) |
| paid census literals (paid-retry-supervision, paid-overlay-scheduling, overlay-lifecycle, overlay-measurement, paid-shards, touchfiles, periodic-fixture-selection, codex-eval-selection, paid-pr-profile) | counts / key lists | 更新为已删除的文件和键(除命名已删除键的之外,没有删除任何断言) |

### C(弃用发现计数集群,C2 helper 修剪)

规则:当每个断言主题位于 C 后的实时闭包之外(已修剪的 helpers,或通过注册适配器加载的已删除付费文件)时,免费测试块被删除;仅使用死代码作为实时主题的*输入构建器*的块被保留,并且构建器被替换或恢复(下面的行)。LIVE 块被保留。Touchfile 自断言仅失去删除的键(E 删除它们)。

| 文件 | 块 | 决定 |
|---|---|---|
| test/skill-e2e-autoplan-chain.test.ts | 整个文件 | 删除(C1;C0 类 harness/budget,see triage) |
| test/skill-e2e-plan-ceo-finding-count.test.ts | 整个文件 | 删除(C1;C0 类 harness/budget,see triage) |
| test/skill-e2e-plan-design-finding-count.test.ts | 整个文件 | 删除(C1;C0 类 harness/budget,see triage) |
| test/skill-e2e-plan-devex-finding-count.test.ts | 整个文件 | 删除(C1;C0 类 harness/budget,see triage) |
| test/skill-e2e-plan-eng-finding-count.test.ts | 整个文件 | 删除(C1;C0 类 harness/budget,see triage) |
| 84 free test files (list in commit) | 整个文件 | 删除:每个块仅行使已修剪 helpers 或已删除付费文件 |
| test/autoplan-eval-budget.test.ts | 整个文件(AUTOPLAN_CHAIN_BUDGET, dedicated slice) | 删除;timer-safe/explicit-override checks 移至 eng-finding-retry-budget 'ordinary tiers and registered allocations remain unchanged' |
| test/plan-review-native-default.test.ts | 3 个测试(omitted multiSelect default) | 移动 → plan-review-decisions 'an omitted native multiSelect receives the false default only in evaluator input' (removal-checked) |
| test/autoplan-chain-fixture.test.ts | 'native sequencing config reaches the real CLI reader…' | 移动 → plan-count-fixture.test.ts;其他 3 个测试删除(链源 pin) |
| test/eng-finding-fixture.test.ts, test/design-finding-fixture.test.ts | 整个文件 | 删除(读取/导入已删除的付费文件) |
| test/ceo-current-decision-record.test.ts (PROD-TOUCH) | all 28 | 删除:将 plan-ceo-review 模板读取为已弃用的 ceo-payment-findings 计数器的输入 |
| test/devex-finding-fixture.test.ts | DX registration (8) + materialized devex-existing-sdk checks (5) | 删除(fixture 仅由已删除的 DX 计数评估消耗);保留 'every host exposes the DX per-call rule…' |
| test/ceo-finding-fixture.test.ts | 'native count registration: %s' (11) | 删除(imports the deleted paid file);fixture tests 保留 |
| test/eng-semantic-terminal.test.ts | evaluateEngTerminalReview/buildEngSeedDecisionInput blocks (6), registration loops (7) | 删除;'real native Exit…' 和 'a late substantive answer…' 保留,带有直接的 id 回调代替已死的评估器 |
| test/eng-seeded-coverage.test.ts | 'Eng semantic native evidence boundary' describe, 2 mixed, touchfile test | 删除(buildEngSeedDecisionInput 死;验证器由 plan-review-decisions 拥有) |
| test/plan-count-fixture.test.ts (PROD-TOUCH) | real PTY children worker | 保留;死 design/devex predicates 由 inline caller policies 替换;死-classifier 断言已删除 |
| test/plan-count-native-input.test.ts | design outside-voices cases | 保留;pickDesignCountOutsideVoices 由 inline caller policy 替换;autoplan routing test delete |
| test/plan-pending-question-pty.test.ts | hook PTY test | 保留;autoplanSetupDecision 导航由固定的原生密钥序列替换 |
| test/helpers/claude-pty-runner.unit.test.ts | findModeOption (7), design/devex Step0 + first-review (14) | 删除;2 prompt-parser tests 保留,带有死边界断言的修剪 |
| test/autoplan-method-read-audit.test.ts, autoplan-phase-handoff, autoplan-publication-guard, plan-count-session-cwd, autoplan-preconfigured-onboarding-ar (PROD-TOUCH) | all but chain caller pins | 保留;helpers autoplan-method-read-audit.ts / autoplan-preconfigured-fixture.ts 已恢复(它们适配生产阶段发布 hook / skill-start) |
| test/autoplan-artifact-recorder, autoplan-edit-digests-al, eng-test-plan-edit-approval | recorder tests | 保留;readPendingAutoplanArtifact 已恢复(记录器由 claude-pty-runner 导入) |
| test/carve-guards (helper) | autoplan externalTest | 行为 'none'(链是其唯一的 section-read 证明;TODOS 条目) |

[desktop write output truncated: original_bytes=55654; if the omitted remainder is needed, continue with read using offset=182 and the same path.]
