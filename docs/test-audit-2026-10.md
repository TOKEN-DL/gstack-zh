# 测试、评估和 CI 审计 — 2026 年 10 月

本文档记录了 2026 年 10 月测试、评估和 CI 浪潮背后的证据:EVAL_POLICY v2 决策备忘录(D1)及其预注册的回溯测试,以及重现每个数字的命令。需要合并后流量的指标使用 `bun run test:health --since-days 7`(合并 + 7 天)和 `bun run test:health --since-days 28`(在第四次计划普查之后)测量。

## EVAL_POLICY v2 决策备忘录(D1)

### 为什么普查是红色的

每周普查在算术上是红色的,而不是通过一些损坏的测试。在 22 次普查中,具有试验结果的周期 lane 记录了 3,550 次失败试验中的 76 次失败(每次运行约 3.5 次失败试验),gate 普查在 2,241 次中失败 27 次(每次运行约 1.2 次)。具有独立失败,P(green)对于周期 lane 是 e^-3.5 ≈ 3%,对于 gate 普查是 e^-1.2 ≈ 30%;观察到的 green 率是 0/22 和 8/22。将约 190 个单试验 lane 保持 80% 的时间为绿色需要每次运行的失败率约 0.12% 或更低,这是由一次试验判断的实时智能体无法达到的。修复前五个慢性用例使 lane 保持红色。

三个杠杆保持在不重试、不阈值、不重新判断的规则内:使检测器对结构化工件确定性(W3),诚实地分类 INFRA(W2d,由可靠性后续拥有),并使隔离对检测器、harness 和 model-latency 失败可达到。v2 针对第三个杠杆。

`test:health` 将普查报告为三个单独的数字,因此策略更改不能读取为可靠性增益:每个种类和失败类的原始试验失败率、来自记录面板下合并的每用例率的 P(lane red),以及使用每次试验判断的每个用例的相同证据重新聚合。两个红色概率将每个观察到的失败视为非确定性,因此它们是已知良好输入的上限。

**错误阻塞目标。** 周期 lane:在 28 天检查时最多 1.0 预期失败试验每次普查(P(red) ≤ 63%);gate 普查:最多 0.5(P(red) ≤ 39%)。任何目标都不能通过阈值、重试或重新判断更改来实现。未达到目标会打开一个 P1 TODOS 条目,其中包含来自 `test:health` 的每种类、每失败类别的细分。

### v2 更改的内容

- **系列标识(W4a、CEO-27、ENG-3)。** 用例的通过率系列由它拥有的字节加上 `HARNESS_VERSION` 键控:其付费测试文件、其 fixture,以及它命名的 skill 的提示文件(模板、sections、清单和 model 读取的生成的 `SKILL.md`)。系列键添加 model id、固定的 Claude Code CLI 版本和策略版本。共享 helpers 不再启动新系列。每次试用还记录完整的消耗输入指纹(所有触摸文件,包括 globals)为 `series_fingerprint`,加上 `harness_version`,作为出处。(`scripts/eval-trial-series.ts` 中的 `caseSeriesIdentitiesV2`)
- **Harness 保护。** `scripts/harness-version.json` 固定每个共享执行 harness 文件的 blob:`GLOBAL_TOUCHFILES` 加上会话 runner、PTY harness、e2e gate 和 LLM judge(36 个文件)。免费测试(`test/harness-version.test.ts`)在一个更改而没有决定时失败。`bun run scripts/bump-harness-version.ts --bump "<reason>"` 提高 `HARNESS_VERSION` 并为每个用例启动新系列;`--non-behavioral "<reason>"` 将新 blob 记录为兼容而不重置。每个文件的条目是一行,因此触及不同文件的并发 PR 可以干净地合并;在冲突时,采用更高的版本,然后重新运行。
- **每周历史(W4b)。** 用于隔离过期和漂移的周是 `evals-periodic.yml` 在 `main` 上运行:计划的运行和 main 调度。同一窗口中来自分支普查运行的试验汇集到一个系列 `main` 也运行了(下面的选项 (b),2026-10-05 批准);`main` 未运行的分支标识被丢弃并且永远不会成为当前系列。
- **种类(W4c)。** 没有种类更改;请参见下面的证据。
- **Pre-v2 证据不符合条件。** 读者严格按 `policy_version` 细分:只有 v2 试验计入隔离条目规则;v1 试验保持可见,仅作为显示。隔离不会在合并时播种;用例仅在达到 `entry.minTrials` 合格的 v2 试验后才进入,并且 `test:health` 显示合格与必需计数,其中"证据不足"与健康不同(DX-12)。

