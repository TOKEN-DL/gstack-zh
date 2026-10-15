# AskUserQuestion 分割规则 — 完整参考

内联摘要存在于规范的 preamble(`scripts/resolvers/preamble/generate-ask-user-format.ts`)中。该子部分故意压缩,因为它注入到每个 tier-2+ skill 的 `SKILL.md` 中。当 N>4 选项出现并且你需要工作示例或完整的 Hold / 依赖关系 / 最终摘要语义时,此文件是内联指导指向的深度参考 — 按需加载。

## 这防止的错误

规则前的失败模式(从激励此问题的用户投诉中逐字摘录):

> "我达到了 Conductor 在 AUQ 中的 4 个选项限制,所以我需要削减一个。E4(detect-mappings codegen)是最大的提升,可能超出 v0.42 的范围 — 用户可以为 9 个集群手工编写其映射规则。我将删除它并保留 E1、E2、E3 和 E5..."

> "Conductor 上限为 4 个选项。修剪:E4(detect-mappings codegen)是最大的努力项目,是 v0.43+ 的自然后续 — 在不询问的情况下将其移动到 TODOS.md。使用 4 个重新触发。"

智能体单方面削减了没有用户输入的真实选项。选项集是用户的决策空间;默默地缩小它是错误。

## 哪种形状:批处理 vs 分割

两种合规形状。通过阅读选项来选择:

1. **批处理为 ≤4 组** — 选项是连贯的备选方案,将选择其中一个。示例:"major / minor / patch / micro"作为版本提升,"5 个布局变体,用户选择一个","哪个框架:rspec / minitest / cucumber / none"。将前 4 个批处理为一个 AskUserQuestion;如果前 4 个都不合适,则将第 5 个呈现为后续问题。这是适用的较低摩擦路径。

2. **按选项分割** — 选项是独立的范围项目,每个都带有自己的 include/defer/cut 决策。示例:"E1..E6,我们发运哪些?"、"Q3 的 5 个候选集成"、"审计发现的 8 个 TODO — 我们落地哪些?"。触发 N 个顺序的 AskUserQuestion 调用,每个选项一个。

**不确定时默认为按选项分割。** 将错误的选项批处理在一起 — 将正交的范围项目硬塞进一个问题 — 与删除是相同的失败模式。

## 按选项分割的机制

### 在链之前

检查选项之间的依赖关系。如果 E3 需要 E1,或 E5 与 E2 冲突,在每个选项的 ELI10 中显示该依赖关系:

> "削减这个会使 E3 孤立 — 它们是链接的。"

没有依赖关系显示,链会产生不连贯的选择集(用户为 E3 选择 Include + 为 E1 选择 Cut,发运不可构建的范围)。

### D 编号

- 父决策:`D<N>`,其中 N 是全局问题计数器。
- 每个按选项调用:`D<N>.k`,k=1..K 子级。
- 最终摘要:`D<N>.final`。
- 单选项修订:`D<N>.revise-<k>`。

父级 D3 的 5 选项示例链:

```
D3.1 → D3.2 → D3.3 → D3.4 → D3.5 → D3.final
```

### 按选项调用的形状

对于每个选项 Eₖ,触发一个 AskUserQuestion,带有:

- `D<N>.k` 标题(例如 D3.1、D3.2 ... D3.5)
- 仅此选项的范围、成本和其携带的任何依赖关系的 ELI10
- 推荐:Include / Defer / Cut,带有具体原因
- 每个选项 4 个桶:
  - **A) Include** 在此范围内(推荐/不推荐)
  - **B) Defer** 跟进(TODOS / 下一个版本)
  - **C) Cut** 完全
  - **D) Hold** — 停止链,讨论后再决定
- 注意:选项在种类上有所不同,而不是覆盖范围 — 没有完整性分数。(Include/Defer/Cut/Hold 是决策动作,因此现有格式规则适用:省略 `Completeness: N/10` 并改用种类注释。)

### Hold 意味着停止,而不是排队

当用户在任何按选项调用上选择 Hold 时,**立即停止链**。不要在 Hold 后面继续询问后面的选项 — 用户希望首先讨论所选选项。讨论后,用户通过说"continue"或命名要询问的下一个选项来恢复。

错误行为:在 E3 上将 Hold 在后面排队 E4 和 E5,然后使用陈旧的上下文触发它们。正确行为:停止,让用户重置父决策,从他们离开的地方恢复。

### 最终摘要

链解析后(无 Hold),触发 `D<N>.final` 以确认和验证组装集。

**步骤 1 — 验证依赖关系和容量。** 如果所选集不连贯(例如 E3 已包含但其必需的 E1 已 Cut),显示冲突和实际先前的答案。询问要重新访问哪个候选:

> "E3 需要 E1。你的答案包括 E3 并削减 E1。我们应该重新访问哪个?
> A) 重新访问 E1
> B) 重新访问 E3
> C) Hold — 将此冲突保持未解决以供讨论"

