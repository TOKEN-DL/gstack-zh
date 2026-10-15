# 排查 gstack 消息

找到你看到的消息,然后按照其修复操作。gstack 运行的每个检查都结束于三种状态之一:

- **ran(已运行)**:检查执行了。它可能仍然发现了问题。
- **not run(未运行)**:gstack 选择不运行它(你关闭了它,或者没有适用项)。
- **unavailable(不可用)**:gstack 尝试了但无法运行它。这是缺失的覆盖范围,永远不是通过。

不是 `ran` 的消息具有以下形状:

```
<check>: not run (<reason>). Fix: <command>
<check> unavailable: <reason> (<detail>). No review ran; this is missing coverage, not a pass. Fix: <command>
<check>: ran, verdict unverified (<reason>). Fix: read the output above
```

在本页面中搜索 `unavailable:` 或 `not run (` 后面的词。每个部分都有一个稳定的链接锚点;原因代码和锚点来自 `lib/gate-outcomes.ts`,如果有代码没有相应的部分,自由测试会失败。

<a id="gstack-doctor"></a>
### 在 skill 运行前检查就绪状态

运行 `~/.claude/skills/gstack/bin/gstack-doctor`(其他主机:在你的 gstack checkout 中运行 `./setup --status` 打印医生的绝对路径)。它为安装、state root、Bun、hooks、Codex、缓存的 Codex 模型检查、artifacts 同步、browse bundle、其他编译的二进制文件、`/cso` native helper、Claude Code、最大会话日志和最近的 `/autoplan` guard 代码各打印一行。每一行都是 `ok`、`warn`、`not configured` 或 `fail` 以及其修复;只有 `fail` 以非零退出。它不进行付费调用:Codex 行报告缓存的模型检查及其年龄,`--live` 运行该检查一次。将其输出粘贴到 bug 报告中。

---

## 模型策略

[model-policy guide](model-policy.md) 解释了两个层级、全部六个设置、优先级和只读检查命令。已解析的选择并不证明所请求的 provider 或 model 能够运行。

<a id="model-policy-config"></a>
### 模型策略配置无效或不可读

**含义。** 新策略路径无法读取连贯的配置或配置的值无效。不可读的文件不会被当作缺失的设置来处理:那样做可能会默默地选择高级默认。

**修复。** 修复错误命名的文件或键。`plan_review_tier` 接受 `frontier`、`smart` 或 `host`;`implementation_tier` 接受 `frontier` 或 `smart`。使用 `gstack-config unset <key>` 恢复默认值,而不是空值。被拒绝的 `set` 会保留旧值。纠正所有权/读取权限,而不是删除不相关的状态目录。对于格式错误的行,例如 `plan_review_tier = smart`,编辑报告的文件并首先纠正或删除该行:`unset` 只能删除规范的 `key:` 记录,无法修复拼错的分隔符或带引号的键。

**预期结果。** `gstack-models list` 或相应的 `resolve` 命令解释获胜的来源。这次错误上没有运行外部模型调用。

<a id="model-policy-provider"></a>
### 自定义 provider 需要显式 model 或 host mode

**含义。** 原生设置或环境标志将评审器路由到自定义 endpoint 或合作伙伴平台。公共 API 目录标识符不一定是那里的有效部署标识符。格式错误的原生 provider 配置也会阻止 gstack 假定公共 endpoint。

**修复。** 为该请求提供兼容的 model,设置 provider 的 gstack 环境覆盖或 tier-model 设置,或选择 `gstack-config set plan_review_tier host`。将凭据保留在 provider 的正常身份验证机制中,而不是模型设置中。如果错误命名了格式错误的原生配置,请先修复该来源。

**预期结果。** 检查报告明确的选择或诚实的 host-managed delegation。它不证明账户资格,也不将公共默认发送到推断的自定义 endpoint。

<a id="model-policy-selection"></a>
### 所选评审 model 无法使用

**含义。** 请求的 model 被拒绝,或者 provider 无法完成评审。Model 可能不存在、账户不可用、已停用或位于配置错误的 endpoint 后;保留 provider 的证据而不是将所有这些视为同一诊断。没有暗示成功的外部评审,gstack 不会替换不同的 model。