### W4c:命名的"随机"用例在记录中是确定性的

记录的普查和 PR 试验(所有 95 个记录的修订,2026 年 10 月):

| 用例 | 种类 | 通过/试验 | 失败 |
|---|---|---|---|
| plan-ceo-finding-floor | rule | 26/27 | 1 个超时 (608 s) |
| plan-devex-finding-floor | rule | 27/27 | — |
| plan-eng-finding-floor | rule | 9/9 | — |
| plan-design-finding-floor | rule | 9/9 | — |
| plan-ceo-review-plan-mode | rule | 34/34 | — |
| plan-devex-review-plan-mode | rule | 27/27 | — |
| plan-eng-review-plan-mode | rule | 9/9 | — |

将 `plan-*-finding-floor` 称为"随机 ask-first" 的评论已过时:记录显示这些用例表现得像规则。AGENTS.md 禁止在没有通过率证据的情况下更改种类,并且 3 次试验面板会将其成本增加三倍而无可测量的收益,因此它们保持 `rule`。`plan-design-review-plan-mode`(17/22)由其检测器修复处理,而不是种类更改。

### 回溯测试(ENG-3、DX-12)

`bun run scripts/eval-trial-series.ts --backtest --weeks 30 --records <dir>` 通过今天的注册表重放历史树的身份(在某个修订中不存在的测试文件的用例在那里缺失)。

**每周主要修订**(29 个星期一,06:00 UTC,2026-03-16 至 2026-09-28)。用例在一周内合格,当一个身份保持足够长的时间以在最近 10 次每周运行中收集 10 次试验(规则和 judge 用例记录每次运行一次试验,行为用例记录三个试验的面板):

| 身份 | 在最近一周合格 | 平均合格份额,最近 8 周 | 每个用例的中位身份更改 |
|---|---|---|---|
| v1(自己的触摸文件减去 globals) | 0/195 | 0.3% | 15 |
| v2(case-owned 字节) | 0/195 | 0.7% | 14 |
| v2,每个固定 harness 更改时重置 | 0/195 | 0.1% | 14 |
| v2 不带生成的 `SKILL.md`(原型) | 0/217 | 2.0% | 7 |

**记录的试验** 按产生它们的修订的 v2 身份重新键控(在单个系列上至少 10 次试验的阻塞用例):

| 试验来源 | 记录的 (v1) 身份 | v2 身份 |
|---|---|---|
| 22 次普查运行(5,665 次试验) | 52/210 | 67/210 |
| 普查 + PR lane,95 次修订(14,898 次试验) | 107/210 | 134/210 |

**Harness churn。** 在 28 个连续周的过渡中,固定的 harness 改变了 24 次,最近 60 个主要提交(2026-08-04 至 2026-10-04)中有 33 个触及了固定文件。因此大约一半的 PR 因此记录了一个 `bump-harness-version` 决定,并且只有 `--bump` 决定重置系列。

### 回溯测试显示的内容

