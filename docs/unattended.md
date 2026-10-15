# Unattended 运行:工件合约

当父智能体使用 `GSTACK_SESSION_KIND=unattended` 启动 gstack skill 并读取其工件而不是其散文时,gstack skill 处于 *unattended* 状态。本页是该父级和运行之间的合约:会话种类更改的内容、运行写入的文件、父级如何决定运行是否完成,以及它如何读取结果而不读取任何 markdown。

会话种类是:`interactive`(人工回答提示)、`headless`(`GSTACK_HEADLESS`、CI、eval harness;问题 BLOCK)、`spawned`(编排器子智能体;推荐选项被自动选择)和 `unattended`(本页)。`unattended` 仅作为显式的 `GSTACK_SESSION_KIND=unattended` 覆盖被尊重,它胜过 `GSTACK_HEADLESS`,以便父级可以在 CI 内驱动运行。测试或 eval harness 中的任何内容都不会设置它,因此 `headless` 保留其 BLOCK 语义。

## 会话种类更改的内容

`gstack-skill-start` 打印 `SESSION_KIND: unattended` 和 `UNATTENDED_SESSION: true`,然后在整个 skill 中应用这些规则:

- **没有提示。** 更新检查、升级提供、遥测和同步同意、入门提示和路由块被完全抑制,就像 `spawned` 一样。未回答的同意是 `off`。
- **不发布或同步,在写入器处强制。** 不写入使用分析(`TELEMETRY_WRITE: skipped (unattended session)`),既不拉取也不推送工件仓库(`ARTIFACTS_SYNC: skipped (unattended session; mode=<m> queue=<n> retained)`),同步 spool 不被馈送(`gstack-brain-enqueue` 是无操作,并且 `gstack-review-log` 在 stderr 上说 `review-log: sync skipped (unattended session)`),并且 `gstack-skill-end` 不排空任何内容。继承的启用设置和挂起的队列工作保留给下一次人工会话。升级提供变为技能结束时的一行,从本地缓存读取:`UPGRADE_AVAILABLE: <version>; run /gstack-upgrade`。
- **问题自行决定,gates 不决定。** 具有 `(recommended)` 选项的问题采用它,并将选择附加到 `decisions.jsonl`,其中 `kind: "auto"`。没有推荐的问题、任何同意、任何破坏性或不可逆选项以及最终批准 gate 都附加 `kind: "approval"` 和 `status: "pending"`。该技能以其挂起列表呈现结束于其下一个 gate;它永远不会批准,也永远不会发布。
- **State root 持久性已报告。** `STATE_ROOT: <path> durable=yes|no`(`no` 当主机设置 `GSTACK_EPHEMERAL=1`)。在临时根上,运行会说 `learnings: skipped (state root is ephemeral; set GSTACK_STATE_ROOT)` 和 `timing: analytics skipped (...)`,而不是写入将消失的磁盘。当你想要跨运行历史时,将 `GSTACK_STATE_ROOT` 指向持久目录(参见 [state-root.md](state-root.md))。
- **Autoplan 发布保护已命名,而不是伪造。** 在不执行 Claude Code hooks 的主机上,以及在每个 unattended 会话中,`gstack-skill-start` 打印 `autoplan guard: not enforced by this host; publication order is unverified (GUARD_NOT_INSTALLED)` 一次。`gstack-artifact validate` 在 unattended 运行的 `review-record.md` 中需要该行。Claude Code 的 deny-on-absent 行为保持不变。

未知的 `GSTACK_SESSION_KIND` 值强制为 `interactive` 并这样说:`SESSION_KIND: interactive (unknown kind '<x>')`。

## 终端行

每个工作流技能以一行结束 unattended 运行,父级 grep:

```
GSTACK_RESULT: skill=<name> status=<status> run=<dir>
```

`status` 是 `complete`、`gate_pending`、`incomplete`、`interrupted` 或 `refused` 之一;`run` 是保存下面工件的目录。`/autoplan` 在到达最终 gate 时始终以 `gate_pending` 结束,因为 gate 已写入并且永远不会被批准。纯查询(`gstack-artifact schema`、`gstack-gate render`)不打印结果行。

