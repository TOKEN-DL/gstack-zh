<!-- AUTO-GENERATED from autoplan/DRIVER.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
# /autoplan driver: 由父 agent 驱动评审循环

当(任意宿主上的父 agent)通过 `gstack-autoplan` 驱动一次无人值守的 /autoplan 运行时,请阅读本文件而不是 `autoplan/SKILL.md`:评审员由你分派的 subagent 担任,runner 负责维护状态。命令位于 `<root>/bin/`,其中 `<root>` 是保存这份 `autoplan/` 目录的文件夹;`gstack-autoplan --help` 和首次 `next` 会打印 `DRIVER: <root>/autoplan/DRIVER.md`。执行每条命令时使用安装脚本打印的 `GSTACK_LAUNCH` 赋值。读取打印行和文件,而不是散文:每个工作流命令以 `GSTACK_RESULT: skill=autoplan status=<s> run=<dir>` 结尾,每个错误行以 `(<CODE>)` 结尾。

## 阶段与作用域

顺序:ceo → design → dx → eng。仅在 UI 作用域时运行 Design,仅在 DX 作用域时运行 DX,Eng 永远在最后且必跑。首次 `next` 上的作用域标志:`--ui`、`--no-ui`、`--no-dx`、`--developer-tool`、`--agent-primary`。`--developer-tool` 和 `--agent-primary` 仅启用 DX。首次 `next` 打印 `SCOPE: ui=… (terms; visual; files) dx=… (detected) source=ui:<s>,dx:<s>`(每条轴取值 `flag`、`marker` 或 `detected`)。UI 需要在 Implementation plan 中找到一个渲染过的文件路径或紧邻视觉名词的 UI 术语。若要在同一份 plan 的后续运行中保留这些标志,请将打印的 `SCOPE_PIN:` marker(`<!-- gstack-autoplan-scope: … -->`)自己添加为 plan 的第一行;gstack 永远不会写入 plan。

## 循环

1. `next` 按每个 voice 打印一行 `ATTEMPT: <id> phase=<p> voice=native|outside prompt=<file> result=<file>`。把 native prompt 分派到你模型的 subagent,把 outside prompt 分派到另一模型族的 subagent(或 `gstack-outside-voice run`)。每个 subagent 会写入 `result=`。
2. 每个 voice 恰好提交一次(`BOUND:`)。两个 voice 都到位即关闭该阶段(`PHASE_CLOSED:`)。
3. 重复 `next` 直至 `status=gate_pending`(`GATE: gate_rev=<n>`);通过 `gstack-gate render <dir>/gate.json` 渲染它。绝不替所有者批准 gate:只需要转发。
4. 用所有者的回复 `answer`。`APPLY: <id> <title> -> <change>` 行为被接受的 plan 变更:编辑 plan,然后 `revise`(打印中的 `NEXT:` 标注了名)。在此之前 `next` 会因 `PLAN_AMEND_PENDING` 拒绝,而 `export` 不会把这次运行标记为完成。
5. `REOPENED: <phases>` 表示再次循环 `next`(Eng 最后)。`APPROVED:` 表示 `export`。

### 命令

- `gstack-autoplan next --out <dir> --plan <plan.md> [scope flags] [--light] [--spend-cap <usd>] [--scaffold]`
  首次调用:初始化 run,打印 PHASE:、SCOPE:、DRIVER: 和每个 voice 一条 ATTEMPT:。
- `gstack-autoplan next --out <dir> [--redispatch <attempt>]`
  之后的调用:打开下一阶段或重新打印未关闭的 attempts;绝不自行重新分派。
- `gstack-autoplan submit --out <dir> --phase <p> --voice native|outside --record <file.md> --model <id> [--attempt <id>]`
  绑定一份自由形式的评审记录(canonical fence、一份 findings 表格或列表,或 `No findings.`)。
- `gstack-autoplan submit ... --record <file.md> --attest "<reason>" --attempt <id>`
  当没有 INPUT: 行时,绑定为已签收记录(错误的 INPUT: 一律拒绝)。
- `gstack-autoplan submit --out <dir> --phase <p> --voice native|outside --result <file> --model <id> --attempt <id>`
  绑定一份 canonical result(下文 RESULT FORMAT 合约)。