1. v2 使系列诚实且在试验密集时更长:在记录的普查运行中,67 个阻塞用例在一个系列上达到 10 次试验,而 v1 下为 52 个,包括 PR-lane 试验则为 134 个对 107 个。
2. 在批准每周历史(主要计划的运行加上主要调度,大约每周一次运行)下,没有阻塞用例在任何最近一周达到 10 次合格试验。规则用例需要连续十次每周运行,字节不变,而用例自己的提示文件大约每两周更改一次(生成的 `SKILL.md` 携带共享 preamble)。排除生成的 `SKILL.md` 将身份 churn 一分为二,但仍然最多合格 2% 的用例,并且它将允许 preamble 和解析器提示更改汇集,除非每次此类编辑都通过 harness 保护。
3. 因此 v2 单独不能使隔离可达到。要达到它需要以下之一:(a) 按批准保留历史并接受隔离仍然罕见,因此普查红色在源处修复(W3)并由 `test:health` 跟踪;(b) 从每次 v2 普查运行中汇集试验,其系列键匹配,包括分支调度,同时用于过期的周仍然仅从 main 计数 — 记录的回溯测试的 67/210,代价是汇集未增加 harness 的分支代码编辑匹配;或 (c) 在未更改的主要修订上进行额外的普查调度(每次约 $175)。Garry 在 2026-10-05 批准了 (b),并且此 PR 实现了它:汇集的分支试验仅计入 `main` 运行也具有的系列。
4. 因此 v2 单独不能使隔离可达到。要达到它需要:(a) 按批准保留历史并接受隔离仍然罕见,因此普查红色在源处修复(W3)并由 `test:health` 跟踪;(b) 从每次 v2 普查运行中汇集试验,其系列键匹配,包括分支调度,同时用于过期的周仍然仅从 main 计数 — 记录的回溯测试的 67/210,代价是汇集未增加 harness 的分支代码编辑匹配;或 (c) 在未更改的主要修订上进行额外的普查调度(每次约 $175)。Garry 在 2026-10-05 批准了 (b),并且此 PR 实现了它:汇集的分支试验仅计入 `main` 运行也具有的系列。

### 考虑的替代方案

- **门控 vs 趋势。** 原生 CEO voice 建议保持实时智能体用例趋势(报告,从不阻塞)并仅在确定性检测器上设门。v2 保持普查阻塞,因为隔离和通过率 gate 已经为每个用例提供了趋势路径;上面错误阻塞目标是该选择的检查。如果 28 天检查未达到它们,则对剩余随机用例进行趋势报告是下一个提案。
- **自适应顺序面板**(一次试验,在失败时升级到三次,跨约 190 个用例约 1.04 倍成本)将是绿色普查的最强杠杆。它们未被采用:由失败触发的升级是重试,Garry 的规则禁止。
- **购买 vs 构建。** 托管评估平台(Braintrust、LangSmith、Weights & Biases Weave)本地存储按用例的通过率历史和成本,但 gstack 的 verdict 是在 GitHub Actions 中从具有预注册面板和收据的 Claude Code 会话中产生的,并且历史是每周 `gh` 已服务的几千行 JSONL。一个平台会增加外部依赖项和 verdict 的第二个事实来源,而无需删除仓库中的策略。在现有工件上构建仍然是选择;如果历史超出工件保留(90 天),请重新审视。

### 推出和回滚(ENG-14、DX-12)

读者兼容性提交首先落地:读者仅对其自己的 `EVAL_POLICY.version` 下的试验进行评分,将较旧策略的试验显示为仅显示,并忽略具有打印计数的新策略的试验。v2 writer(身份盖章和 `version: 2`)最后落地。受支持的回滚是恢复 v2 writer 提交;不声明整个 PR 回滚是安全的。

在混合的 v1/v2 fixture(10 个通过的 v1 试验,12 个 v2 试验有 3 个失败)上,2db0b3a 的预 PR 读者将 v2 记录汇集为当前 v1 系列并失败其 gate:

```
pass-rates: policy v1, 22 post-policy trial(s), 0 pre-policy (display only)
  FAILING       rule      gate      9/12 [46.8%–91.1%]  ...  plan-ceo-review-plan-mode  (baseline reset)
ACTION REQUIRED (2): [drift] ... [rule-as-behavior] ...
exit=1
```

兼容性读者忽略 v2 记录并保留 v1 系列:

```
pass-rates: policy v1, 10 post-policy trial(s), 0 pre-policy (display only)
  ignored 12 trial(s) recorded under a policy newer than v1: this checkout predates them. Fix: ...
  PASSING       rule      gate      10/10 [72.2%–100%]  ...  plan-ceo-review-plan-mode
exit=0
```

## 浪潮证据

### PR 付费 lane 选择重放(CEO-23、ENG-18)

重放将每个记录的 PR 推送的更改文件通过今天的选择器和依赖图:

```bash
gh run list -R garrytan/gstack --workflow evals.yml --json databaseId,headSha,event,createdAt,conclusion -L 200 > runs.json
bun run scripts/replay-pr-selection.ts --runs runs.json
```

在 168 个具有计划清单的 PR 推送中(2026-09-25 至 2026-10-04):

