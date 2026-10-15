# 模型策略

使用前沿模型挑战计划,然后使用智能模型来构建它。这些是精心策划的建议,而不是声称一个模型在每个任务中都获胜。用户选择优先,建议永远不会切换正在运行的会话或编辑主机的原生配置。

| 层级(Tier) | 默认工作 | Anthropic | OpenAI |
| --- | --- | --- | --- |
| `frontier`(前沿) | 独立计划评审 | `claude-fable-5-1` | `gpt-6-astra` |
| `smart`(智能) | 实现推荐 | `claude-opus-5-5` | `gpt-6.1-sol` |

## 检查和更改你的策略

本快速入门假定 gstack 已安装。它不使用网络、凭据或付费模型调用。从项目仓库根目录在 Bash、Git Bash 或 MSYS 中运行:自动化的外部评审使用该目录。从子目录的检查反映该子目录的原生设置,可能会有所不同。如果你的安装是 vendored(供应商式)或其他位置,请将 `GSTACK_ROOT` 设置为该安装的 gstack 运行时。

```bash
if [ -x "$HOME/.claude/skills/gstack/bin/gstack-models" ]; then
  GSTACK_ROOT="$HOME/.claude/skills/gstack"
elif [ -x "${CODEX_HOME:-$HOME/.codex}/skills/gstack/bin/gstack-models" ]; then
  GSTACK_ROOT="${CODEX_HOME:-$HOME/.codex}/skills/gstack"
else
  printf '%s\n' 'Upgrade the gstack install for your host, then rerun this quickstart.' >&2
  exit 1
fi
"$GSTACK_ROOT/bin/gstack-models" list
"$GSTACK_ROOT/bin/gstack-models" resolve --role plan-review --provider openai
```

输出解释所选 model 和获胜的设置。它还显示目录验证元数据。离线选择**不**证明 CLI 已安装、你的账户可以使用该 model 或供应商的当前目录已检查。

选择较便宜的评审层,然后检查有效结果:

```bash
"$GSTACK_ROOT/bin/gstack-config" set plan_review_tier smart
"$GSTACK_ROOT/bin/gstack-models" resolve --role plan-review --provider openai
```

固定 provider 的 tier 内的 model,或将 pin 重置为出厂默认:

```bash
"$GSTACK_ROOT/bin/gstack-config" set model_frontier_openai gpt-6-astra
"$GSTACK_ROOT/bin/gstack-config" set model_frontier_claude claude-fable-5-1
"$GSTACK_ROOT/bin/gstack-config" unset model_frontier_openai
"$GSTACK_ROOT/bin/gstack-config" unset plan_review_tier
```

要继续将原生 CLI model 设置用于计划评审:

```bash
"$GSTACK_ROOT/bin/gstack-config" set plan_review_tier host
```

设置在下次调用时应用,无需重新渲染 skill。已经在检查的调用会通过执行保留其选定的 model。配置使用现有的 [gstack state root](state-root.md),而不是新的每个仓库的设置文件。无效写入被拒绝并保留先前的值;使用 `unset` 而不是空值来恢复默认值。

格式错误的已知设置(例如 `plan_review_tier = smart`)停止选择而不是充当缺失的键。纠正或删除报告的配置文件中命名的行;`unset` 仅删除拼写正确的 `key:` 记录。

Codex 计划评审就绪状态和派发共享一个截止日期。就绪状态预算占剩余时间的一半,带现有的终止宽限。缓慢的预检不会延长调用者的时间限制,而是为评审留下更少的时间。过期的截止日期不会启动新命令,意味着缺少的评审覆盖范围,而不是已完成的评审。

## 设置和优先级

| 键(Key) | 默认 | 接受的值 |
| --- | --- | --- |
| `plan_review_tier` | `frontier` | `frontier`、`smart`、`host` |
| `implementation_tier` | `smart` | `frontier`、`smart` |
| `model_frontier_claude` | 目录的 Anthropic 前沿 model | 显式 Anthropic model 标识符 |
| `model_frontier_openai` | 目录的 OpenAI 前沿 model | 显式 OpenAI model 标识符 |
| `model_smart_claude` | 目录的 Anthropic 智能 model | 显式 Anthropic model 标识符 |
| `model_smart_openai` | 目录的 OpenAI 智能 model | 显式 OpenAI model 标识符 |

对于计划评审,为该调用命名的 model 胜过 `GSTACK_CLAUDE_MODEL` 或 `GSTACK_CODEX_MODEL`。该环境覆盖胜过所选 tier 的配置 model,而后者胜过出厂目录。通用编码 model 设置不会默默地选择更便宜的计划评审员;当这是你想要的行为时选择 `host` 模式。

在 `host` 模式下,Codex 保留其现有的原生配置解析,包括 `CODEX_HOME`、活动的 profile 和 `exec` 与原生 `review` 的区别。Claude 可以将其 model 选择委托给其 CLI 而不强制使用 `--model`;检查器报告该委托而不是发明确切的 model ID。实际返回的 model 使用情况与请求的 model 分开。