每个工作流命令共享一个退出表:`0` ok,`1` fail,`2` usage,`3` refused or needs a flag。错误行以括号中的结果代码结尾,例如 `findings.jsonl:12: ... (ARTIFACT_SCHEMA)`;每个代码在 [troubleshooting.md](troubleshooting.md#unattended-runs-and-artifacts) 中都有一个锚点,并带有 `fix:` 子句。Grep 代码,而不是散文。

## 运行目录

| 文件 | Schema | 内容 |
|------|--------|----------|
| `run.json` | `run` v1 | 清单:`run` id,`status`,`required_phases`,每个 phase 的 `native` 和 `outside` 结果(`status`、`model`、`provider`),输入 `snapshot` 哈希,`counts`,每个工件的 `path`(相对于运行目录)和 `sha256`,`session_kind`,`gate_rev`,`deviations`,`consumed_by`。 |
| `decisions.jsonl` | `decisions` v1 | 每个决策一行:`id`(`<run>-<label>` 在 gate 修订 1,`<run>-r<rev>-<label>` 在重新打开之后,因为标签在每个 gate 重新启动),`title`、`options`、`recommended`、`kind`(`auto`、`approval`、`user_challenge`)、`status`(`pending`、`approved`、`overridden`、`rejected`)、`gate_rev`,以及一旦回答 `chosen`、`answered_at`、`reply`。 |
| `findings.jsonl` | `findings` v1 | 每个评审员发现一行:`id`、`phase`、`voice`(`native`/`outside`)、`model`、`severity`、`title`、`disposition`、`plan_items`、`native_counterpart`(字符串或 null)、`source`。行共享评审日志行形状,因此 `gstack-review-log --findings` 从此文件派生评审状态。 |
| `tasks.jsonl` | `tasks` v2 | 实现任务,带有 `id`、`title`、`status`、`depends_on`、`findings`(在 `findings.jsonl` 中的 id)、`blocked_by`、`acceptance`、可选的 `pr`/`item`/`tier`。 |
| `timing.json` | `timing` v1 | 每个 phase 的 `started_at`、`ended_at`、`wall_s`、`outside_s`、`native_s`;运行的 `started_at` 和 `total_wall_s`;Phase 0 `estimate`。 |
| `plan.md` | prose | `## Implementation plan` 部分,在其上方写有 `## URGENT, outside this plan`。 |
| `review-record.md` | prose | 评审记录和决策审计跟踪;在 unattended 运行中携带 `GUARD_NOT_INSTALLED` 行。 |
| `gate.json` | 参见 `gstack-gate --help` | `/autoplan` 渲染和解析的最终 gate 列表。 |

Id 是运行绑定的:每行的 `id` 以 `<run>-` 开头,`run.json` 是声明运行 id 的唯一位置。`gstack-artifact schema <name>` 打印任何 `findings`、`decisions`、`tasks`、`timing`、`run`、`pregate` 和 `ship-receipt` 的 JSON Schema;`schema --list` 打印名称。

## 父级如何使用运行

1. **决定运行是否完成。** `gstack-artifact validate <dir>/run.json` 是确定性完成检查。它退出 `0` 并仅在清单解析并匹配其 schema、每个必需 phase 和每个评审员 voice 具有终端结果、每个列出的工件存在于运行目录中且具有记录的哈希、每个 JSONL 文件逐行解析其具有运行绑定唯一 id 的 schema、任务依赖关系和发现引用解析无循环、`counts` 与文件一致、以及(unattended)保护行存在时,才打印 `ARTIFACT_VALID: run=<id> status=<status>`。否则,它以每个问题一行错误退出 `1`,以及 `ARTIFACT_INVALID: <n> error(s)`;`--json` 将相同的内容作为 `{ valid, run, status, run_dir, errors[] }` 返回。被中断的运行在此处显示为 `ARTIFACT_RUN_INTERRUPTED`,而不是部分成功。
2. **阅读终端行。** `grep '^GSTACK_RESULT:' <log> | tail -1`。使用 `status=gate_pending`,gate 在工件中;没有批准任何内容。
3. **列出待处理的内容。**
   ```bash
   jq -c 'select(.status=="pending")' <dir>/decisions.jsonl
   gstack-gate render <dir>/gate.json          # the numbered list and reply grammar
   ```
4. **通过组合回复语法回答 gate**(`all`、`<id><option>` 标记,例如 `d3b uc1a`,或 `all except <tokens>`;裸 yes/no 被拒绝),并使用你读取的 gate 修订反馈:
   ```bash
   gstack-gate parse <dir>/gate.json --reply "all except d3b" --gate-rev 1 --json
   gstack-gate decisions <dir>/gate.json --reply "all except d3b" --gate-rev 1 >> <dir>/decisions.jsonl
   ```
   陈旧的修订是 `GATE_REV_STALE`,未解析的标记的回复是 `GATE_REPLY_UNPARSED`;两者都退出 1 并且不写入任何内容。
5. **阅读散文之外的发现和任务。**
   ```bash
   jq -c 'select(.severity=="Critical" and .disposition!="accepted")' <dir>/findings.jsonl
   jq -r '.id + "\t" + .title' <dir>/tasks.jsonl
   gstack-artifact urgent <dir>/findings.jsonl   # the URGENT block, or "None."
   ```
6. **确认消耗** 以便第二个父级(或重试)可以说运行已被读取:`gstack-artifact ack <dir>/run.json --consumer <id>` 是幂等的,并在清单中记录 `consumed_by`。

## 从父级驱动 `/autoplan`:`gstack-autoplan`

在父级自行调度评审员的主机上(Capy、任何没有 Claude Code harness 的编排器),`bin/gstack-autoplan` 是整个运行的显式、持久状态机。父级永远不会读取评审记录;它运行两个命令的循环并读取打印的行。任何主机上的父级读取 `autoplan/DRIVER.md`(其路径由 `gstack-autoplan --help` 和第一个 `next` 打印为 `DRIVER:`)而不是 autoplan/SKILL.md:阶段顺序、工件合约和 gate 协议,大约一百行,从代码生成。

```bash
AP=bin/gstack-autoplan
$AP next --out <dir> --plan <plan.md> [--ui|--no-ui] [--developer-tool] [--no-dx] [--light] [--spend-cap <usd>]
#   PHASE: ceo snapshot=<sha> order=ceo,design,dx,eng
#   SCOPE: ui=yes (...) dx=yes (detected: ...) source=ui:detected,dx:detected
#   DRIVER: <root>/autoplan/DRIVER.md
#   Skipped sections: none (scope: ui=yes,dx=yes,source=ui:detected,dx:detected; checklist=loaded)
#   ATTEMPT: <run>-a1 phase=ceo voice=native  prompt=<dir>/ceo-native-prompt.md  result=<dir>/ceo-native.md  model_family=...
#   ATTEMPT: <run>-a2 phase=ceo voice=outside prompt=<dir>/ceo-outside-prompt.md result=<dir>/ceo-outside.md model_family=differs from the native reviewer
#   SPEND: spent=0.00 reserved=0.00 unknown=0(0.00) worst_case=0.00 cap=none
#   GSTACK_RESULT: skill=autoplan status=awaiting_result run=<dir>
```

第一个 `next` 打印 `SCOPE: … source=ui:<flag|marker|detected>,dx:<…>`,并且当 CLI 标志与计划的标记不同时,`SCOPE_PIN: add <!-- gstack-autoplan-scope: … --> as the first line of <file>`;gstack 从不写入计划。

1. **调度每个 `ATTEMPT:`。** 原生 voice 是父级自己的 model 上的子智能体,带有 prompt 文件;外部 voice 是另一个 model 系列上的子智能体,或 `bin/gstack-outside-voice run --runner codex-cli|api --prompt <file> --out <file> [--model <id>]`,它打印 `OUTSIDE_STATUS: <status> provider=<runner> model=<id> family=<f>`。每个评审员将其结果写入 RESULT FORMAT 下的打印的 `result=` 路径。`next` 是幂等的:它重新打印打开的尝试,并且不会自行重新调度(`EXECUTION_UNKNOWN` 列出其进程在没有结果的情况下死亡的尝试;`--redispatch <attempt>` 铸造一个新的并且 `release --attempt <id>` 返回其保留)。
2. **仅提交每个结果一次。**
   ```bash
   $AP submit --out <dir> --phase ceo --voice native  --result <dir>/ceo-native.md  --model <id> --attempt <run>-a1
   $AP submit --out <dir> --phase ceo --voice outside --result <dir>/ceo-outside.md --model <id> --attempt <run>-a2 [--runner host-subagent] [--usage-usd <n>]
   #   BOUND: <run>-a1 phase=ceo voice=native model=<id> findings=7 sha256=<result hash prefix>
   #   PHASE_CLOSED: ceo confirmed=4 disagree=1 new=2 native_only=1 coverage=both
   ```
   绑定根据 phase snapshot 检查收据行(`RESULT_RECEIPT_MISSING`),解析规范的发现围栏(`RESULT_FINDINGS_MISSING`),拒绝来自原生系列的外部 model(`MODEL_FAMILY_CONFLICT`),并拒绝相同尝试的第二次绑定或在 phase 不等待它时的绑定(`ATTEMPT_MISMATCH`、`PHASE_NOT_AWAITING`,退出 3)。两个 voice 关闭 phase:协调(`confirmed` / `disagree` / `new` 每个外部发现,不同意和解决保持分离)写入 `<phase>-consensus.{json,md}`,并且 Eng prompt 拒绝在没有每个关闭的先前 phase 的共识的情况下打开(`CONSENSUS_MISSING`)。
3. **循环 `next` 直到 `status=gate_pending`。** Eng 始终最后运行。Gate(`gate.json`,`gate_rev`)将用户挑战列为批准项,将不同意列为自动项,并将 `p1` 计划批准;没有批准任何内容。
4. **使用回复语法和你读取的修订回答 gate**:
   ```bash
   $AP answer --out <dir> --gate-rev 1 --reply "all"              # every recommendation; agreeing plan changes print APPLY:
   $AP answer --out <dir> --gate-rev 1 --reply "all as-is"        # APPROVED: keeps every direction, no new paid phase
   $AP answer --out <dir> --gate-rev 1 --reply "all except d1b"   # APPLY: <id> ...; REOPENED: ceo,eng; NEXT: gstack-autoplan revise ...
   ```
   用户挑战两个 voice 在确认的发现上以相同的 `changes` 值提出建议为 `b`,并且选项 b 显示两个 voice 的补救措施(`recommended: b (both voices; remedies shown)`);`--reconcile` 记录命名一个共享 `proposal` 的显示该内容。不同意或缺少补救措施保持 `a`。每个接受的计划更改打印 `APPLY: <decision id> <title> -> <chosen text>` 并成为挂起的修正:`next` 拒绝 `PLAN_AMEND_PENDING` 并且 `export` 拒绝 `complete` 运行,直到 `revise --incorporates` 命名它(见下文)。陈旧的修订(`GATE_REV_STALE`,退出 3)、裸 `yes`/`no` 或冲突的回复(`GATE_REPLY_UNPARSED`)不写入任何内容。留下批准项挂起的回复仅记录已回答的自动项并保留 gate;以后的回复永远不会覆盖较早的答案。更改计划的选项重新打开该 phase 以及 Eng,将其绑定文件归档为 `*.r<rev>.*`,并且下一个 gate 是 `gate_rev + 1`;`p1d` 在当前计划上重新运行每个 phase;`p1e` 以 `incomplete` 结束。
5. **导出和验证。** `$AP export --out <dir>` 写入 `plan.md`(首先包含 `## URGENT, outside this plan` 块)、`review-record.md`、`tasks.jsonl`、`decisions.jsonl`、`findings.jsonl`、`timing.json` 和 `run.json`;然后 `gstack-artifact validate <dir>/run.json` 并且上述消耗步骤不变地应用。`$AP status --out <dir>` 和 `resume` 在运行目录移动后工作(路径重新生根)。

并发和恢复:每个状态写入都是锁定的读-改-写(`run.lock`;第二个写入器看到 `RUN_LOCKED`),尝试在调度之前记录到 `attempts.jsonl`,并且在 `submit` 之前或之后被杀的父级恢复 `next` 而无需重复收费(`test/autoplan-run-recovery.test.ts`、`test/autoplan-run-concurrency.test.ts`、`test/autoplan-answer-invalidation.test.ts`)。`--spend-cap` 通过 `spend.json` 接受每个尝试(`SPEND_CAP_EXCEEDED` 拒绝,退出 3);host-subagent 花费为 `unknown` 并计入 `worst_case`。`--deadline <min>` 使 `run` 子命令取消它启动的子项并将运行标记为 `interrupted`。

### RESULT FORMAT(每个评审员文件必须包含的内容)

每个 prompt 文件以此合约结束;`submit` 强制执行。

1. 第一行正好是 `INPUT: <phase> <snapshot sha256>`(将结果绑定到两个 voice 评审的快照的收据)。
2. 评审是散文,然后是一个规范的发现围栏,每行一个 JSON 对象:`{"id":"F1","severity":"High","title":"...","file":"...","line":12,"fix":"..."}` 在 ```` ```gstack-findings ```` 围栏内。`severity` 是 Critical | High | Medium | Low | Informational;`"user_challenge": true` 标记对计划所述方向的推荐更改,需要 `"changes": "scope" | "cost" | "direction"`(接受它会更改什么);`"urgent": true` 与 `"suggested_owner"` 标记计划不拥有的安全或数据丢失发现。空围栏表示没有发现;缺少围栏被拒绝。
3. 整个结果写入打印的 `result=` 路径。

### 外部 voice runners

| Runner | 访问 | 记录标签 | 需要 |
|---|---|---|---|
| `codex-cli` | 仓库,只读,在此执行 | `repository review` | PATH 上的 `codex` 并已登录(`bin/gstack-codex-login` 从 stdin 读取 `OPENAI_API_KEY`) |
| `api` | 仅提供的输入 | `supplied-input review` | `--model <id>` 和 `OPENAI_API_KEY` 或 `ANTHROPIC_API_KEY`;永远不会静默等于仓库评审 |
| `host-subagent` | 主机的 | `host-dispatched review` | 子智能体写入的 `--result <file>`,`--model <id>`;花费 `unknown` |

`bin/gstack-outside-voice runners` 打印每个 runner 在此机器上的可用性;`family <model>` 打印 `submit` 比较的 model 系列。已完成的 `api` 或 `host-subagent` 评审是记录中的完整外部 voice,由其 runner 标记。结果代码:`EXECUTION_UNKNOWN`、`RUN_LOCKED`、`RUN_NOT_INITIALIZED`、`PHASE_NOT_AWAITING`、`ATTEMPT_MISMATCH`、`RESULT_RECEIPT_MISSING`、`RESULT_FINDINGS_MISSING`、`MODEL_FAMILY_CONFLICT`、`SPEND_CAP_EXCEEDED`、`CONSENSUS_MISSING`、`OUTSIDE_RUNNER_UNAVAILABLE`(锚点在 [troubleshooting.md](troubleshooting.md))。

### 手动运行的记录、修订的计划和准入

- **手动运行的评审。** `submit --record <file>` 绑定未遵循 RESULT FORMAT 的评审:`gstack-findings` 围栏、一个发现表(Severity 和 Title 列)、一个严重性标记列表(`- [High] title`)或 `No findings.` 行。没有其 `INPUT:` 行,它仅与 `--attest "<reason>"` 和 `--attempt <id>` 绑定(收据 `attested`;在以后的 `revise` 后被拒绝),并且 `gstack-artifact validate` 打印 `attested=<n>`。当恰好一个尝试打开时,可以省略 `--attempt`。`submit --voice outside --unavailable "<reason>"`(或 `--failed`)在 native 绑定后关闭 phase `native_only`;`next` 和 `resume` 永远不会重新调度它。
- **在同一运行中修订计划。** 编辑源计划,然后 `$AP revise --out <dir> --plan <file> [--incorporates <decision id,...>]`。其评审的 Implementation-plan 哈希不同的每个 phase 重新打开(加上 Eng);新加入范围的 phase 被添加;归档保留 `.r<rev>` 名称,仅在冲突时使用 `.v<k>`;在 `gate_pending` 修订使下一个 gate 为 `gate_rev + 1`。`--incorporates` 仅释放命名的修正并拒绝未更改的计划;`revise --out <dir> --withdraw <decision id>` 将其作为已记录的所有者决策删除。`next` 打印 `PLAN_CHANGED` 并且 `answer` 在源计划自快照以来已移动时打印 `PLAN_SOURCE_CHANGED`。没有 `## Implementation plan` 和 `## Review record` 的计划以 `PLAN_SECTIONS_INVALID` 拒绝;`next --scaffold` 评审缺少标题的副本。`resume` 完成中断的修订。
- **准入和升级。** `next`、`submit`、`answer`、`revise`、`export` 和 `run` 在 Capy 主机(`GSTACK_CAPY=1`)上没有此 checkout、HEAD 和工作流的 ok install verdict 以及 `$GSTACK_INSTALL_VERDICT`(`INSTALL_UNVERIFIED`)的情况下拒绝,并且在任何 unattended 会话中,当运行中的 Bun 低于 gstack 支持的最低版本(`RUNTIME_BELOW_MINIMUM`)时拒绝;该决定与 `gstack-skill-start` 所做的相同(参见安装准入);`GSTACK_ALLOW_UNSUPPORTED_RUNTIME=1` 覆盖并记录运行偏差。`status` 和 `resume` 从不设限。由较旧 gstack 编写的运行目录打印 `GSTACK_VERSION_CHANGED` 一次并向前移动(旧决策 id 被重命名并记录);由较新 gstack 编写的运行被拒绝。

## Owner brief

`gstack-owner-brief render --out <dir>` 是所有者在最终 gate(交互式或 unattended)看到的内容:从 `gate.json`、`decisions.jsonl` 和 `findings.jsonl` 构建的一页(`BRIEF_INPUT_MISSING` 当一个缺失)。它以 `URGENT, outside this plan` 块开头,然后是简单语言的摘要(关闭的 phase、接受的发现、多少决策采用其默认值、周期时间;`--summary <file>` 或 `<dir>/summary.md` 用运行的自己的内容替换生成的句子),然后通过 gate 列表的编号决策:每页七个(`--page N`),稳定的 id,每个 `auto` 项带有未回答时采用的默认值的注释,每个 `approval` 项 `pending until you answer` 或 `decided <id><letter> on <date>`。最后一行命名 `review-record.md`,其中存在完整的评审,以及回答该页的 `gstack-autoplan answer` 调用。`--write` 还在运行目录中保存 `brief.md`。`/autoplan` Phase 4 逐字呈现此输出,并将以前的长报告写入 `review-record.md`。

在最终 gate 之后,`gstack-owner-brief close --out <dir> --plan <file>` 重写计划中的每个 `draft direction stands until the owner decides` 行到决定的选项和日期(`DRAFT_DIRECTION: line <n> <id> rewritten`);决策仍然挂起的行保留并报告为 `DRAFT_DIRECTION_UNRESOLVED`(退出 1)。`gstack-autoplan answer` 在回复完成 gate 时运行相同的重写(日志事件 `draft_directions_closed`)。

## Lane 完成:pregate、密封保护和交接行

协调器下的 lane 在其分支上运行 `gstack-pregate run --allow-repo-commands`

[desktop write output truncated: original_bytes=26946; if the omitted remainder is needed, continue with read using offset=329 and the same path.]