| 措施 | 记录 | 重放 |
|---|---:|---:|
| 全回退推送 | 168 中的 165 (98.2%) | 168 中的 164 (97.6%) |
| 错过(依赖 diff 的失败用例且不再选择) | — | 0 |
| 不再存在的失败用例(单独计数) | — | 6 |

168 个 diff 中的 113 个超过比较 API 的 300 文件上限,并在其前 300 个文件上进行重放。该比率几乎没有变化,因为重放使用每个推送的累积分支 diff,并且该周的浪潮分支触及共享输入,正确选择每个用例:156 个推送中的 `scripts/`,125 个推送中的 `.github/`(主要是评估工作流),120 个推送中的 `bin/`,90 个推送中的 `make-pdf/`,86 个推送中的 `test/helpers/`,67 个推送中的 `design/`,58 个推送中的 `lib/`,56 个推送中的 `hosts/` 和 37 个推送中的 `tsconfig*.json`。选择器删除单类回退(持续时间种子、仅免费工作流、`scripts/ubicloud/**`、免费测试和免费 fixture;运行 36497566037 现在选择 `pr` profile)并且无法缩小更改 `bin/` 或 `lib/` 的分支;缩小这些需要 `bin/`、`lib/`、`hosts/` 和 `make-pdf/` 的触摸文件映射。合并后 `test:health` 检查测量普通 PR 上的回退率。

### 付费切片和并发(ENG-17)

使用 `EVALS_ALL=1` 进行实时普查(周期: `--list --slice-budget S --jobs 2`;gate 普查: `--emit-plan <file> --slice-budget S --jobs 2 --skip-judges`):

| Lane | 之前 (540 秒) | 之后 (420 秒) |
|---|---|---|
| 周期切片/峰值分片进程 | 26 / 52 | 33 / 66 (最大并行 26 → 36) |
| 每周 gate 普查切片/峰值 | 12 / 24 | 16 / 32 (最大并行 16 → 20) |
| CI 工作上限 | 每个切片的单个值:173 分钟周期,254 分钟 gate 普查 | 每个切片:普通 50 分钟(30 分钟分片墙 + 20),周期异常值 52 和 73 分钟,覆盖切片 173 分钟(5 个序列化的覆盖包装器);gate 普查 34–82 分钟 |

PR lane(`evals.yml`)使用相同的 420 秒预算和每个切片上限。

### 免费套件

- Windows lane 在六个 `windows-latest` 作业上运行,按 Windows 测量的持续时间打包:在免费套件分支的验证运行中约 4.6 分钟墙,从一台 runner 上的 8.1 分钟降低。
- 未播种的文件在第 99 百分位打包(以前是第 75 百分位),种子增长 ratchet(`test/free-seed-ratchet.test.ts`)将超过 60 秒的文件保留在仅缩小的允许列表上。
- bash+zsh 可移植性测试(#2669)现在在 CI 中运行,因为免费套件作业安装了 zsh。
- 原生 Windows 和 Dia 限定活动从 PR 工作流移至仅调度的 `native-qualification.yml`。

### 散文 pin(D2)

D2 已批准:[test-value-bar.md](test-value-bar.md) 中的 prompt-byte 合约仅涵盖机器读取的 token,并且 55 个免费测试文件中的句子 pin 通过 `test/helpers/prompt-structure.ts` 变为结构检查。这里的句子 pin 是 40 个或更多字符的英语字符串文字,或五个或更多单词的散文正则表达式,针对 SKILL.md、模板或 section 文本在免费测试中断言。

| 措施 | main (2db0b3a) | 此浪潮 |
|---|---:|---:|
| 句子 pin | 217 个文件中的 2,332 个 | 209 个文件中的 1,190 个 |
| 大写 pin(NEVER/MUST/ALWAYS/CRITICAL 强调) | 8 | 4 |

剩余的 4 个大写 pin 是枚举或标记 token(`PLAN MODE EXCEPTION — ALWAYS RUN`,`[CRITICAL]`,`ALWAYS-ON` CLI 输出),而不是强调。大多数剩余的句子 pin 位于 QA 调用者、检测器-helper 和工作流文件中,这些文件正在由正在进行的工作拥有。
