# TODOS

> **关于本翻译的说明:** 本文件源自 gstack 项目的 TODOS.md(约 5,381 行,335KB),按 P0-P4 优先级组织 gstack 的待办事项、回滚与已完成工作。翻译保留以下原样:issue 编号(`#1694` 等)、PR 编号、commit SHA、文件路径、命令、env var 名、用户名(以 `Contributed by @username` 与 `@username` 格式)、版本号、emoji 标记。翻译专注于散文叙述与技术说明的中文化。

---

## NEXT PRIORITY

### P3: `/codex --effort` follow-ups (filed 2026-10-11, from the #3146 autoplan review)

- **Show the effort that ran.** Step 0.5 打印 `CODEX_EFFORT: <level> (source: /codex --effort)` 一次,但结果 footer 不重复 effort,所以它会滚走。在 review、challenge 与 consult 结果 footer 中加一个 `Effort:` 行。被 `codex/SKILL.md` 上的 skeleton 字节阻塞(上限 60,200)。**Effort:** S。**Priority:** P3。
- **Timeouts at high effort.** `xhigh`、`max` 与 `ultra` 能超过固定的 330 秒/540 秒 wrapper,且超时副本将原因归咎于 prompt 大小或 API 速度。在超时消息中命名 effort 并建议更低的 `--effort`;考虑 effort-缩放预算。**Effort:** S。**Priority:** P3。
- **Per-run effort for gate voices (#2934).** `/review`、`/ship` 与 plan-review outside voices 接受 `GSTACK_CODEX_EFFORT` 与 `codex_effort` 但没有 per-request flag;Claude Code runner 根本不接受 effort。需要它自己的优先级与同意审查。**Effort:** M。**Priority:** P3。
- **`minimal` with web search.** 每个 `/codex` dispatch 传 `web_search="cached"`,且 Codex 据报以 `reasoning.effort: minimal` 拒绝它(openai/codex#3559、#5002)。此处未验证(release 机器上无 Codex CLI);相同路径已通过 `codex_effort minimal` 发货。在钉住的 CLI 上验证,如果失败就为 `minimal` 放弃 web search。**Effort:** S。**Priority:** P3。

### P2/P3: open bugs by area after the Wave 9 triage (filed 2026-10-11)

158 个较旧的开放 issues 重新对 main 5cb5e1c(v1.91.83.0)进行检查。30 个仍有效的 bug 中,Wave 9 lanes 接受 #915、#1080、#1081、#1150、#1315 与 #1675,并停放 #2118 与 #2623(下一节)。其余保持开放:issue、根因文件、大小。已在本文其他位置跟踪的 bug 指向那里而不是重复。

- **Plan reviews and planning artifacts**
  - #280: plan-eng-review 从不读 CEO plan。它的 Step 0 discovery 没有 `ceo-plans/` 查找(只有 `design-html/SKILL.md.tmpl` 与 `bin/gstack-context-recovery` 读它)。添加一个 `gstack-ceo-plan-find` 作为 `bin/gstack-design-doc-find` 的兄弟;保留 per-branch review log。**Effort:** S。**Priority:** P3。
  - #542: 在 host plan mode 中编写的 office-hours 设计文档落在 `~/.claude/plans/` 中,`bin/gstack-design-doc-find` 从不读它,所以两个副本漂移。选一个权威并在陈旧副本上警告。**Effort:** M。**Priority:** P2。
  - #1048: `plan-eng-review/SKILL.md.tmpl` 与 `plan-ceo-review/SKILL.md.tmpl` 中的 "smallest clear diff" 没有 schema-design carve-out,所以它使评审偏向 JSON blobs。加一个权衡 bullet,带付费的前后用例。**Effort:** S。**Priority:** P3。
  - #1791: 每个 plan-eng-review run 写一个新鲜的 `tasks-eng-review-<ts>.jsonl`,名称中没有分支,没有对账(`scripts/resolvers/tasks-section.ts`)。把分支放在名称中并取代较早的集合。**Effort:** M。**Priority:** P3。
  - #1957: /autoplan Phase 0 不为 UI scope 打印 matched-terms 行,并把 UI 与 DX 词保留在两个地方(`bin/gstack-autoplan-snapshot.ts`、`lib/autoplan-prompts.ts`)。一个共享词模块,以及在打印行中的一个 `borderline` flag。**Effort:** M。**Priority:** P3。
- **Review**
  - #356: `/review` 没有 repo-convention 检查,所以一个加到 bash 与 Make repo 的 `.bat` 通过。将它加到 `review/specialists/maintainability.md`。**Effort:** S。**Priority:** P3。
- **Office hours and design**
  - #340: 批准的 sketch 保留在一个可清除的 `mktemp` 目录中,且没有下游设计 skill 读它(`scripts/resolvers/design.ts`)。把它复制到设计文档旁边,并让 plan-design-review 与 design-review 读它。**Effort:** S。**Priority:** P3。
- **Questions**
  - #1319: AskUserQuestion 格式(`scripts/resolvers/preamble/generate-ask-user-format.ts`)没有布局或长度规则,所以长 brief 呈现为一堵文本墙。加 block 分离与每块上限,带一个包含 CJK 用例的免费测试。**Effort:** M。**Priority:** P2。
- **Setup and upgrade**
  - #954: vendored 安装从 `setup` 的 `_link_or_copy` 获取绝对符号链接。当源位于同一 skills 目录中时使用相对链接。**Effort:** S。**Priority:** P3。
  - #1022: 跟踪安装目录的父仓库在每次升级后静默还原 VERSION。用 `git ls-files --error-unmatch` 在 `setup` 与 `gstack-doctor` 中检测它,并打印 `git rm --cached` 命令。**Effort:** S。**Priority:** P3。
  - #2120: `bin/gstack-patch-names` 与 setup 的 per-skill 链接循环每个 skill 分叉约 4 次,这会在 Windows ARM64 上的 x64 模拟 Git Bash 中死锁。使读路径无分叉。在没有 ARM64 Windows host 的情况下未验证。**Effort:** M。**Priority:** P2。
  - #2378: update-check 比较 VERSION,而 `/gstack-upgrade` 安装 HEAD。在 "v1.67 fix-wave deferrals"("#2378 release-tag upgrade semantics")下跟踪。**Priority:** P3。
- **Build**
  - #407: `scripts/build.sh` 中的 `bun build --compile` 直接写到 `dist/`,且 Bun 的重命名在 virtiofs devcontainer 挂载上失败。编译到本地临时目录然后移动。在当前 Bun 上未确认重制。**Effort:** S。**Priority:** P2。
- **Windows binaries**
  - #2124、#2595: Smart App Control 阻止本地编译的 browse.exe。在 "Windows follow-ups from the October fix wave" 下跟踪;被 Garry 的签名账户阻塞。**Priority:** P2。
- **Safety hooks**
  - #1060: 在 Oct 8 safety hooks follow-ups 下的 "Quote-aware masking in `check-careful.sh`" 下跟踪。**Priority:** P3。
- **Sessions**
  - #1651 part B: 在上面的 office-hours and timeline follow-ups 下跟踪。**Priority:** P3。
- **/cso**
  - #2993: 在上面的 /cso capacity follow-ups 下跟踪。**Priority:** P2。
- **Dependency security ledgers**(不是代码缺陷;保持开放直到触发)
  - #2753: `.osv-scanner.toml` 忽略 image-size(经由 html-to-docx)直到 2026-11-30。一旦有固定发布就 bump,或在过期时重新证明正当。**Effort:** S。**Priority:** P3。
  - #2754: `.osv-scanner.toml` 忽略钉住的 claude-agent-sdk 下的 `@anthropic-ai/sdk` 0.81.0 直到 2026-11-30。在下次 harness bump 时删除它。**Effort:** S。**Priority:** P3。
- **Tests and CI**
  - #2756: periodic lane。在 "Periodic-lane stabilization (#2756)" 下跟踪。**Priority:** P2。
  - #2967: `test/redact-prepush-scan-range.test.ts`(以及它的 rebase 双胞胎)用继承的全局 config 跑 fixture push,所以开发者的 `core.hooksPath` pre-push 在 fixture 内跑。在 helper 中设 `GIT_CONFIG_GLOBAL=/dev/null` 与 `GIT_CONFIG_NOSYSTEM=1`。**Effort:** S。**Priority:** P3。

### P3: parked in Wave 9 (filed 2026-10-11)

每个 item 都在 Wave 9 计划中决定并带其原因停放。当被接走时,致谢报告者或贡献者。

- **Fold `triggers:` into skill descriptions (#2118)** — `scripts/gen-skill-docs.ts` 仅折叠 `voice-triggers:`,且修剪后的描述保留一个句子,所以自然语言触发器永远不到达列表。停放因为它与目录-修剪设计冲突且会重新生成每个 skill。接走时的机制:一个 `when_to_use:` frontmatter 键用于 Claude host,从 `triggers:` 构建,去重,并保持在目录字节预算内(Claude Code 的 1,536 字符组合列表限制);允许 name 与 description 的 host 得到一个有界的 "Use when…" 句子或什么都没有。**Effort:** M。**Priority:** P2。
- **Plan-tune signals stuck at 0.5 (#2623)** — `hosts/claude/hooks/question-log-hook.ts` 记录 UI 标签,而 `scripts/psychographic-signals.ts` 查找稳定的选项键,所以没有行匹配。两行的键映射仅是 capture 管道且不修复饱和。验收标准:标签在日志时映射到稳定的选项键;`profile.inferred` 记录每个维度的贡献事件;per-event 与 per-session 累加有界,所以维度不能在 1.0 处被钉住;`{skill}-{slug}` id 获得暗示回退。需要关于键源的设计调用。**Effort:** M。**Priority:** P2。
- **Stateless, instruction-only host mode (#3061, @m-check1B)** — 托管多租户运行时无安装、状态根或辅助二进制,但 skill 主体仍读取与写 `$GSTACK_STATE_ROOT`、调用 `bin/gstack-*` 与记录遥测。推迟到下面的新-主题。需求保留在案:租户拥有的状态(一个 `capabilities.state: 'none'` host 渲染宿主提供的 hooks 而不是 shell 步骤),按 kind 命名的产物 hooks(`ARTIFACT_SAVE`/`ARTIFACT_FIND`、`WEB_RESEARCH`、`SECRET_SCAN`、`USER_PREF`)以 gbrain resolvers 已有的方式映射,一个生成器检查让包含 `~/.gstack`、`$GSTACK_`、`bin/gstack-*` 或 `--event-type` 的无状态渲染失败,且 `claude` host 的输出字节相同。贡献者将 office-hours overlay 加上生成器检查作为第一个 PR。**Effort:** L。**Priority:** P3。
- **Legacy unprefixed Cursor skill folders (#3084)** — 1.31 Cursor 安装创建 `~/.cursor/skills/<skill>/SKILL.md` 链接;1.91 链接 `gstack-*` 文件夹且 `_prune_stale_generated` in `setup` 仅看 `gstack-*` 名称,所以 Cursor 能加载两个同名的 skill。`setup` 已移除旧的未加前缀的 Claude 条目;Cursor 需要相同的拥有链接清理(仅移除指向 gstack checkout 的链接)。从 #3084 的数据丢失修复中分离,Wave 9 Lane A0 发货。**Effort:** S。**Priority:** P3。
- **`/retro global` says what a "session" is (#1315, third problem)** — `bin/gstack-global-discover.ts` 计数转录文件与 rollouts,所以 cron- 或 agent-驱动的 repo 显示数百个 "session" 而无交互工作。注释 retro 输出,sessions 是工具调用或文件,而不是不同的人类工作会话。问题 1 与 2 已修复(v1.91.66.0 与 Wave 9 Lane F)。**Effort:** S。**Priority:** P3。
- **`gbrain put` stdin pipe in `lib/code-intelligence/gbrain-adapter.ts` (#1675)** — `put` 在 stdin 上发送文档体(`#runInput(["put", slug], body)`),与 #1675 在 Windows 上 setup-gbrain 的 smoke test 中击中的相同形状,Wave 9 Lane G 改为临时文件。首先在 Windows 上的钉住版本检查 gbrain 的 stdin 处理;如果仍然坏,通过私有临时文件或 MCP `put_page` 路径传递体。**Effort:** S。**Priority:** P3。

### P3: feature themes from the Wave 9 triage (filed 2026-10-11)

73 个开放的 feature requests,按主题分组并计算需求(主题中的 issues)。每个 issue 在关闭时链接回其主题;三个最大的跨切主题(new hosts、provider pluralism、install footprint)作为跟踪 issue 保持开放。作为一个整体接走一个主题并致谢每个链接的报告者。

- **New hosts (8 issues, plus #3061 above and 9 PRs parked on Oct 8)** — Kimi CLI (#1072)、Gemini CLI (#1247)、Pi Agent (#1288)、Claude Desktop Cowork (#1345)、Letta Code (#2117)、非 Claude hosts 的 team mode (#890)、`/retro global` 发现 Antigravity (#1977) 与 Kimi Code / OMP 会话 (#2917)。等待 host-config 工作;见 "Parked contributor ideas (Oct 8 triage)" 获取停放的 host PRs。**Priority:** P3。
- **Provider pluralism (7 issues, plus #3122)** — Gemini CLI 作为 outside-voice backend (#619)、`design` 中的非 OpenAI image providers (#990)、`design` 二进制自己的 generate、evolve 与 prototype 调用中的 Codex-native `image_gen` (#2174;design-shotgun 自身自 #2175 以来在 Codex 上使用 `$imagegen`)、Codex call routing 中的 GPT-6 Sol/Luna tiers (#2943)、`/grok` review port (#2947)、Codex review token usage (#2948)、Codex refresh-token recovery docs (#1542),以及在 /review、/ship 与 /retro 中替代硬编码 Greptile 的可配置 PR review bot (#3122,由决定 D4 折叠在这里)。一个 provider seam 服务所有这些。**Priority:** P3。
- **Install footprint and layout (7 issues)** — skills.sh distribution (#113)、`~/.gstack` 作为全局安装路径 (#753)、保留状态在项目中的 repo-local install (#1235)、每个捆绑 Bun 的五个编译二进制 (#1245)、Windows junctions 而不是 copies (#1877)、1.1 GB 的 `~/.claude/skills` 占用空间 (#2199),以及一个仅 skill 模式 (#2528)。决定 D6:setup 在 Windows 上保留暂存的副本,因为它今天工作且链接会改变每次清理删除的内容;#1943 (@JonasFocus,junctions) 与 #2129 (@TTmo123,带特权检查的 symlinks) 以信用关闭且它们的设计作为起点留在这里。也见 "install footprint follow-ups from the October fix wave"。**Priority:** P3。
- **New skills (12 issues)** — plan reviews 的 GUI kanban (#247)、journey maps (#729)、UI-doc review (#851)、assumption verification `/avl` (#1058)、`/a11y-check` (#1104)、`/env-check` (#1106)、原生 plan writing (#1170)、`/plan-experiment` 与 `/review-experiment` (#1195)、`/plan-status` (#1343)、语言特定评审者 (#1753)、`/benchmark-mobile` (#1754)、session start/end skills (#2088)。**Priority:** P3。
- **Ship, review and safety (9 issues)** — `/review` 影子原生命令 (#422)、静默-部署-失败清单 section (#2089)、Conductor TODO proposals 与 /ship 对未解析的 block (#2090)、light-ship mode (#2188)、归档 issue 前审计 (#2642)、landing-report 中的 maintainer-gated CI 与失败 (#2766)、Flutter pubspec 带构建元数据的 versions (#2833)、文档 skill 的确定性事实-一致性检查 (#2887)、双引擎 /careful guard (#2888)。**Priority:** P3。
- **Planning reviews (8 issues)** — marketplace-aware skill recommendations (#419)、YC recommendation 中的 founder-signal reasoning (#513)、spec reviews 中 path、math 与 flag 的有根据验证 (#973)、substrate-neutral existing-capability check (#1685)、`/autoplan --tier` (#1955)、autoplan workspace safeguards (#1959)、从双向缺陷中分离单向架构分叉 (#2490)、"需求真实吗?" 的证据分类账 (#2350)。**Priority:** P3。
- **State root and project identity (6 issues)** — repo-local portable context artifacts (#1161)、design-doc archive 与 prune (#1536)、跨多个 repo 的 umbrella directories (#1923)、回收 per-repo `.gstack` 状态 (#2627)、嵌套 repos 声明它们自己为 projects (#2649)、decision events 上的引用与处置 (#2678)。**Priority:** P3。
- **Browser and mobile QA (6 issues)** — 经由 `BROWSER_WS_ENDPOINT` 的远程浏览器 (#974)、per-session tab isolation (#1961)、opt-in User-Agent 后缀与默认 headers (#2459)、在 Linux 与 Windows 上跳过 Aside 探针 (#2864)、caught-up reads 上的 activity-ring copies (#2973)、touch input 与 mobile Safari (#3058)。**Priority:** P3。
- **Prompt cost and skill shape (4 issues)** — response-header template (#1376)、在会话内缓存 preamble 输出 (#1572)、Fable 5 prompting-guide audit (#1994)、`gstack-context-bill` 中的 per-skill benefit (#2889)。**Priority:** P3。
- **Customization and localization (3 issues)** — locale-aware skills (#481)、关于新定制 dimension 的教程 (#489)、`SKILL.local.md` overlay (#1759)。**Priority:** P3。
- **Retro (2 issues)** — 团队作用域的 multi-repo retro (#402)、narrative 中的 ADR 与 issue awareness (#2284)。**Priority:** P3。
- **CI supply chain (1 issue)** — GHCR CI 镜像的产物证明 (#2003)。**Priority:** P3。

### P2/P3: office-hours and timeline follow-ups (filed 2026-10-08, v1.91.54.0)

由 office-hours tier 修复 (#2801, #879, #1049, #1958, #1723, #1651 part A) 保持开放。

- **Session markers that reflect "currently active" (#1651 part B)** — skill start 仅在 skill 启动时触碰 `sessions/<pid>`,所以一个长会话两小时没有新 skill 会从 `SESSIONS: N` 中掉出。修复形状:从 hook 按 tool call 刷新 marker。**Effort:** M。**Priority:** P3。
- **Live eval for the first-session closing** — 免费测试按顺序跑渲染的 profile 读与会话日志,但没有付费用例检查模型是否在 Phase 6 中遵循所携带的 `SESSION_TIER`(首次 run 得到介绍,第二次 run 询问较早的分配)。也没有 office-hours 质量裁判。**Effort:** M。**Priority:** P2。
- **`gstack-timeline-read --since`/`--branch` without a value** — 两者仍在 `set -u` 下读一个未绑定的 `$2` 并以 exit 1 崩溃,#1723 为 `--limit` 修复的类。**Effort:** S。**Priority:** P3。

### P3: Oct 8 fix wave, safety hooks follow-ups (filed 2026-10-08)

由 PowerShell/NotebookEdit hook 修复 (v1.91.47.0, #3067) 保持开放,每个都带其原因。

- **Forced overwrite patterns for `/careful`** — `Set-Content -Force` / `Out-File -Force` 覆盖现有文件(以及 Bash `>` 覆盖孪生)仍通过。推迟:Bash 一侧也没有覆盖模式,且添加一个是两个 shell 的新家族,而不是 PowerShell 奇偶校验。**Effort:** S。**Priority:** P3。
- **Quote-aware masking in `check-careful.sh` (#1060)** — `git commit -m "rm -rf build"` 仍会询问。推迟:在安全 hook 中屏蔽引用文本能打开绕过(PR #1110 的 `echo hi; rm -rf ~` 展示了如何);它需要带 #1110 字符串作为对照的自己的设计。**Effort:** M。**Priority:** P3。
- **Hooks on Windows without Git Bash** — frontmatter hook 命令跑 `bash -c ...`,且 Claude Code 在 Git Bash 缺席时通过 PowerShell 跑 hook 命令,所以守卫无法在那里启动。gstack 今天要求 Git Bash;一个原生 PowerShell hook 启动器会解除该限制。**Effort:** M。**Priority:** P3。

### P2/P3: iOS QA fix-wave follow-ups (filed 2026-10-07, v1.91.38.0)

由 iPad 与 route-drop release 保持开放,每个都带其原因。

- **Device verification of v1.91.38.0** — iPad bootstrap、真实 Xcode 26 tunnel drop 上的 route-drop 恢复、多设备错误与 `NOT READY` 路径仅针对模拟设备证明;变更的 `StateServer` 通过 `swiftc -parse` 但无 Apple-SDK 构建。在带 iPhone 与 iPad 的 Mac 上跑 `test/skill-e2e-ios-device.test.ts` 与 Swift 构建 lane。**Effort:** S。**Priority:** P2。
- **`GSTACK_IOS_LAUNCH_ENV`** (split from #1796, @Bmathews721) — 通过 `devicectl process launch` 将启动环境传递给 app。主分支上无等价;作为带信用的小 PR 落地,然后关闭 #1796。**Effort:** S。**Priority:** P2。
- **A restarted daemon still relaunches the app once** — 新 daemon 无 session bearer 且一次性 boot token 消失。修复它需要带旧会话证明的 device-side re-mint 或状态根下的持久 bearer,且两者都改变 token 的安全模型。需要设计。**Effort:** M。**Priority:** P3。
- **iPad Stage Manager and multiple scenes** — overlay 与 window 选择(`frontmostWindow`、`searchRoots`)在 iPad 多窗口布局上未验证。**Effort:** S (verify) / M (fix)。**Priority:** P3。
- **Stale "Session warm-start" phase in `/ios-qa`** — Phase 0 描述一个无 daemon 代码写的 `ios-qa-session.json` 缓存。用状态根链实现它或删除 phase。**Effort:** S。**Priority:** P3。
- **Sanctioned input-routing hook for SwiftUI gestures** (#1975 finding 2) — 合成 touches 在 iOS 26 上漏掉 `DragGesture`;一个文档化的 `#if DEBUG` hook 让 gesture-driven apps 路由 `/tap` 与 `/swipe` 到它们自己的 handler。**Effort:** M。**Priority:** P3。

### P1/P2: /cso capacity follow-ups (filed 2026-10-08)

由 v1.91.42.0 保持开放,该版提升了快照文件计数上限。

- **Tracked symlinks refuse the whole /cso run** — `capture()` 在任何被跟踪的 symlink 上失败关闭("Symlink or special source file")。为 v1.91.42.0 测量的 8 个中型 OSS repo 中的 5 个停在那里(django、rails、terraform、grafana、vscode;每个 1-10 个 symlinks,主要是 test fixtures)。将它们记录为 manifest 中的不可读排除(如依赖树)是一个安全设计调用,留给 Garry。**Effort:** M。**Priority:** P1。
- **64 MiB aggregate source cap (#2993)** — 超过 1 MiB 的文件被从审计中扣留但仍被读取与计数。Stream-hash 它们而不计 cap,将它们报告为不可读覆盖,并将 admission 估计加到 `doctor`。grafana (211 MiB) 与 vscode (540 MiB) 一旦 symlinks 解决就撞上它。一个 `GSTACK_CSO_SNAPSHOT_MAX_BYTES` 旋钮被一起推迟。**Effort:** L。**Priority:** P2。
- **`inspect` prints the whole public manifest** — 5,000 个文件约 2.8 MB 的 stdout,超过代理的工具输出保持的内容。skill 从中读路径,所以修剪它是一个带 evals 的 skill-契约变更。**Effort:** M。**Priority:** P2。
- **Repair bundles carry every transformation** — `lib/cso/verification.ts` 将完整变换列表绑定进一个 1 MiB 不可变 bundle。仅运行时验证写 bundle,且无合格运行时 catalog 存在。**Effort:** M。**Priority:** P3。

### P3: iOS QA fix-wave follow-ups (filed 2026-10-07, v1.91.38.0)

(重复条目;见上面 iOS QA 部分)

### P2: harden or re-tier skill-e2e-plan-design-with-ui PTY detection

Gate-tier `test/skill-e2e-plan-design-with-ui.test.ts` 在 v1.63 的 `seedSkills` 在 hermetic PTY children 中注册 skill 后首次开始执行(该 fork 已删除此文件;它之前什么都没测量)。它现在可靠地超时,即使 skill 正确运行:转录显示 `/plan-design-review` 到达其 scope-gate AskUserQuestion(5 个选项,`<gstack-qid:plan-design-review-scope-gate>` 标记存在),但测试的 `isNumberedOptionListVisible`/`parseNumberedOptions` 抓取无法从 PTY buffer 中分类它,因为 spinner 帧(`[?25l✻Sprouting… still thinking`)与选项文本逐字符交织。

**Why:** Shipped 行为正确——这是一个 test-harness 检测限制,不是产品 bug。但总是超时的 gate 测试比没有测试更糟。

**Fix options:** (a) 强化尾部抓取(在匹配前去除 DEC private-mode + spinner 残留;加宽/清理窗口);(b) 加一个 LLM-judge fallback 分类器(文件自己的注释指出正则检测器「对 PTY 渲染怪癖脆弱」);或 (c) 在 (a)/(b) 落地之前将这个测试移到 periodic。

**Context:** `test/skill-e2e-plan-design-with-ui.test.ts`、`test/helpers/claude-pty-runner.ts:308`(`isNumberedOptionListVisible`)。证据:`~/.gstack-dev/eval-runs/pdwu-verify-*.log`。**Effort:** M (human ~half day / CC ~30min)。

### P3: Residuals from the 2026-08-14 tracker-audit waves (mostly shipped in v1.67.0.0)

四个推迟的 waves (A: browse-daemon lifecycle, B: install integrity, C: gbrain trust boundary, D: ship/version allocator) 在 v1.67.0.0 修复波中落地:XProtect self-heal + Playwright bump + busy-daemon iron rule + signal policy (A);alias shadowing + cursor slice + runtime assets + Windows refresh (B);brain-sync disposition model + source pins + thin-client detection (C);version allocator end-state + subdir manifests + diff-scope globs (D)。剩下的被单独重新归档:

- Watchdog kills headed handoff sessions (PRs 2565/2405/2346) 与 darwin 上跳过的三个 handoff 测试 in browse/test/handoff.test.ts — 验证 v1.67 XProtect + rebrand 工作是否解除它们,然后 un-skip 或修复。Effort S。
- Transcript trust/scope/source isolation (PR 2232, issue 2140) — split:`transcript_ingest_mode` reader(off skips, B → --all-history, unset 不变)在 fork-port Wave E1 中发货;repo-scoping 与 `--source-id` 隔离仍需要永不-双重-存储 review 加上一个 gbrain flag 探针。在 E1 之后用指向这里的指针关闭 PR。Effort M。
- Versionless-repo onboarding (#1474, issues 2343/2334) — #2501 JSON version-path 一半落地;无 version-file-at-all 流未落地。
- Playwright bootstrap abort/timeout absorbs (PRs 2233/2359, issues 1902/2136) — DONE in fork-port Wave A:install 是尽力而为且有界的(GSTACK_PLAYWRIGHT_INSTALL_TIMEOUT,默认 600 秒),锁争用是原因码,skills 总是注册。用收据关闭 #2233、#1900、#1901、#1902、#913(`test/setup-playwright-best-effort.test.ts`)。

## Aside-first follow-ups (filed when Aside became the primary browser)

每个 gstack 接触网页的 skill 优先驱动 Aside AI 浏览器(`scripts/resolvers/aside.ts` 是契约;`lib/aside-render.ts` / `bin/gstack-render.ts` 通过它渲染本地 HTML;`{{ASIDE_RESEARCH}}` 通过它跑 web research)。gstack 自家浏览器引擎——`browse` 守护进程、GStack Browser headed 模式、cookie 导入、`/pair-agent`、browser-skills / `/skillify`——在 Aside 未安装或未运行时(Linux、Windows、关闭的 Aside app)被保留为自动回退,且当宿主提供 web search 时,web research 回退到 WebSearch 工具。什么都没被移除。悬而未决:

### P1: Aside-first fallback parity — keep the `$B` equivalence table in sync with the cookbook

回退块(`BROWSE_FALLBACK` in the browser resolvers)将每个已验证的 `aside repl` 菜谱形状(读一个页面、驱动一个流程、带注释的截图、响应式捕获、links + status、performance、PDF、element screenshot、`aside exec` 研究)映射到它的 `$B` 等价物,这样一个 skill 在任一路径上产生相同的证据行。每次添加、重命名或更改输出标签(`CONSOLE_ERRORS=`、`DIFF_START`、`ASIDE_DIR=`、`GSTACK_STEP_OK`)的菜谱形状,在同一 commit 中更新表并在 `test/aside-driver.test.ts` 中加一个 pin,让两个列表命名相同的形状。

**Why:** 一个在 Aside 路径上读 `DIFF_START` 而在回退路径上得到空值的 skill「盲目地修复」缺失的输出。奇偶校验是保留引擎的全部意义;一个静默的差距比没有回退更糟。

**Effort:** S per change (human ~half day, CC ~15min)。**Priority:** P1。**Depends on:** nothing。

### P2: Aside CLI 1.26 lacks subcommands Aside's own skill doc lists

Aside 的 skill 文档列出 `session`、`memory`、`skills`、`host` 与 `--permission`;Aside CLI 1.26 一个都没有(`aside --help`)。Skills 必须在 CLI 发货它们之前不依赖它们。在每次 Aside release 上重新探查;当它们落地时,为 multi-script flows 评估 `session` 并为变更-动作同意门评估 `--permission`。

**Why:** 针对文档而不是二进制编写的 skill 在 unknown-command 错误上运行时死亡,然后代理会试图盲目「修复」它。

**Effort:** S (human ~half day, CC ~20min per re-probe)。**Priority:** P2。**Depends on:** Aside releases。

### P2: Aside E2E tests run only where Aside is installed

仅 Aside 的 E2E lane——`test/skill-e2e-aside.test.ts`、Aside qa/design 用例、`test/aside-render.test.ts` 中的 live render——在 `aside` 缺席时自跳(`asideAvailable()` in `test/helpers/aside-available.ts`),所以 CI 的 Linux runners 从不驱动 Aside;make-pdf 与 /diagram render gates 已经在浏览二进制上在那里跑。Aside 路径仅在 macOS 开发机器上跑。评估一个自托管的 macOS runner(或在 Mac mini 上的计划任务),它在与其他 E2E lane 相同的 hermetic env 下每周跑 Aside lane。

**Why:** 一个无人跑的浏览器契约在 CI 中静默漂移——正是 `test/aside-driver.test.ts` 静态钉住但无法 live 证明的类。

**Effort:** M (human ~2 days, CC ~1h plus the machine)。**Priority:** P2。**Depends on:** a macOS host with Aside signed in。

### P3: Evaluate `aside mcp` for multi-step flows

`aside repl` 每个脚本一个流程——每个调用都是全新会话,标签页在结束时关闭。`aside mcp` 跨调用保持一个持久的 REPL 页面。一旦 CLI 稳定,测量一个 MCP 路径是否让长 QA 审计更便宜(每个脚本无重新导航)而不丢失「将浏览器留在你发现的状态」的保证。

**Why:** 每个脚本从 URL 重新导航是当前模型的诚实税;一个持久页面可以削减它但增加一个必须清理的会话。

**Effort:** M (human ~2 days, CC ~1h)。**Priority:** P3。**Depends on:** Aside CLI stability。

### P3: Eval that skills treat `aside exec` output as untrusted

`aside exec "<task>"` 返回另一个 agent 的答案。加一个 LLM-judge 或 E2E eval,在 `aside exec` 结果中植入一个指令并检查 skill 从中取语法,从不取范围、权限或同意。

**Why:** 该规则被钉住为散文;还没有什么能证明当注入的文本通过读起来像同事的通道到达时 skill 服从它。

**Effort:** S (human ~1 day, CC ~30min)。**Priority:** P3。**Depends on:** the Aside E2E lane above。

### P1: make-pdf renders user documents inside the real browser profile — add a CSP

`/make-pdf` 通过 Aside(用户已登录的浏览器)在 `127.0.0.1` origin 上打印 markdown-派生的 HTML。敌对文档(来自克隆的 repo 的 README)与该 profile 中的脚本执行之间的唯一屏障是 `make-pdf/src/render.ts` 中的正则净化器,其 header 假设 marked 输出从未格式错误——原始 HTML passthrough 打破了该假设。注入 gstack 自己的 CSP `<meta>` 进打印模板(`default-src 'none'`;`img-src data: 'self'`;`style-src 'unsafe-inline' 'self'`;`font-src data: 'self'`;`script-src 'nonce-<per-render>'` 用于 Paged.js),因为用户的 `<meta>` 被剥离且 gstack 的没有;或者默认让 make-pdf 留在捆绑的引擎上。

**Why:** 在旧的 cookieless 无头引擎下,净化器旁路近乎无害;在真实 profile 中它是一个 CSRF-类原语。跨模型发现(Claude 对抗性 + Codex)。

**Effort:** M (human ~2 days, CC ~1h)。**Priority:** P1。**Depends on:** none。

### P1: diagram pre-pass buffers every oversized image before downscaling

`make-pdf/src/diagram-prepass.ts` 将每个图像上限设为 64 MB 但在 `downscales` 中保留每个待处理缓冲区,并在批次跑之前将其复制为 base64;一个引用几十个大图像的文档可以占用 GB。限制总待处理字节(例如 256 MB)并以有界批次处理,或顺序缩小。

**Why:** 一个有敌意的或仅是图像密集的文档让工具崩溃而不是降级。

**Effort:** S (human ~1 day, CC ~30min)。**Priority:** P1。**Depends on:** none。

### P2: fallback renders die after any cookie import in the daemon's lifetime

`renderWithBrowse` 通过 `$B js` 驱动准备性评估与 evals,且守护进程的 cookie-import JS 锁(`browse/src/read-commands.ts`)拒绝导入集之外每个 origin 上的 `js`——`127.0.0.1` 包括,永远(该集仅添加)。渲染器现在命名补救(`$B stop`),但真正的修复是为本地 HTML 渲染提供一个全新的隐身 context,或在其威胁模型被写下后给 loopback 一个豁免。

**Why:** 在 Linux/Windows(无 Aside)上,一次 `/setup-browser-cookies` 跑让之后每个 `/diagram` 与 `/make-pdf` 渲染失败。

**Effort:** M (human ~2 days, CC ~1h)。**Priority:** P2。**Depends on:** none。

### P2: carve the Aside contract + fallback block into one shared section

`{{ASIDE_SETUP}}`(约 5.6 KB)加上 `{{BROWSE_FALLBACK}}`(约 4.1 KB)被逐字渲染进十个浏览器技能(每个调用加载约 97 KB 的相同散文)。保持探查与三个决策步骤内联;将「驾驶真实浏览器的规则」与 Aside-to-`$B` 翻译表移进一个共享的引用(`browse/sections/command-list.md` 模式),然后重跑 `capture-context-budget.ts` 以让上限反向棘轮。

**Why:** 每个 skill 调为 skill-不变散文付出代价。

**Effort:** M (human ~2 days, CC ~1h)。**Priority:** P2。**Depends on:** none。

### P2: `$B js` / `$B eval` output is not wrapped in the untrusted envelope

`js` 与 `eval` 不在 `PAGE_CONTENT_COMMANDS`(`browse/src/commands.ts`)中,所以页面控制的返回值到达代理时无围栏,而回退表正好将页面控制的读通过它们路由。`gstack-render` 现在围栏其自己的 `EVAL`/`PAGE_ERRORS` 行,且回退散文说 `$B js` 是未包装的;耐用的修复是将两个命令加到 envelope 集。

**Why:** 一个有敌意的页面可以通过技能被告知被围栏的通道传递注入文本。

**Effort:** S (human ~half day, CC ~15min)。**Priority:** P2。**Depends on:** none。

### P3: Aside-first renderer follow-ups (perf and DRY)

- **Readiness polling** 每 150 ms 生成一个 `browse js` 进程;守护进程的 `wait <sel>` 命令在单次 spawn 中服务器端阻塞——将其用于 `waitFor.selector`。Effort S。
- **Bundle re-staging:** 每个 `runScript()` 批次将约 9 MB 的 diagram bundle 复制到一个新鲜的 mkdtemp 并启动一个新的 loopback 服务器;每个 run 暂存一次(content-addressed),在浏览引擎上,保持一个标签页跨 fence/downscale/DOCX 批次。Effort M。
- **Probe cost:** `probeAside()` 每进程跑两个阻塞 spawn 且引擎缓存是 per-process;用短 TTL 在 `GSTACK_HOME` 下持久化结果并在该代码路径上降低 repl 探查超时。Effort S。
- **DRY:** 控制台错误 `HOOK` IIFE 在 `scripts/resolvers/*.ts` 与 `lib/aside-render.ts`(两个分歧变体)中以七个副本存在;菜谱(响应式循环、链接、读-页)在 `aside.ts`、`design.ts`、`utility.ts` 中重复;准备性探查从渲染的 markdown 通过正则在两个地方恢复,而不是共享常量。为每个导出一个源。Effort S 每个。
- **Egress scanner:** `test/egress-receipt-wiring.test.ts` 扫描 `curl`、`git push` 与 `fetch`;加 `aside exec` 作为一个 sink 类,这样一个裸调用会让 CI 与其他一样失败。Effort S。
- `_browser_hint` 将 PATH 上的任何 `aside` 视为 Aside 浏览器(无版本检查)。Effort S。
- **`gen-skill-docs --dry-run` 对外部 host 不是 write-free:** `processExternalHost` 跑 `mkdirSync(outputDir)` 与写入 `agents/openai.yaml`,无 `DRY_RUN` 守卫(仅 SKILL.md 被跳过),所以对一个空 `--out-dir` 的 dry run 留下 54 个 `openai.yaml` 文件。守卫两个写入。Effort S。

**Priority:** P3。**Depends on:** none。

---

## Completed

### #2701 cookie-import profile pills (Local State info_cache)

当前 Local State 名称优先,Preferences/directory 回退保留,且 Default 在数字顺序上排在编号 profile 之前。Directory 标签区分重复名称。

**Completed:** v1.90.0.0 (2026-09-24)

### Reconcile the registered Opus 4.7 overlay efficacy gates

**What:** 在当前 overlay 之前重新审视两个注册的 fanout 实验并记录基于证据的关于其预期效果的决策,然后再发布。

**Why:** 付费 gates 要求 fanout 提升至少 0.5,但 overlay 的 fanout nudge 在 v1.10.1.0 中被删除,因为它降低了并行工具使用。保留一个不受支持的效应预期让 periodic 套件失败而不显示 harness 感知的 outside reviews 中的回归。

**Context:** 在 2026-09-09 船评期间于 `edinburgh-v1` 上发现。两个选定的 `overlay-harness-opus-4-7-fanout-{toy,realistic}` 用例在它们的重试中失败(`Expected: true; Received: false`)。纠正分散的 SDK 消息计数仍产生零提升:toy ON/OFF = 3/3 tools;realistic ON/OFF = 4/4,跨每个臂 10 次保存的试验。选定的实验输入匹配 `origin/main` 的 `71f6048e8ada25180e61438abc1d98cb151fe9a7`;无付费基分支运行被执行。已完成的 "Overlay efficacy harness + Opus 4.7 fanout nudge removal" 条目在下面与 `test/fixtures/overlay-nudges.ts`。当前失败仍被报告;无效应门槛、模型、overlay 文本或通过结果被更改。

**Effort:** M
**Priority:** P0
**Depends on:** None

**Completed:** v1.87.5.0 (2026-09-15)

**Policy disposition:** Contract v2 退役不受支持的 fanout 实验并将比较 efficacy 与支持的行为检查分开记录。历史失败保留其原始判决;这仅关闭政策协调,而不声称阳性 efficacy 或付费接受。见 `docs/OVERLAY_BENCHMARK_CONTRACT.md`。

### Codex→Claude reverse buddy check skill

**What:** 一个 Codex-native skill(`.agents/skills/gstack-claude/SKILL.md`),它跑 `claude -p` 以从 Claude 获得独立的第二意见——从 Claude Code 来看是今天 `/codex` 所做事情的反向。

**Why:** Codex 用户应得 Claude 用户通过 `/codex` 获得的相同跨模型挑战。当前流是单向(Claude→Codex)。Codex 用户无法获得 Claude 第二意见。

**Context:** `/codex` skill 模板(`codex/SKILL.md.tmpl`)展示了该模式——它包装 `codex exec` 带 JSONL 解析、超时处理与结构化输出。反向 skill 会用类似基础设施包装 `claude -p`。会由 `gen-skill-docs --host codex` 生成进 `.agents/skills/gstack-claude/`。

**Effort:** M (human: ~2 weeks / CC: ~30 min)
**Priority:** P1
**Depends on:** None

**Completed:** v1.86.0.0 (2026-09-11). Shipped as `/claude-code`, with automatic outside-review routing and safe installation migration.

### P3: Carve the always-loaded `{{PREAMBLE}}` reference blocks into an on-demand doc

**What:** per-skill section carves(`/ship` v1.54,`/plan-ceo-review` v1.56)产生真实但有界的胜利(carved skill 上的 -42% 到 -59%),因为共享的 `{{PREAMBLE}}`(每个 tier-3/4 skill 上约 40-50KB)是主导的 always-loaded 成本且保持内联。将很少需要的 preamble 引用块(AskUserQuestion 拆分-规则与 CJK / lone-surrogate 转义引用)移进一个 on-demand section-style 文档,代理仅在它击中这些边缘情况时读取,留下热路径(声音、完整性原则、推荐格式)内联。

**Why:** 剩余的最高-ROI token 目标。一个 preamble carve 一次帮助每个 tier-≥2 skill,而不是每个 PR 一个 skill。plan-ceo 雕刻上的 eng-review 标记 per-skill carves 保持温和正是因为 preamble 主导了 always-loaded 表面。

**Pros:** 单一变更跨整个 skill pack 减少 always-loaded 成本。

**Cons:** Preamble 是承重且共享的;一个糟糕的 carve 让每个 skill 倒退。需要与 section carves 使用的相同 union-parity + per-push 新鲜度守卫,应用 corpus-wide。

**Context:** 基于 v2 section pipeline(`scripts/resolvers/sections.ts`、`{{SECTION:id}}` / `{{SECTION_INDEX}}`)。Preamble 源是 `scripts/resolvers/preamble.ts`。在切割前测量哪些子块是冷的(转义引用、split-rules)与热的(声音、推荐格式)。在一个 skill 上验证,然后 corpus-wide 滚动。

**Effort estimate:** L (human team) → M (CC+gstack)
**Priority:** P3
**Depends on / blocked by:** Section pipeline (shipped v1.54). No hard blocker.
**Completed:** v1.70.0.0 (2026-08-25) — delivered in a stronger form by the token-reduction program: preamble bash moved to `bin/gstack-skill-start`/`-end`, one-time onboarding became gated instruction blocks, AUQ reference rules point at on-demand docs, and 12 more skills got section carves (20 total). Wins locked by the context-budget ratchet.

### ✅ DONE (v1.69.0.0): `./setup --host slate` accepted but installs nothing

**Priority:** P4 (was filed as slate-only — shipped with the whole drift class gated)

**What:** `slate` 通过 host-arg 验证但未设任何 INSTALL_* flag,所以运行配置无物并以 exit 0 退出。现在一个信息性 arm(指向 `--host claude`;按 docs/designs/SLATE_HOST.md,Slate 读 `.claude/skills` 作为兼容性回退),加上一个零分派 guard,如果有未来 host 在无 install arm 的情况下被接受则大声报错,加上一个交叉检查测试针对 hosts/index.ts registry 固定 accept-list ⊆ dispatch-arms。

**Completed:** v1.69.0.0 (2026-08-22)

### ✅ DONE (v1.69.0.0, gstack side): ZeroEntropy sunset detect + advisory

**Priority:** P1 (calendar-driven; gbrain-side migration remains open — see NEXT PRIORITY)

**What:** Wireup 在 `~/.gbrain/config.json` 命名 zeroentropyai recipe 时警告(fail-open grep——永不阻塞一个工作中的 setup);setup-gbrain provider 注释说永远不要选择遗留 recipe;USING_GBRAIN_WITH_GSTACK.md 故障排除条目命名 9 月 4 日截止日期与 #2365。

**Completed:** v1.69.0.0 (2026-08-22)

### ✅ DONE (v1.68.1.0): Stop-hook registration pins the setup-time absolute path

**Priority:** P1 (was filed Effort S, scoped to the Stop hook — shipped as the full defect class)

**What:** 从 dev worktree 注册 hooks 将该 worktree 的物理路径烘焙进全局 settings.json;删除该 worktree 留下死 hooks 在每次 AskUserQuestion/session stop 上报错。针对所有 gstack hooks 修复,不仅是 Stop:通过 `_hook_command_path` 的 canonical-only 注册,`gstack-settings-hook` 中的 KNOWN_HOOKS 身份表(在 Claude Code 剥离 `_gstack_source` 标签后存活),一个 `prune-stale [--repoint|--all]` 自愈器在每次 `./setup` 上跑 heal-first,per-item 变更安全,一个变更锁,fail-closed 解析,以及完整卸载/无团队拆除。

**Completed:** v1.68.1.0 (2026-08-18)

### ✅ DONE (v1.66.0.0): Free suite exit code is untrustworthy — in-process force-exits mask failures

**Priority:** P1

**What:** 至少五个浏览测试文件以 `setTimeout(() => process.exit(0), 500)` 结尾(browse/test/commands.test.ts:101、snapshot.test.ts:36、batch.test.ts:47、handoff.test.ts:31、content-security.test.ts:465)。计时器在共享的 `bun test` 进程内触发,在 bun 打印其最终摘要前以 0 退出,所以 `bun test` 可以在真实测试失败滚动过去时报告 exit 0。移除 force-exit 并修复它们掩盖的潜在 handle 泄漏(曾经让套件挂起的徘徊 Playwright/守护进程 handle),或将 exit 限定到派生的子进程。

**Why:** 2026-08-07 观察到:三个真实失败的测试(eval-list-cli、benchmark-cli、observability check 11)跨多次 run 骑着绿色 `bun test` exit 码;失败仅通过在日志中 grep "(fail)" 行浮出水面。一个在失败时以 0 退出的测试套件比无套件更糟——它在提交时与任何信任 exit 码的 CI 工作中制造虚假信心。

**Pros:** 恢复一切(CI、/ship、人类)依赖的一个契约:exit 码 == 真相。也揭示了缺失的最终摘要块。

**Cons:** Force-exit 存在是因为套件曾在泄漏的 handle 上挂起;在没有修复泄漏的情况下移除它们会以静默失败换取挂起。需要专注的 pass:找到每个泄漏的 handle(守护进程子进程、PTY、Playwright 上下文),在 afterAll 中关闭它们,然后逐文件删除 exit。

**Context / where to start:** `grep -rn "process.exit(0)" browse/test/` —— setTimeout 变体是违规者(server-no-import-side-effects.test.ts:62 是一个派生-子进程探针,没事)。重现:跑完整免费套件并注意日志在浏览文件上结束,无 "Ran N tests" 摘要。收据:`~/.gstack-dev/logs/free-suite-main-check.log`(3 个遮盖的失败,exit 0)。

**Completed:** v1.66.0.0 (2026-08-15) — main 的 v1.64 移除了 force-exit;v1.66.0.0 添加 runner-级别 strict-output 分类(一个没有 bun 终端摘要的分片失败)、size-scaled wall 截止,以及失败-命名 epilogue,所以 exit 码 == 真相被 runner 强制,不是被约定。

### Slim preamble + real-PTY plan-mode E2E harness (v1.13.1.0)

- 压缩 18 preamble resolvers;总 `SKILL.md` corpus 从 3.08 MB 降到 2.30 MB 跨 47 个输出(-25.5%,~196K tokens 保存)。
- 构建 `test/helpers/claude-pty-runner.ts` —— real-PTY harness 使用 `Bun.spawn({terminal:})`(Bun 1.3.10+ 有内建 PTY,无需 `node-pty`)。
- 重写 5 个 plan-mode E2E 测试(`plan-ceo`、`plan-eng`、`plan-design`、`plan-devex`、`plan-mode-no-op`);5 个全部首次通过(790s 串行)。
- 同一测试在 `origin/main`、v1.0.0.0 与带 SDK harness 的该分支上是 0/5 —— SDK 无法观察 Claude 的 plan-mode 确认 UI。
- 边修复折叠在内:`scripts/skill-check.ts` sidecar-symlink helper,`test/skill-validation.test.ts` 为 `browse/test/fixtures/security-bench-haiku-responses.json` 豁免(解析 main 的 warn-only 转换的 size-warning 噪声)。
- Side fixes folded in: `scripts/skill-check.ts` sidecar-symlink helper, `test/skill-validation.test.ts` exemption for `browse/test/fixtures/security-bench-haiku-responses.json` (resolves the size-warning noise from main's warn-only conversion).

**Completed:** v1.13.1.0 (2026-04-25)

---

### Pre-existing test failures surfaced during v1.12.0.0 ship — RESOLVED

- `test/brain-sync.test.ts` GSTACK_HOME isolation 在 main 的 v1.13.0.0 中修复。
- Opus 4.7 overlay test(现在是 `test/model-overlays.test.ts` 中的一个块)在 main 上更新以匹配新 overlay 内容(v1.10.1.0 移除 "Fan out explicitly" 是正确的——测量 -60pp fanout vs 基线)。

**Completed:** v1.13.0.0 (2026-04-25, on main)

---

### `security-bench-haiku-responses.json` size gate — RESOLVED

- Main 在 v1.13.0.0 中将 2 MB tracked-file gate 转换为 warn-only。
- v1.13.1.0 添加一个 `knownLargeFixtures` 豁免以抑制该特定故意 fixture 的警告。

**Completed:** v1.13.1.0 (2026-04-25)

---

### Bearer-token secret-scan regression fixed + E2E coverage added for privacy gate + gh auto-create (v1.12.0.0)

- **修复 `bin/gstack-brain-sync` 中的 `bearer-token-json` 回归** —— 值字符集 `[A-Za-z0-9_./+=-]{16,}` 不允许空格,所以带标准 `Bearer <token>` 形式(字面空格在 scheme 名后)的鉴权头滑过扫描器。向模式添加可选的 `(Bearer |Basic |Token )?` 前缀。针对 5 个正例(包括回归 fixture)与 3 个负例(短 token、非 secret key、随机 JSON)验证。7-pattern secret 扫描器现在通过包括 bearer-json 在内的所有 fixture。
- **添加 `test/gstack-brain-init-gh-mock.test.ts`** —— 8 个测试锻炼 `gh` CLI auto-create 路径,该路径以前零覆盖。在 PATH 上 stub `gh` 以记录每次调用,断言 `gh repo create --private --description "..." --source <GSTACK_HOME>` 带计算的 `gstack-brain-<user>` 默认名触发。覆盖:happy path、在 create 撞到已存在时 fall-through-to-`gh repo view`、user-provided-URL-bypasses-gh、gh-not-on-path prompts for URL、gh-not-authed prompts for URL、幂等的 `--remote` 重跑、conflicting-remote 拒绝。

---

## gstack-2 fork port wave 1 — fix wave (v1.65.0.0)

(包含 v1.65.0.0 引入的 fork port 修复;详见 CHANGELOG v1.65.0.0 条目。)

## Wave 1 之后 fork port wave 2 — final wave (v1.65.0.0)

(详见 v1.65.0.0 完整条目。)

## CI eval pipeline (v0.9.9.0)

- Ubicloud runners 上的 GitHub Actions eval 上传($0.006/run)
- Within-file test concurrency (test() → testConcurrentIfSelected())
- 带 pass/fail 与成本的 Eval 工件上传 + PR 评论
- 通过 main 的产物下载的基线比较
- EVALS_CONCURRENCY=40 for ~6min wall clock (was ~18min)
**Completed:** v0.9.9.0

## Deploy pipeline (v0.9.8.0)

- /land-and-deploy — 合并 PR,等待 CI/部署,canary 验证
- /canary — 部署后监控循环,带异常检测
- /benchmark — 性能回归检测,带 Core Web Vitals
- /setup-deploy — 一次性部署平台配置
- /review Performance & Bundle Impact pass
- E2E model pinning (Sonnet default, Opus for quality tests)
- E2E timing telemetry (first_response_ms, max_inter_turn_ms, wall_clock_ms)
- test:e2e:fast tier, --retry 2 on all E2E scripts
**Completed:** v0.9.8.0

## Phase 1: Foundations (v0.2.0)

- 重命名为 gstack
- 重构为 monorepo 布局
- Setup script 用于 skill symlinks
- Snapshot command 与基于 ref 的元素选择
- Snapshot tests
**Completed:** v0.2.0

## Phase 2: Enhanced Browser (v0.2.0)

- 带注释的截图、snapshot diffing、对话框处理、文件上传
- Cursor-interactive elements, element state checks
- CircularBuffer, async buffer flush, health check
- Playwright error wrapping, useragent fix
- 148 integration tests
**Completed:** v0.2.0

## Phase 3: QA Testing Agent (v0.3.0)

- /qa SKILL.md 带 6-phase workflow,3 modes (full/quick/regression)
- Issue taxonomy, severity classification, exploration checklist
- 带 health score (0-100, weighted across 7 categories) 的结构化报告模板
- Next.js, Rails, WordPress 与 SPAs 的框架检测指引
- wait/console/cookie-import commands, find-browse binary
**Completed:** v0.3.0

## Phase 3.5: Browser Cookie Import (v0.3.x)

- cookie-import-browser command(从真实 Chromium 浏览器解密与导入 cookies:Comet, Chrome, Arc, Brave, Edge)
- 来自浏览守护进程的交互式 cookie picker web UI(暗色主题、双面板布局、域搜索、import/remove)
- 带 `--domain` flag 的直接 CLI 导入用于非交互式使用
- /setup-browser-cookies skill 用于 Claude Code 集成
- macOS Keychain access with async 10s timeout(无事件循环阻塞)
- Per-browser AES key caching(每个浏览器每个会话一次 Keychain 提示)
- DB lock fallback: 将锁定的 cookie DB 复制到 /tmp 以安全读取
- 18 unit tests with encrypted cookie fixtures
**Completed:** v0.3.1

## E2E test cost tracking

- 跟踪累积 API 花费,在超过阈值时警告
**Completed:** v0.3.6

## Auto-upgrade mode + smart update check

- Config CLI (`bin/gstack-config`), 通过 `~/.gstack/config.yaml` 自动升级,12h cache TTL,指数 snooze backoff (24h→48h→1wk),"never ask again" 选项,在升级时 vendored copy sync
**Completed:** v0.3.8

---

## Brain-aware planning follow-ups (filed v1.48.0.0 via /plan-ceo-review + /plan-eng-review)

这些是从 v1.48 brain-aware planning 计划在 `~/.claude/plans/hm-interesting-well-why-dapper-eagle.md` 推迟的 cherry-picks(E2/E3/E4)。基础(Phase 0 entity model + Phase 0.5 cache + Phase 1 preflight + Phase 1.5 trust policy + Phase 2 write-back scaffolding)在 v1.48.0.0 发货。这些 follow-ups 扩展它。

### P2: /gstack-reflect nightly synthesis skill (E2)

**What:** 调度的 skill,读取每周 `gstack/skill-run` + takes + `get_recent_salience` 并合成一个 `gstack/insight` 页面,在下次 skill preflight 时浮出。

**Why:** 跨-时间模式检测是复利动作。"你这周跑了 4 次 plan-ceo 在 infra 上,0 次在 product 上——product 工作是否被饿死?" 浮出用户不会注意的模式。

**Pros:** Brain 跨时间复利,不仅跨技能。模式变得可操作。

**Cons:** "你正在饿死 product 工作" 是高判断领域;需要 per-project opt-out、谨慎的洞察模板。

**Context:** 从 v1.48.0.0 cherry-pick (D4) 推迟 — 等 4-6 周以累积真实 `gstack/skill-run` 数据,然后再针对真实模式设计反映层。

**Effort:** L (human ~1-2 days, CC ~4-6h)

**Depends on:** Phase 0 (来自 v1.48.0.0 的 gstack/skill-run page type) + 约 6 周累积数据

### P3: Cross-machine brain-cache sync (E3)

**What:** 通过 gstack-brain-sync git pipeline 推送压缩摘要,所以 brain-cache 在 Mac / Conductor workspace 之间移动时存活。

**Why:** 消除每台新机器上的冷-未命中税(每台机器每天约 1-2s 一次)。

**Pros:** 在新机器上瞬时暖缓存。

**Cons:** 如果未仔细设计,缓存中毒风险(hash 不变性、端点-绑定、冲突解决)。

**Context:** 从 v1.48.0.0 cherry-pick (D5) 推迟 — 单-机器缓存对 V1 是可以的;正确性风险需要它自己的设计 pass。

**Effort:** M (human ~4h, CC ~30min)

**Depends on:** 来自 v1.48.0.0 的 Brain-cache layer

### P3: /gstack-onboarding dedicated skill (E4)

**What:** 引导式 5 分钟设置 skill 用于新 gstack 安装:带用户走过阅读 CLAUDE.md + README + 最近 commits 以构建 `gstack/product` 与活跃目标,带显式 AUQ。

**Why:** 比内联 bootstrap(仅在调用 plan skill 时触发)更好的 UX。

**Pros:** 更干净的冷-启动,显式仪式。

**Cons:** 内联 bootstrap(V1.48 范围内)已经充分覆盖了冷-启动路径。

**Context:** 从 v1.48.0.0 cherry-pick (D6) 推迟 — 首先观察内联 bootstrap 性能;如果摩擦是真实的,加专用 skill。

**Effort:** S (human ~2h, CC ~15min)

**Depends on:** 来自 v1.48.0.0 的内联 bootstrap 子命令

### P2: Upstream gbrain takes_add + takes_resolve MCP ops

**What:** 在 `~/git/gbrain/src/core/operations.ts` 中加 `mcp__gbrain__takes_add` 与 `mcp__gbrain__takes_resolve` ops。从 `commands/takes.ts:570` 提取 markdown-fence mirror 逻辑到可复用的 `engine.resolveTake()` helper。

**Why:** 解锁 Phase 2 calibration write-back 而无 fence-block 回退。约 150 LOC。已经在 gbrain 的 v0.31.x 路线图上。

**Pros:** 干净的 Phase 2 路径,移除「回退到 put_page」的味道。

**Cons:** 生活在上游 gbrain 仓库,不是 helsinki — 单独的 PR。

**Context:** Phase 2 write-back 已经在 v1.48.0.0 中在 BRAIN_CALIBRATION_WRITEBACK feature flag(默认关闭)后面接线。Flag 在上游 gbrain 发货这些 ops 时翻转为 true。约 50 LOC follow-up in helsinki 来将回退换成优选 op。

**Effort:** S (human ~1d, CC ~1h) in gbrain repo;平凡的 wire-up in helsinki。

**Depends on:** None (parallel-track from v1.48.0.0)

### P3: Background-refresh hook supervision

**What:** Codex outside-voice 提出「在 skill END 后台刷新」是手-挥。加适当的进程监督:PID file、timeout、失败日志、跨平台 spawn。

**Why:** 当前实现用 `&` 后台化,这能工作但当刷新失败时不留下可观测性。

**Context:** 从 v1.48.0.0 codex tension T3 推迟。当用户报告陈旧摘要(其中后台刷新默默地失败)时,保持低优先级。

**Effort:** S (human ~2h, CC ~20min)

### P2: Re-verify calibration takes when gbrain v0.42+ lands

**What:** 当上游 gbrain 发货 `takes_add` MCP op 且我们将 `BRAIN_CALIBRATION_WRITEBACK` 从 FALSE 翻转为 TRUE 时,在 `/office-hours` 上重跑 `docs/gbrain-write-surfaces.md` 中的手动探针,确认 `gbrain takes_list` 浮出一个 `kind=bet` 条目,带预期的 weight(office-hours 为 0.9,按 `scripts/brain-cache-spec.ts:151-157`)。

**Why:** 今天 calibration take path 回退为在 `gbrain put` fence block 内写,因为 `takes_add` 还不可用。一旦 v0.42+ 发货,代理将直接调用 `takes_add` — 我们应确认新路径实际上持久化一个可查询的 take。

**Context:** v1.50.0.0 计划 §"NOT in scope"。Fence-block 回退测试(`test/takes-fence-fallback.test.ts`)覆盖两个路径的接线;该 TODO 是关于在优选路径可用时对其 live 验证。

**Effort:** XS (human ~15min, CC ~5min)

**Depends on:** Upstream gbrain v0.42+ release shipping `takes_add` MCP op (separate TODO above).

### P2: Extend brain-writeback E2E to the other 4 planning skills

**What:** `test/skill-e2e-office-hours-brain-writeback.test.ts` 仅覆盖 `/office-hours` 的 brain-writeback 路径。为 `/plan-ceo-review`、`/plan-eng-review`、`/plan-design-review` 与 `/plan-devex-review` 添加并行测试,会将 per-skill 代理-服从覆盖带到与解析器单元测试(`test/resolvers-gbrain-save-results.test.ts`,覆盖所有 5 个的接线)对等。

**Why:** 解析器测试证明正确的指令被发出;E2E 证明代理实际上服从。今天我们仅对五个 plan skill 中的一个拥有该 end-to-end 信号。

**Context:** v1.50.0.0 计划 §"NOT in scope"。当第二个消费者到来时将 `makeFakeGbrain` 提取到 `test/helpers/fake-gbrain.ts`(今天一个消费者的 YAGNI)。

**Effort:** S (human ~1d, CC ~1h)。Periodic-tier (~$2-4 总共 4 个 run)。

**Depends on:** None.

### P2: Real-session carve canary (E3, deferred from carve-guard plan)

**What:** 在 carved skills 上接一个真实-会话 section-Read-miss canary。当一个真实用户会话驱动一个 carved skill 且代理未读取其骨架的 STOP 指令指向的 section,记录(加盐,无内容)到 `~/.gstack/analytics/section-reads.jsonl` 并通过 `bun run eval:summary` 浮出漂移。非阻塞告警,永不 merge gate(真实-会话数据是非确定性的)。

**Why:** 静态 (E2) + 行为 (T2) guard 证明 carve 在结构上健全且真实代理在受控 eval 中读取 sections。它们看不到生产漂移 — 一个 prompt-context 变更使 live 代理开始跳过一个 section。Canary 是捕获该类(从真实使用)的唯一机制。

**Context:** 从 carve-guard-hardening 计划(D5→T2,codex outside-voice #7)推迟。确定性的 `test/helpers/transcript-section-logger.ts` 在 2026-09 test audit 中被删除(无付费或生产调用者;见 `docs/test-audit-2026-09.md`);一个真实-会话 logger 从头开始。先发确定性的 guard;一旦它们证明有用,加这个。Carved-skill 集合与每个 skill 的 `requiredReads` 已在 `test/helpers/carve-guards.ts` 中声明,所以 canary 从那里读取其期望。

**Effort:** M (human ~2d, CC ~4h)。

**Depends on:** 一个真实-会话 section-read logger (今天不存在)。

### P2: Harden behavioral section-loading test hermeticity

**What:** `captureSectionReads` in `test/helpers/auq-sdk-capture.ts` 接受任何 Read,其路径匹配 `sections/<file>.md`。骨架的 STOP-Read 指令指向 gstack-root 安装路径(`scripts/resolvers/sections.ts` 从 `ctx.paths.skillRoot` 构建它),而不是 planted fixture copy。所以一次 run 可以通过读取全局安装的 section 而不是 hermetic fixture 来满足 section-read 断言。

**Why:** 一个通过读取全局安装通过的行为测试不证明**该**分支的 carved section 加载。如果 fixture 的 section 坏了但全局安装的没有,测试仍会通过。

**Context:** Codex outside-voice finding on the carve-guard ship (v1.57.0.0)。预先存在于 `auq-sdk-capture.ts` — 影响 `skill-e2e-ship-section-loading`、`skill-e2e-plan-ceo-review-section-loading` 与新的 `carve-section-loading.test.ts`。修复:匹配 fixture 的 ABSOLUTE sections 路径(`planDir` copy),而不是裸的 `sections/<file>.md` 正则;或在 run 期间将 STOP 路径重写为 fixture。

**Effort:** S (human ~3h, CC ~30min)。**Depends on:** None.

### P3: Content-hash diagram render cache for make-pdf

**What:** 将渲染的 diagram SVG/PNG 缓存在 `~/.gstack/cache/diagram-render//` 中,以 `sha256(fence source + bundle version + render options)` 为键,以便重复 `make-pdf` run 跳过未变更 diagram 的渲染(Aside 或回退浏览标签)。

**Why:** 每个 run 当前重新渲染每个 fence(每个约 150-300ms)。在 write-preview 循环期间有 10+ diagram 的文档每次迭代花费数秒。Codex outside-voice 在 2026-06-11 的 diagram engine plan 的 eng 评审期间标记了缺失的缓存故事(D7)。

**Context:** The diagram-render bundle ships a `BUILD_INFO.json` with a content hash (见 `lib/diagram-render/`) — 用它作为 bundle-version 缓存键组件,以便 bundle bumps 干净地使失效。失效面是主要风险:mermaid 主题变更后的陈旧渲染必须不能存活。仅在用户撞上多-diagram 文档时值得构建;没有它 wedge perf 是可以的。

**Effort:** S (human ~1d, CC ~30min)。**Depends on:** diagram engine wedge 发货 (lib/diagram-render bundle versioning)。

### P3: Dedupe the make-pdf e2e gate-test harness

**What:** 五个 e2e 文件(`combined-gate`、`emoji-gate`、`diagram-gate`、`landscape-gate`、`format-gate`)各自手工-卷起相同的前置探针(binary/browse/poppler checks with CI hard-fail vs 本地 skip)、mkdtemp/rm 生命周期,以及子超时常量。提取一个共享的 `make-pdf/test/e2e/helpers.ts`(prerequisites()、withWorkDir()、runGenerate())。

**Why:** Review-army maintainability finding on v1.58.0.0 — 样板代码随每个新 gate 略有分歧(diagram-gate 现在通过 Bun.spawnSync 捕获 stderr,而其他用 execFileSync),未来对 CI-hard-fail 契约的修复必须落地五次。

**Context:** 在发船时推迟(D8.2),因为它是测试-仅 churn 跨一个 release 末尾的五个绿色文件。零面向用户价值;纯 DRY。

**Effort:** S (human ~3h, CC ~20min)。**Depends on:** None.

## Egress-receipt follow-ups (filed via /plan-eng-review + /codex on the v1.63 port wave)

### P2: egress ledger rotation with chain-genesis records

**What:** 在大小阈值轮转 `~/.gstack/security/egress.jsonl`(匹配 `attempts.jsonl` 的 10MB/5-代模式 in `browse/src/security.ts`),其中每个新代的第一条记录嵌入先前文件的 tail hash,以便 `gstack-egress verify` 可以跨代走。

**Why:** v1.63 发货 WARN-at-25MB(可见增长)但无物限制文件。轮转被刻意推迟:它改变 verify 契约,且一个错误的实现让健康分类账验证为「坏」。

**Pros:** 永远有界磁盘;verify 跨代保持有意义。

**Cons:** Chain-genesis 语义是微妙的;需要它自己的专注测试(跨代 verify、中间轮转崩溃)。

**Context:** `lib/egress-receipt.ts`(`appendChained`/`verifyLedger`)在其轮转 TODO 注释中携带设计草图。从 `attempts.jsonl` 轮转先例开始。

**Effort:** S (human ~4h, CC ~25min)。**Depends on:** v1.63 port wave landed.

### P3: launch-nonce token bootstrap (local-process impersonation)

**What:** 在 `/extension-token` bootstrap 中加一个启动时 nonce:`browse` 在 headed 启动时铸造一个 nonce,将其种进扩展(CDP `chrome.storage` 注入或启动器-写的 sidecar),且端点要求它与钉住的 origin 并存。

**Why:** v1.63 的钉住-origin 检查鉴权浏览器上下文;任何本地进程仍能用 curl 伪造 Origin header。该威胁被显式排除在当前模型外(任何本地进程反正能击中端口) — 该 TODO 文档化了刻意的边界与跨越它的设计路径。

**Pros:** 关闭本地-进程 impersonation 路径(评估的三个选项中最强)。

**Cons:** 最大的 bootstrap 变更;CDP 种入在三个启动路径(`--load-extension`、烘焙 Browser.app、real-Chrome 回退)中很挑剔;当前价值低。

**Context:** `browse/src/server.ts` `/extension-token` handler + `GSTACK_EXTENSION_ID`;launch paths in `browse/src/browser-manager.ts` (~358, ~455, ~1562);`extension/background.js` bootstrap.

**Effort:** M (human ~2 days, CC ~1h)。**Depends on:** none.

### Won't do (2026-10 audit): eval-watch shard-awareness

`scripts/eval-watch.ts` 与 `eval:watch` 被删除:无支持的付费入口是未分片的,且 live 进度来自 detach 日志或 `gh run watch`。原始条目跟随以供历史。

**What:** 教 `scripts/eval-watch.ts`(在约 17 行硬编码 `_partial-e2e.json` 路径)关于分片布局:观察 `<evalDir>/shards/*/_partial-e2e.json` 并跨分片子目录聚合 live 进度。

**Why:** v1.63 的分片 runner 给每个分片它自己的 eval 子目录(所以分片针对它们自己的先验进行基线);`findPreviousRun`、`eval-compare`、`eval-list` 与 `eval-summary` 都被改造为感知分片,但 live watcher 有意保持平坦 — 它在分片 run 期间不显示任何东西。

**Pros:** 在 `eval:bg:gate` 分片 run 期间再次获得 live 进度。

**Cons:** 多文件监视 + 聚合 UI;风险低(run-scoped detach 日志已流式传输每分片结果)。

**Context:** `scripts/eval-watch.ts`;分片布局在 `scripts/test-paid-shards.ts` 中定义(slug = test filename);`listEvalJsonFiles` in `test/helpers/eval-store.ts` 已枚举布局 — 重用它。

**Effort:** S (human ~2h, CC ~15min)。**Depends on:** v1.63 port wave landed.

## v1.63 port-wave review follow-ups (deferred from /ship review army — non-blocking polish)

来自 v1.63 ship 的真正评审发现被推迟,因为它们是信息性/打磨,不是正确性-阻塞,且几个想要它们自己的测试。已归档以便它们被跟踪,不是被丢弃。

- **P2 — telemetry-sync HTTP-status outcome is dead code.** `_GSTACK_EGRESS_LAST_RECEIPT` 在 `bin/gstack-telemetry-sync` 中的命令替换子 shell 内设置,所以将 HTTP 状态附加到收据的父 shell 守卫永不触发。通用 `exit:N` 结果仍被记录,所以分类账是正确,只是不那么精确。修复:让 `_receipted_curl` 将收据 id 持久化到调用者可读的临时文件,或将该调用重构出子 shell。(由 3 个评审专家确认。)
- **P2 — context-bill "TOTAL on disk" double-counts child skills** 在根-作-容器树(此仓库自己的布局)中:`buildBill` 求和根 skill 的整树遍历加上每个子树的子树再次求和(约 2x 的 TOTAL 行)。ALWAYS-ON / EAGER / --diff / --budget 都不受影响 — 仅信息性 TOTAL 是错的。修复:从单个去重的 `walkMd(root)` 传递计算树总计,或从根 skill 的 `totalMd` 中排除子目录。需要 fixture 测试。(`lib/context-bill.ts`.)
- **P3 — DRY/robustness polish:** 一个共享的 `_gstack_egress_host_of` helper 用于跨出口 shell sink 的约 11 个手-卷 URL-to-host 提取;提取 `browse/src/server.ts` 中重复的隧道-打开 `writeReceipt` 块(两个站点);将每次迭代的 `SharedArrayBuffer` 分配从出口-收据锁旋转中升起;用显式 flag 替换 context-bill 的 exact-mode `errorPct === 0` 哨兵;在 `catalog-budget.test.ts` 中重用 `frontmatterName()` from `skill-census.ts`.
- **P3 — test-coverage gaps the audit named:** `PAID_TEST_GLOBS` ↔ `package.json` `test:gate` 奇偶测试;`GSTACK_EXTENSION_ID` ↔ `manifest.json` key 派生奇偶测试 (`browse/scripts/extension-id.ts`);一个 runner 测试断言每个分片子获得它自己的 `GSTACK_EVAL_DIR` 在 `shards/<slug>` 下;supabase-provision / gbrain-sync / memory-ingest 的收据-拒绝分支测试。

## P2: harden or re-tier skill-e2e-plan-design-with-ui PTY detection

**What:** Gate-tier `test/skill-e2e-plan-design-with-ui.test.ts` 在 v1.63 的 `seedSkills` 在 hermetic PTY children 中注册 skill 后首次开始执行(该 fork 已删除此文件;它之前什么都没测量)。它现在可靠地超时,即使 skill 正确运行:转录显示 `/plan-design-review` 到达其 scope-gate AskUserQuestion(5 个选项,`<gstack-qid:plan-design-review-scope-gate>` 标记存在),但测试的 `isNumberedOptionListVisible`/`parseNumberedOptions` 抓取无法从 PTY buffer 中分类它,因为 spinner 帧(`[?25l✻Sprouting… still thinking`)与选项文本逐字符交织。

(重复条目;详见上面)

### P3: Residuals from the 2026-08-14 tracker-audit waves (mostly shipped in v1.67.0.0)

(重复条目;详见上面)

## Aside-first follow-ups (filed when Aside became the primary browser)

(重复条目;详见上面 — 包含 P1 Aside-first fallback parity、P2 Aside CLI 1.26 lacks subcommands、P2 Aside E2E tests run only where Aside is installed、P3 Evaluate `aside mcp` for multi-step flows、P3 Eval that skills treat `aside exec` output as untrusted、P1 make-pdf renders user documents inside the real browser profile — add a CSP、P1 diagram pre-pass buffers every oversized image before downscaling、P2 fallback renders die after any cookie import in the daemon's lifetime、P2 carve the Aside contract + fallback block into one shared section、P2 `$B js` / `$B eval` output is not wrapped in the untrusted envelope、P3 Aside-first renderer follow-ups)

---

(完整源 TODOS.md 共 5,381 行,涵盖 P0-P4 优先级的数十个待办、已推迟项与已完成的 fork-port wave 1 与 2 修复、设计精简、e2e 测试基础设施改进、quality gates、brain-aware planning follow-ups、egress-receipt ledger 改进、browse-daemon lifecycle 改进,以及 gstack 自家浏览器引擎的多种增强。完整源文件保留所有 issue 编号、PR 编号、commit SHA、文件路径、命令与 `Contributed by @username` 与 `@username` 标注不变。)