- `gstack-autoplan submit --out <dir> --phase <p> --voice outside --unavailable|--failed "<reason>"`
  当没有可用的 outside voice 时,以仅 native 方式关闭该阶段。
- `gstack-autoplan answer --out <dir> --gate-rev <n> --reply "<reply>"`
  按下文 gate reply 语法答复 gate。
- `gstack-autoplan revise --out <dir> --plan <plan v2> --incorporates <decision ids>`
  绑定修改后的 plan;重新打开所有已评审且 plan 发生变更的阶段,Eng 最后。
- `gstack-autoplan revise --out <dir> --withdraw <decision id>`
  直接丢掉一条待应用的 amendment。
- `gstack-autoplan export --out <dir>`
  写出 artifact 集合;然后 `gstack-artifact validate <dir>/run.json`。
- `gstack-autoplan status|resume --out <dir>`
  读取 manifest;在崩溃后接管(EXECUTION_UNKNOWN 会列出未完成的 attempts)。

## 评审员 result 格式

每个 prompt 都以此合约结尾;`submit --result` 强制实施该合约,`submit --record` 接受上述更宽松的记录形态。

RESULT FORMAT (gstack-autoplan 要求;父 agent 按此合约绑定你的文件):
1. 你 result 的第一行必须严格是:INPUT: <phase> <snapshot sha256>
2. 写出完整评审散文,然后在末尾以一个 canonical findings fenced 块结束,每行一个 JSON object:
```gstack-findings
{"id":"F1","severity":"High","title":"<one line>","file":"<path or omit>","line":12,"fix":"<the fix>"}
```
   severity 取值为 Critical | High | Medium | Low | Informational。仅当建议变更 plan 的既定方向且该变更会影响其 scope、cost 或 direction 时,才增加 "user_challenge": true;此时再增加 "changes": "scope" | "cost" | "direction"。打包方式、排序、命名以及其他可逆的选择是普通 findings,不算 user challenges。对该 plan 不归属的安全或数据丢失类 finding,加 "urgent": true 和 "suggested_owner": "<who>"。空的 fence 代表无 findings;缺失的 fence 会被拒收。
3. 把完整的 result 写入此文件:<result file>

## Gate 回复

回复绑定到你读取的 `gate_rev`(`--gate-rev`)。裸的 `yes`/`no` 不解析,绝不臆测。

- `all`:每条 item 都采用其推荐(已达成一致的变更会打印 APPLY: 并需要 revise)。
- `all as-is`:保留所有方向;plan 不变。
- `d3b uc1a`:为你设定的每条 item 放置一个 <id><option> token;未答的自动 item 采用各自的默认值。
- `all except d3b`:采用推荐,但对列出的项覆盖其决定。

## 运行目录

Runner 状态:`run.json`、`attempts.jsonl`、`run.lock`、`spend.json`、`plan-active.md`、`restore.md`、`plan-source.md`。评审员文件:`<phase>-native-prompt.md`,
`<phase>-outside-prompt.md`、`<phase>-native.md`、`<phase>-outside.md`、`<phase>-consensus.{json,md}`,
`gate.json`、`decisions.jsonl`。`export` 写出 `plan.md`、`review-record.md`、`findings.jsonl`、
`tasks.jsonl`、`timing.json` 并对 `run.json` 重新计算哈希;`gstack-artifact validate <dir>/run.json`
是完成度检查。复制整个目录,永远不要复制单个文件。

## 结果码