实现推荐使用明确请求该工作的 model,然后是配置的实现 tier 及其 provider 覆盖,然后是目录。外部评审环境覆盖不会更改实现推荐。已知的 Claude Code 和 Codex 主机推荐自己的 provider;未知或多 provider 的主机可以显示两种选择。

自定义 endpoint 和合作伙伴平台需要显式兼容的 model 或 `host` 模式。Gstack 不会猜测公共 API ID 是否适用于 Bedrock、Vertex、Foundry 或自定义 Codex provider。通过语法验证的 model 标识符不是账户可用性的证明。参见 [provider recovery](troubleshooting.md#model-policy-provider)。

检测是保守的,而不是 CLI 设置引擎的替代品。Claude 检查环境、用户设置、项目/共享/本地设置、仓库和工作树主 checkout 的本地设置,以及本地管理的文件。Codex 检查环境、用户/活动 profile 设置和系统配置;不检查项目本地的 Codex 配置。任何检测到的本地自定义路由信号都需要显式 model,即使原生信任或优先级规则可能会抑制它。服务器管理或 MDM 策略以及额外的原生 CLI 标志对此检查不可见;`--help` 和 JSON 输出列出了这些限制。为这些情况使用显式兼容的 model,或对计划评审使用 `host` 模式。实现推荐需要显式的实现 model,而不是委托给隐藏的原生选择。

## 推理努力

Codex 调用带有 `model_reasoning_effort`。它的形式与 model 相同:对 `/codex` 的每个请求标志、每个运行的环境覆盖、然后是存储的设置,然后是每个 gate 出厂附带的默认值。

| 顺序 | 来源 | 设置方式 |
| --- | --- | --- |
| 0 | `/codex ... --effort <level>`(仅此请求;仅 `/codex`) | `/codex review --effort medium` |
| 1 | `GSTACK_CODEX_EFFORT`(此 shell,此运行) | `export GSTACK_CODEX_EFFORT=max` |
| 2 | config.yaml 中的 `codex_effort` | `gstack-config set codex_effort low`(`unset` 恢复默认值) |
| 3 | gate 的默认值 | 无:评审、计划评审和 `/codex` review 和 challenge 为 `high`;`/codex` consult 和 design direction 为 `medium` |

级别:`minimal`、`low`、`medium`、`high`、`xhigh`、`max`、`ultra`。支持取决于 model,因此 model 不支持的级别会在请求时通过通常的 model 错误失败。无效值永远不会到达 Codex:`gstack-config set` 拒绝它,并且 `gstack-codex-probe run-with-timeout --effort <default>`(解析上面的顺序并附加标志)在发送任何内容之前停止调用,并命名要修复的来源。`gstack-codex-probe check-effort <default>` 打印相同的解析而不调用。

对于一个请求,`/codex review --effort medium`(或 `--effort=<level>`;`--xhigh` 是 `--effort xhigh` 的简写)胜过其他三个,包括导出的 `GSTACK_CODEX_EFFORT`,并且不会更改任何持久的内容。该标志仅在 `/codex` 或其 mode 词之后,在前导标志(`-m`/`--model`、`--role`,任意顺序)之间以及提示文本之前计数;后面的 `--effort` 是提示的一部分,`/codex` 会这样说。Consult 跟进会重新解析其努力。`/codex` 将该级别作为 `--request-effort` 传递给探针,因此未知级别会在任何 Codex 调用之前停止,并命名该标志。

`/spec` 的质量 gate 在其 120 秒预算内保持 `medium`,无论你设置什么,因为更高的努力会花费 gate 所设计的时间。`gstack-models list` 打印有效的努力及其来源(`--json`:`codexEffort`)。Claude Code runner 不采用努力设置。

## 哪些工作流使用这些层级?

计划评审角色适用于 autoplan 的 CEO、design、DX 和 engineering 外部声音;独立的 CEO、engineering、developer-experience 和 design 计划评审;以及 spec 质量 gate。现有的推理努力、访问限制、超时、评审 gate 和 `codex_reviews` 同意仍然有效。

主机独立于 tier 选择对立的 provider。Claude Code 使用 Codex 包装器;Codex 使用 Claude Code 包装器。Model 的名称不能确定哪个 harness 正在运行。

手动对立 model 包装器可以选择加入 `--role plan-review`。普通的代码评审、实时设计评审、设计咨询、office-hours 第二意见和文档评审保留其现有的 model 选择行为。历史无角色默认值、评估 actor/judge、廉价的辅助 model 和安装时的行为叠加不遵循此目录。

升级后第一次受影响的付费计划评审之前,gstack 解释独立的前沿默认值和获胜的来源,以及 `smart` 和 `host` 替代方案。检查不会将该通知标记为已见。禁用的外部评审保持禁用;不可用的 model 会被报告,永远不会默默替换。

## 第一次完成的外部评审

策略设置和已完成的评审是不同的里程碑。在调用付费评审之前,使用 [host setup instructions](../README.md#install--about-30-seconds) 安装并验证对立的 CLI。从 Claude Code,使用 `/codex consult --role plan-review`。从 Codex,使用 `/claude-code consult --role plan-review`(或其已安装的命名空间名称)。要求它挑战计划并提供完整的计划文本。Consult 是提示驱动的,在有仓库 diff 之前就可以工作;Challenge 模式用于评审代码更改。

确认宣布的选择、provider 的实际结果和覆盖状态。已解析的 model、成功的可用性探针或原生回退不是已完成的外部评审。无法使用的显式选择需要在错误中命名的来源进行修复;更改较低优先级的 tier 无法覆盖环境 pin。参见 [selection recovery](troubleshooting.md#model-policy-selection)。

## 机器可读的检查

不带参数的 `gstack-models` 是 `list`。`list` 和 `resolve` 都接受 `--json`。计划评审解析需要 `--provider anthropic|openai`;实现解析可以省略 provider 以返回两个推荐。

```bash
"$GSTACK_ROOT/bin/gstack-models" resolve --role plan-review --provider anthropic --json
"$GSTACK_ROOT/bin/gstack-models" resolve --role implementation --json
```

JSON 输出是一个文档,带有 `schemaVersion: 1`、`selections` 数组以及失败时的结构化错误。人工诊断转到 stderr。退出 `0` 表示有效解析,包括 host delegation;`1` 表示配置或选择失败;`2` 表示无效用法。检查和解析都不会写入通知标记、探测 model 或获取供应商来源。

## 保持出厂建议为最新

模型策略新鲜度工作流每周运行一次并支持手动派发。它检查官方的 Anthropic 和 OpenAI 建议和弃用源,无需 provider 凭据或付费模型调用。普通 PR 检查使用离线 fixture;供应商中断不会使不相关的 PR 失败。

在 GitHub Actions 中,选择 **Model-policy freshness** 以手动运行它。从 gstack checkout,`bun scripts/model-policy-freshness.ts` 执行相同的公共源检查而不更改问题;与 `gstack-models` 不同,它使用网络。工作流的受保护的 `--publish` 路径拥有问题更新。其 JSON 报告和作业摘要保留源证据。

报告的主要状态区分 `current`、`update-candidate` 和 `unknown/source-unavailable`。其单独的 `evidenceStatus` 在 30 天后将保留的成功证据标记为 `stale`;即使该保留证据过时,失败的获取在标题中仍然是 unknown。较新或新推荐的 model 是候选,不是自动替换。"不早于"之类的支持承诺不是计划的弃用,而是偶然的 model 提及也不是新的默认。此证据涵盖公共 API 建议,而不是合作伙伴平台资格。

一个 Actions-bot 拥有的跟踪问题保留成功检查的目录/来源/解析器身份以及任何未解决的生命周期发现。当问题关闭时,健康状态仍然可读。失败或部分检查无法擦除已知的弃用、从较旧的目录借用成功,或者假装其证据是新鲜的。目录回滚会重新打开历史发现,以便重新使用 model。仅当前默认分支运行可以更新该问题;用户拥有的问题中的复制标记不是受信任的状态。

有据可查的 retain 决定保留当前默认值,但不会使供应商不匹配 `current`:在候选保留期间,咨询问题保持打开或重新打开。在问题评论中记录 retain 基本原理,工作流永远不会编辑这些评论,而不是与计划的问题正文更新竞争。

30 天的 staleness 意味着缺少值得信赖的新鲜度证据,而不是 model 在 30 天前发布。未更改的建议不需要仅日期的目录提交或每月付费基准。记录的下一个检查到期日期有助于维护,但从不运行的工作流无法自行发出警报。

### 维护者更新清单

1. 检查最新的新鲜度运行及其到期日期。在声明目录为 current 之前,解决缺失、损坏或来源不可用的证据。
2. 验证官方建议和生命周期证据。在一个工作日内分类已宣布的弃用/移除,在七天内分类陈旧/不可用的证据,在三十天内分类自由裁量的更新候选。
3. 打开带有源 URL、确切 model ID 和限定证据的目录更改,或记录为什么应保留当前默认值。永远不要编辑用户的个人 pin 或默默替换另一个 provider。
4. 确认 CLI model/profile 支持和 tool/reasoning 兼容性。保持固定的测试运行时和基准标尺稳定。所需的 CLI 升级是单独评审的更改,而不是发现的自动副作用。
5. 使用现有的评估机制、固定的 prompt/panel 和不变的阈值预声明任何预算的基线/候选比较。保持每次尝试、衡量有用的和不受支持的发现、延迟和可用使用,并且不要将供应商营销或创建时间戳视为质量证据。
6. 通过受影响的免费检查和选定的工作流评估,评审更改,并通过正常的发布流程发布它。只有未固定的承担角色的默认值遵循新的出厂目录。

该工作流不会打开自动的 model 升级 PR、合并更改、切换活动会话、更改基准 model 或安排付费评估。