此路由答案不会更改任何处置。对于命名候选,以标准的 **Include / Defer / Cut / Hold** 菜单触发一个 `D<N>.revise-<k>`。保持所有其他候选的先前答案固定。回答后重新验证依赖关系和容量;永远不要默默地削减、交换或包含另一个候选以使集合适配。Hold 停止链:保留实际的先前答案并明确报告未解决的阻塞冲突。不要将不连贯的集合确认为准备好实施。

**步骤 2 — 确认组装集。** 如果连贯:

> "这是组装集:E1、E2、E5。发运此范围?
> A) 发运此范围(推荐)
> B) 修订一个选项(你选择哪个)
> C) 削减更多"

**步骤 3 — 有针对性的修订。** 如果用户选择 B,询问要修订哪个选项,然后在 `D<N>.revise-<k>` 触发一个按选项的 AskUserQuestion 以仅更新该选项。**不要**重新运行整个链。

## 大小规则

- **N ≤ 4**:使用正常的单个 AskUserQuestion 形式。不要分割。
- **N = 5 或 6**:分割(或批处理是否存在干净的分组)。
- **N > 6**:在链之前,在 `D<N>.0` 触发一个元 AskUserQuestion:

  > "即将询问 N 个按选项的问题。选项:
  > A) 继续完整分割(仅当每个选项独立时才推荐)
  > B) 首先缩小范围 — 我将提出一个较小的集
  > C) 改为将组批处理为 4 个"

  这本身是 AskUserQuestion 工具调用,而不是散文 — 它算作链中的第一个提示,而不是违反"工具而非散文"规则。

## 分割链的 question_id 规则

每个按选项的 AskUserQuestion 发出唯一的 `question_id`,形式为 `<skill>-split-<option-slug>`,其中 `<option-slug>` 是选项键 kebab-cased(小写、连字符、仅 ASCII)。

示例:
- `plan-ceo-review-split-e4-detect-mappings`
- `ship-split-rspec`
- `plan-eng-review-split-add-coverage-test`

**冲突处理。** 如果两个选项会产生相同的 slug,则后缀为 `-2`、`-3` 等。

**长度。** 总长度必须 ≤64 个字符(由 `bin/gstack-question-preference --write` 验证)。如有必要,截断选项 slug,保留 `<skill>-split-` 前缀。

## AUTO_DECIDE 与分割链的行为

双层防御。

**第 1 层 — 机制。** 每个按选项的 `question_id` 对于其选项是唯一的,因此在一个选项的 id 上设置的偏好不能跨链泄漏。`ship-split-rspec` 上的 `never-ask` 不会默默地批准 `ship-split-minitest`。

**第 2 层 — 运行时强制。** `bin/gstack-question-preference --check` 检测匹配 `*-split-*` 的任何 id(分割链发出的规范 slug 模式),并强制 `ASK_NORMALLY`,即使存在该确切 id 的 `never-ask` 或 `ask-only-for-one-way` 偏好。当此覆盖触发时,检查会发出解释性注释:

> "split-chain per-option calls always ASK_NORMALLY; your never-ask preference does not apply to options inside a sequential split."

**结果。** 分割链的按选项调用永远不符合 AUTO_DECIDE 资格。这是运行时合约,不仅仅是按 id 唯一性的冲突阻力。用户的选项集是神圣的 — 恢复用户对决策空间的主权是分割的全部意义。

## 与每个 skill 规则的交互

此规则**覆盖任何每 skill 的"批处理决策"指导**。明确要求一次一个问题调用的每 skill 模板(例如 `plan-eng-review`)已经兼容 — 它们是该规则的更严格的特殊情况。

## 工作示例:5 个平台集成

由 `test/skill-e2e-plan-ceo-split-overflow.test.ts` 使用的 fixture。一个计划有 5 个独立的聊天平台候选:

- E1) Slack DM bot(约 2 周,约 40% 的询问)
- E2) Discord guild bot(约 3 周,约 15%)
- E3) Microsoft Teams(约 4 周,约 5%)
- E4) Telegram(约 1 周,约 8%)
- E5) Mattermost(约 2 周,约 3%)

用户希望每个候选单独决策,而不是捆绑选择。智能体应该:

1. 识别这是 5 个选项的独立范围决策 → 分割。
2. 检查依赖关系(此处没有 — 每个平台都是独立的)。
3. 通过 `D3.1` 到 `D3.5` 触发,每个平台一个,带有 Include / Defer / Cut / Hold 桶以及每个选项的努力 + 需求支持的推荐。
4. 链之后,触发 `D3.final` 总结组装范围(例如"发运 E1 + E4 — Slack 和 Telegram 以最少的构建成本吸引最多的需求。延迟其余。A) 发运 / B) 修订 / C) 削减更多")。

修复前的失败形状(错误):智能体构造一个具有 E1..E4 作为四个选项的单个 AskUserQuestion,使用散文如"E5 是最小的收入细分,移动到 TODOS"删除 E5。用户从未有机会权衡 E5。E2E 测试中的 4 之底捕获了这一点。