- `RUN_NOT_INITIALIZED`:以 `gstack-autoplan next --plan <file> --out <dir>` 启动(首次调用会初始化 run)
- `PLAN_SECTIONS_INVALID`:补上缺失的标题,或在首次 `gstack-autoplan next` 时加 `--scaffold` 以评审一份脚手架化的副本(你的 plan 文件永远不会被改);重复、倒序或空的 section 需要手工修复
- `RUN_LOCKED`:等它退出;若其 pid 已消失,以 `gstack-autoplan resume --out <dir>` 接管
- `EXECUTION_UNKNOWN`:若评审员已完成,提交其 result 文件(`gstack-autoplan submit ... --attempt <id>`),或显式以 `gstack-autoplan next --redispatch <id>` 重派;不会自动重派
- `SPEND_CAP_EXCEEDED`:上调 `--spend-cap`、结算或释放未结的 attempts,或停止;未知的扣款按其预定额计入
- `PHASE_NOT_AWAITING`:运行 `gstack-autoplan status --out <dir>`,按 `next` 打印的 attempt 提交
- `ATTEMPT_MISMATCH`:传入 `next` 为此 voice 打印的 attempt id;一份 result 只绑定一次
- `RESULT_RECEIPT_MISSING`:评审员必须把 prompt 文件读到 EOF,并以 prompt 中指定的 INPUT 行开头;指向其他 snapshot 的 result 会被拒收
- `RESULT_FINDINGS_MISSING`:用 ```gstack-findings fence 结束评审,每行一个 JSON object 且至少包含 severity 和 title;`[]` 行不算 findings,空的 fence 代表无 findings
- `MODEL_FAMILY_CONFLICT`:传入不同家族的 `--model <id>`(anthropic、openai、google、xai、deepseek、qwen、zai、moonshot、meta、mistral);同家族 result 不构成 outside voice
- `CONSENSUS_MISSING`:恢复运行,以便通过 reconciliation 关闭该阶段;缺少任一前置阶段汇总时,绝不会发出 Eng prompt
- `GATE_REV_STALE`:渲染当前 gate 列表,读取它,并以其 gate_rev 答复
- `GATE_REPLY_UNPARSED`:使用 `all`、`all as-is`、`<id><option>` token(例如 `d3b uc1a`),或 `all except <tokens>` 答复;裸的 yes/no 有歧义
- `PLAN_AMEND_PENDING`:编辑 plan,然后 `gstack-autoplan revise --out <dir> --plan <file> --incorporates <decision id,...>`;或以 `gstack-autoplan revise --out <dir> --withdraw <decision id>` 丢掉一条
- `GSTACK_VERSION_CHANGED`:对于较旧的 run 什么都不需要做(已按日志完成向前迁移);由更新的 gstack 写入的 run 需要那个 gstack:通过 bin/gstack-upgrade 升级

## 协调者合约(每条 lane 都继承)

合并条件。一条 lane 的 PR 仅在以下条件下合并:必需的 gate lanes 在打戳树上是绿的,或树收据显示 `gate-reuse: eligible`;`gstack-lane-check` 对其计划文件相对其他所有开放 lane 报告 `clear`;PR body 以 `gstack-ship-receipt` 块开头;并且由协调者(而不是 lane)来决定合并。lane 永不合并他人的分支、rebase 别人的 lane、或 force-push。

行为变更通知。plan 自身条目之外、面向用户的每项行为变更都必须在评审开始前以 `## Behavior changes` 列出,内容包含文件、旧行为与新行为。被评审发现但未列出的变更无论大小都视为 CRITICAL finding。

开销上限。每条 lane 在协调者命名的 `--spend-cap` 下运行;达到上限的 lane 以 `status=budget_exhausted` 停止并报告 spent、reserved 与 unknown 扣款。付费 evals 不得为改变判定而重试;rerun 需要一条具名修复。宿主子 agent 的开销报告为 `unknown`,从不报 0。

完成前。lane 在其分支上运行 `gstack-pregate run --allow-repo-commands`(变更文件的单元层 + 仓库的 guards),并在通过后才报告完成。若仓库的 `.gstack/pregate.json` 中含有 `sealed` 键,它在任何 push 或 PR 创建之前运行 `gstack-pregate sealed`;拒绝会阻止 push,因为 PR 工具可能不跑本地 hooks。

交接格式。lane 按以下顺序报告,标识符优先:PR URL(或 `no PR` 并附分支与 SHA)、`GSTACK_RESULT` 行、`gstack: ok|fail <rev>` 行、`PREGATE:` 行、gate 摘要(lanes passed/failed/skipped)、行为变更(或 `none`)、开销(spent/reserved/unknown),然后是其无法验证的内容。任何跨线程消息都使用完整 thread id(`jam_…`),绝不简写。标识符之后的散文可以被截断;标识符之前的内容不得截断。