**修复。** 更改错误中命名的来源。例如,环境覆盖胜过 tier 设置,因此编辑 tier 无法替换它。取消设置或更正该覆盖,或显式为请求命名支持的 model。当报告的失败是现有的 auth、配额、沙箱或超时修复时,遵循它。缺失的使用身份仍然是未知的,即使检查显示了请求的 ID。

**预期结果。** 下一次调用宣布其 model 和来源,检查同一选择并报告实际的 provider 结果。缓存的探针或原生回退不是已完成的外部评审。

<a id="model-policy-freshness"></a>
### 模型目录新鲜度未知、过时或需要更新

**含义。** 官方证据已更改,无法检查所需的来源,或保留的证据与此目录和解析器不匹配。`unknown/source-unavailable` 不是当前的。稍后的源失败不得擦除较早的停用发现或关闭其维护问题。

**修复。** 阅读新鲜度工作流报告和跟踪问题中失败的源和证据详细信息。必要时修复源解析器或发布访问权限,然后针对默认分支手动运行工作流。如果 model 已被取代或弃用,请遵循 [maintainer checklist](model-policy.md#maintainer-update-checklist) 来限定替代项或记录支持的缓解措施。仅因为尚未采用新版本而清除该问题。

**预期结果。** 完整的源检查产生绑定到当前目录的证据,而未解决的生命周期发现仍然可见。个人 pin 和不相关的 PR 保持不变。还要检查工作流的最后运行日期:从未运行的计划作业无法报告其自身的缺失。

#### 修复 `recoveryRequired` 状态

此标志有意保持问题打开,直到维护者修复记录。仅另一个成功的获取无法证明损坏的历史。

1. 保存当前问题正文,包括其保留的恢复存档,并下载最新可信的工作流报告。使用其 `report.state` 作为起始状态;不要发明运行 ID、目录哈希或检查日期。
2. 将其 `lifecycle` 数组与问题的独立生命周期块和任何保留的存档进行协调。保留已知的停用、截止日期和人工处置。当前在目录中的 model 不得保留以前替换中的旧 `resolvedByCatalog` 标志。
3. 在问题编辑器中,在 `state:begin` 和 `state:end` 标记行之间恢复有效的 JSON。将同一生命周期数组放在 `lifecycle:begin` 和 `lifecycle:end` 之间。恢复独立的所有权和区域标记,并将人工文本保留在拥有的区域之外。
4. 仅在该协调之后,将 `recoveryRequired` 设置为 `false`。如果无法验证先前的成功证据,请将 `lastSuccess` 设置为 `null`,而不是编造新的成功。如果无法恢复生命周期历史,请保持该标志设置并将问题保持打开以进行调查。
5. 针对默认分支运行 **Model-policy freshness**。只有完整的、当前的证据且没有未解决的发现才能关闭该问题。解析或源错误保持其打开;不要清除历史以制造通过。

## 外部评审(Codex 和 Claude Code)

外部评审将你的 diff、计划或问题发送给第二个 AI provider。他们的 verdict(裁决)来自 `lib/outside-review-result.ts`,它打印 `VERDICT: clean|findings|unverified|unavailable`。具有 P0 或 P1 发现的 `findings` verdict 像原生 P0/P1 一样阻塞。`unverified` 和 `unavailable` 是缺失的覆盖范围:/ship 和 /review 继续,在就绪仪表板和 PR 正文中显示该差距,并且从不将其计为通过。

<a id="outside-review-verdict"></a>
### 如何从评审中读取 verdict

- **严重性标签。** `[P0]`-`[P3]`、Codex 的原生 `P1:` 标签,以及行首在可选标题、项目符号、数字或粗体后的 P0-P3 标签:`**P1 — x**`、`**P1** — x`、`1. P2 — x`、`P2 —`,或 `### P1`,或表格单元格 `| P1 |`。P0 和 P1 阻塞;P2 和 P3 是咨询性的。在该标签位置中 gstack 无法读取其形式的 P0-P3 token(`## P1/P2 findings`)使评审 [`unverified`](#outside-review-unparsed-severity),永远不是 `clean`。代码围栏和 inline code 是引号,而不是标签。
- **严重性词在标签位置。** `Severity: High`、`Priority: low`、以 `High:`、`High —`、`[High]` 或 `**High**` 开头的行(在可选的标题、项目符号或数字之后)、任何位置的粗体 `**High**`,或表格单元格 `| high |`。Critical 和 High 像 P0 和 P1 一样阻塞;Medium 和 Low 像 P2 和 P3 一样是咨询性的。散文中的词不算:"high-level"、"low-risk"、"a medium-term follow-up" 和 "no critical findings" 都不是发现。
- **每个评审 gate。** 这些规则适用于总结评审的每个 gate,包括 `/codex` consult 和 challenge 后面的 `execution` gate。其最后一个 `Verdict:` 行表示 `not ready` 但没有可读严重性的回复会打印 `FINDINGS: unspecified` 并像阻塞发现一样退出 3。
- **没有发现。** 没有标签也没有标签的评审是 `clean`,只有当它明确说明(`NO_FINDINGS`、`"no issues"`、`"did not find any bugs"`)。否则它是 `unverified`(见下文),永远不是 `clean`。完全没有严重性标签的 `/codex` consult 答案不是评审摘要,保持 `clean`。
- **设计提案** (/design-consultation 和 /office-hours 中的设计方向声音) 使用 `proposal` gate 读取:完成的提案只需要其 `Recommendation: ... because ...` 行。

外部评审提示要求评审员将每个发现标记为 Critical、High、Medium 或 Low,因此大多数评审以 `findings` 或 `clean` 结束。

<a id="sourced-helper-location"></a>
### `gstack: cannot load gstack-codex-probe` / `gstack: cannot locate <helper> (shell: ...)` / `CODEX_MODE: helper_unavailable`

**含义。** Skills 以命令方式运行 `gstack-codex-probe`,每个检查一个子命令,因此你的 agent 使用的 shell 无关紧要。`cannot load gstack-codex-probe` 和 `CODEX_MODE: helper_unavailable` 意味着你的安装中探针文件缺失或不可执行。

`cannot locate <helper> (shell: ...)` 来自仍使用 `source` 加载到你的 shell 的 helper(`gstack-egress-lib.sh`,以及在升级之前渲染的 Codex 探针 skill)。这样的 helper 从 bash(`BASH_SOURCE`)或 zsh(`%x`)中找到自己的目录。在任何其他 shell(dash、sh)中,或者当 shell 无法说出它正在读取哪个文件时,它会停止而不是猜测路径。该消息命名它看到的 shell。

**修复。** 对于 `cannot load`,从你的 gstack checkout 重新运行 `./setup`(或 `/gstack-upgrade`)。对于 `cannot locate`,从 bash 或 zsh 运行该 skill(macOS 和 Linux 的默认值),或者告诉 helper gstack 安装在哪里:

```bash
export GSTACK_ROOT=~/.claude/skills/gstack   # your install dir; it holds bin/
```

如果你看到 `gstack: sourcing gstack-codex-probe is deprecated ...`,升级前渲染的 skill 仍在 source 该探针。它会一直工作,直到 2026-10-21 或之后的版本;运行 `/gstack-upgrade` 以立即重新渲染你的 skill。

**预期结果。** `~/.claude/skills/gstack/bin/gstack-codex-probe select-model exec` 从任何 shell 打印 `CODEX_SEL: <model>`(以及 stderr 上的 `CODEX_MODEL: <model> (exec; source: ...)`),preflight 打印除 `helper_unavailable` 之外的 `CODEX_MODE`。`gstack-codex-probe help` 列出每个子命令。

<a id="codex-sandbox-unavailable"></a>
### `Codex outside review unavailable: Codex's sandbox could not start here (...)`

**含义。** Codex 在 Linux 沙箱(bubblewrap)中运行每个命令。在许多容器和 devcontainer 中,内核不允许非特权用户命名空间,因此沙箱无法启动,Codex 尝试的每个命令都失败。gstack 报告这一点,而不是信任一个什么都没读的评审。`CODEX_MODE: sandbox_unavailable` 是免费 preflight 在任何付费调用之前发现的相同条件。

**保留的内容。** 没有任何内容被发送进行评审,或者评审被丢弃。你的代码和文件保持不变。

**修复。** 为容器启用非特权用户命名空间(对于 Docker,允许它们的 seccomp 配置文件),或者在你信任的容器中,在此 shell 中运行 Codex 而不使用其沙箱:

```bash
export GSTACK_CODEX_NO_SANDBOX=1
```

**预期结果。** 启用命名空间后,下一次评审打印 `OUTSIDE_STATUS: completed`。使用 `GSTACK_CODEX_NO_SANDBOX=1`,每次评审都会打印 `WARNING: GSTACK_CODEX_NO_SANDBOX=1: ...`,因为 Codex 然后可以读写你的用户可以访问的任何内容。只有确切的值 `1` 起作用,并且仅从你的 shell 环境起作用。

<a id="outside-review-commands-failed"></a>
### `... outside review unavailable: the reviewer could not run commands or read the diff (...)`

**含义。** 评审员尝试运行命令但没有成功,或者其答案说它无法运行命令或读取 diff。之后写入的"未发现问题"不是评审。

**保留的内容。** 没有任何内容;答案显示在上面但不计入。

**修复。** 阅读上面评审员的 stderr,修复其命名的内容,然后重新运行。

**预期结果。** 带有 `VERDICT:` 行的 `OUTSIDE_STATUS: completed`。

<a id="outside-review-execution-failed"></a>
### `... outside review unavailable: the reviewer process failed (exit N: ...)`

**含义。** provider CLI 以非零退出。第一个 stderr 行在括号中。

**保留的内容。** 部分输出显示在上面;未计入。

**修复。** 修复 provider 诊断中的原因(再次登录,修复 model,检查网络),然后重新运行。

**预期结果。** `OUTSIDE_STATUS: completed`。

<a id="outside-review-timeout"></a>
### `... outside review unavailable: the reviewer hit its time limit and was stopped (exit 124)`

**含义。** provider 在其截止日期前未完成并被停止。

**保留的内容。** 部分输出显示在上面;未计入。

**修复。** 以较小的范围重新运行,或检查 provider 的状态和你的网络。

**预期结果。** `OUTSIDE_STATUS: completed`。

<a id="outside-review-empty-response"></a>
### `... outside review unavailable: the reviewer returned no response`

**含义。** provider 成功退出但未返回任何内容。

**修复。** 阅读上面的 stderr,然后重新运行。

**预期结果。** `OUTSIDE_STATUS: completed`。

<a id="outside-review-refused"></a>
### `... outside review unavailable: the reviewer declined to review`

**含义。** provider 以拒绝而不是评审进行回答。

**修复。** 重新运行;如果再次拒绝,请依赖原生评审。

**预期结果。** `OUTSIDE_STATUS: completed`。

<a id="outside-review-missing-markers"></a>
### `... outside review unavailable: the response lacks the markers this gate requires (...)`

**含义。** 答案不包含 gate 检查的内容:`Recommendation: ... because ...` 行、`SCORE:`/`AMBIGUITIES:` 对或严重性标签。

**修复。** 重新运行评审。

**预期结果。** `OUTSIDE_STATUS: completed`。

<a id="outside-review-unverified"></a>
### `... outside review: ran, verdict unverified (...)` / `OUTSIDE_STATUS: unverified` / `GATE: UNVERIFIED`

**含义。** 评审完成但没有严重性标签或标签,并且没有明确的未发现结论,因此无法从中读取通过或失败([如何读取 verdict](#outside-review-verdict))。

**保留的内容。** 完整答案显示在上面。

**修复。** 阅读上面的输出并决定。它不是通过,/ship 和 /review 将其列为缺失的覆盖范围。

<a id="outside-review-unparsed-severity"></a>
### `... outside review: ran, verdict unverified (the review labels a finding with a P0-P3 severity in a form the parser cannot read (line N: "..."))` / `REASON: unparsed_severity`

**含义。** 回复以 P0-P3 token 开始一行,该位置是发现标签的位置,但形式 gstack 不读取为单一严重性,例如 `## P1/P2 findings` 或 `- P2/P3 nits`。gstack 不会猜测哪个严重性适用,因此 verdict 是 unverified 而不是 `clean`。括号中的引用行(也打印为 `UNPARSED:`)是第一个这样的标签。当同一回复也有可读的 P0 或 P1 发现时,该发现会阻塞(`VERDICT: findings`,退出 3),未解析的行仍会打印。

**保留的内容。** 完整回复显示在上面。

**修复。** 阅读引用行和回复,并决定;它不是通过。重新运行评审,并要求评审员将每个发现标记为 `[P1] <title>` 或 `P1: <title>`。要重新检查已保存或更正后的回复,请从你的 gstack checkout 运行:

```bash
bun lib/outside-review-result.ts --verdict execution reply.md
```

(对那些 gate 使用 `review` 或 `structured`)。

**预期结果。** 带有 P0-P3 `FINDINGS:` 值的 `VERDICT: findings` 或 `VERDICT: clean`,并且没有 `UNPARSED:` 行。

<a id="codex-model-unusable"></a>
### `CODEX_MODE: model_unusable` / `MODEL_UNUSABLE`

**含义。** Codex 拒绝了所选 model:账户无法使用它(HTTP 400),model 已停用,或自定义 provider 的 `base_url` 错误(HTTP 404)。`CODEX_MODEL:` 行命名 model 及其来源。

**修复。** 选择你的账户可以使用的 model,或更正 provider:

```bash
export GSTACK_CODEX_MODEL=<supported-model>
# or edit model / base_url in ${CODEX_HOME:-~/.codex}/config.toml
```

**预期结果。** `CODEX_MODE: ready`。

<a id="codex-quota-exhausted"></a>
### `CODEX_MODE: quota_exhausted` / `MODEL_QUOTA_EXHAUSTED`

**含义。** Codex 拒绝调用,因为它后面的账户已达到其使用限制(`You've hit your usage limit`,或 `insufficient_quota`)。标记下的行是 Codex 自己的消息,带有其重置时间和购买更多的地方。Model 选择很好。gstack 将外部覆盖报告为不可用,而不是通过,并缓存结果 15 分钟,因此运行的其余部分(和其他 skill)不进行 Codex 调用。HINT 行说明还剩多少分钟。

**修复。** 等待 Codex 消息中的重置时间,或为该账户添加信用或更高级别的计划。要使用其他账户,请重新登录:

```bash
codex login
```

要在 gstack 的 15 分钟缓存到期之前重新检查(例如,购买信用后立即),在一次检查中跳过缓存结果,或删除它:

```bash
export GSTACK_CODEX_PROBE_RETRY=1   # unset it again afterwards
# or
rm -f ~/.gstack/.codex-model-probe  # <state root>/.codex-model-probe
```

**预期结果。** 重置后(或 `codex login` 之后,它会更改身份验证签名并立即重新探针),`CODEX_MODE: ready`。

<a id="codex-rate-limited"></a>
### `CODEX_MODE: unverified (rate_limited)` / `MODEL_PROBE_RATE_LIMITED` / `unavailable: Codex rate-limited the review`

**含义。** Codex 回答 HTTP 429(请求过多),通常会在几秒钟内清除。它是与 `quota_exhausted` 不同的状态,永远不会被缓存。在探针时评审仍然运行,其自己的结果决定。评审期间的 429 意味着该评审失败,因此覆盖范围缺失,永远不是通过。

**修复。** 一分钟后重新运行评审。如果 429 持续存在,请检查 provider 仪表板上账户或 API 密钥的速率限制。

**预期结果。** `CODEX_MODE: ready`,评审完成。

<a id="codex-mode-unverified"></a>
### `CODEX_MODE: unverified` / `MODEL_PROBE_INCONCLUSIVE`

**含义。** 短 model 检查超时或遇到网络错误,因此 gstack 无法确认 model 是否有效。评审仍然运行,其自己的结果被检查。

**修复。** 现在无需修复。如果评审失败,其消息会说明原因。

<a id="codex-auth-failed"></a>
### `CODEX_MODE: not_authed` / `AUTH_FAILED`

**含义。** 找不到 Codex 凭据:没有 `CODEX_API_KEY`,没有 `OPENAI_API_KEY`,没有 `auth.json`,也没有在自定义 provider 的 `config.toml` 中 `env_key` 命名的已设置环境变量。

**修复。**

```bash
codex login
```

或导出你的 provider 的 `env_key` 命名的变量。

**预期结果。** `CODEX_MODE: ready`。

[desktop write output truncated: original_bytes=61063; if the omitted remainder is needed, continue with read using offset=439 and the same path.]
