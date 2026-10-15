# /autoplan 发布保护:故障排除

`/autoplan` 在其 phase 边界安装 `PreToolUse` hook(`autoplan/bin/phase-publication-hook`)。在允许 phase-entry `Read` 或 reviewer `Agent` 调度运行之前,它读取此会话自己的 Claude Code 日志(`~/.claude/projects/<project>/<session>.jsonl`),并检查前一 phase 的报告是否已发布为父助手文本。

每个保护消息命名一个**代码**、写入日志的 Claude Code 版本,以及下面该代码的节的链接。在报告问题时包括代码和版本。保护的决策日志(参见 [决策日志](#the-decision-log))保存相同的信息。

每个代码有四种处置之一:

| 处置 | 含义 | 你做什么 |
|-------------|---------------|-------------|
| Unverified allow(未验证的允许) | Claude Code 保持此调用无法检查。该调用运行,并带有可见的警告。 | 无,除非它持续发生(然后报告它)。 |
| Fallback(回退) | 重试无法帮助。 | 运行下面的回退。 |
| Corrective(纠正) | 必须先发生一个命名操作。 | 执行该操作。 |
| Transient(瞬态) | 某些内容仍在进行中。 | 等待命名内容,然后继续。 |

**回退**(在每个回退拒绝中打印):

1. 手动运行评审:`/plan-ceo-review`,然后 `/plan-devex-review`,然后 `/plan-eng-review`。
2. 或者移动到新会话并保留你的工作:`/context-save`,在项目中启动新的 Claude Code 会话(不是 `--resume`),`/context-restore`,然后 `/autoplan <plan path>`。

没有环境变量可以关闭保护。

## phase 转换如何发布

Claude Code 可以在 hook 已经运行之后将工具调用写入日志,并且与工具调用在同一助手消息中的文本对 hook 不可见。因此,保护仅当以后的日志记录跟随它时才计算 phase 报告:工具结果,或来自另一个助手消息的记录。

因此,phase-close 程序在一条消息中发布每个报告,其唯一的工具调用是 Bash no-op `true autoplan-published <phase>`(它不打印任何内容并且不需要权限;Claude Code 2.1.292 要求批准裸 `:` 命令,因此 no-op 是 `true`),并使下一 phase 的 `Read` 或 `Agent` 调用在以后的消息中。将报告和下一 phase 的调用放在一条消息中的会话会收到一个 [`publication_unflushed`](#publication-unflushed) 拒绝,命名该修复。

当 Claude Code 尚未写入当前调用时,保护从 hook 有效负载而不是评估调用(Claude Code 自己的解析输入),放在最后日志记录之后。前一 phase 的报告仍必须在日志中。

## 未验证的允许

这些状态由 Claude Code 引起,而不是由模型引起。保护让调用在没有权限决定的情况下运行(Claude Code 自己的权限检查仍然运行),向你打印一条警告,命名代码,并使用 `disposition: "unverified"` 记录允许。对于该一个调用,它**不**检查前一 phase 的报告是否已发布。

> [autoplan] phase publication was NOT verified for this session (journal_lag): ...

<a id="unrecognized-shape"></a>
### `unrecognized_shape:<detail>`

日志根具有保护不认识的形状,通常是因为 Claude Code 更改了其日志格式。保护读取日志两次,并且仅当两次读取一致时才允许。`unrecognized_shape:cwd_spelling` 意味着日志的目录是项目的同一文件夹,但拼写不同。请报告详细信息和版本。

<a id="journal-lag"></a>
### `journal_lag`

Claude Code 尚未将前一助手消息写入日志(早期消息中的工具仍然没有结果,或祖先记录缺失)并且未在 2 秒内赶上。

<a id="rewritten"></a>
### `rewritten`

保护已经读取的日志字节在一个 hook 调用期间发生了变化:稍后读取的字节 [0, first size) 的 sha256 与第一次读取的不同,或两个重新加载不再解析为一次索引的记录通过。Claude Code 通常仅追加,并且追加永远不会使读取失败。Hook 调用之间的重写不被检测(接受的限制)。

<a id="too-large"></a><a id="journal-too-large"></a>
### `too_large`

单个日志记录超过 32 MiB 记录限制(`OWNED_RECORD_MAX_BYTES`)。保护逐条记录地读取长日志,因此仅大型日志不会触发此问题;一个巨大的记录(例如一个巨大的粘贴文件)会触发。仍在写入的记录(尚无换行符)根本不被读取。

<a id="unseen-version"></a>
### `unseen_version`

比 gstack 检查的版本更新的 Claude Code 版本将 hook 传递到 reviewer 调度,没有日志具有的键(schema 剥离)。每日 `autoplan-schema-canary` 用例在版本发布后的一天内捕获这些。

## 回退拒绝

<a id="competing-root"></a>
### `competing_root`
日志具有多个对话根,或两条记录共享一个 UUID。

<a id="foreign-cwd"></a>
### `foreign_cwd`
日志的根在另一个项目目录中写入,考虑了拼写和符号链接。项目的链接 git worktree 是可以接受的。

<a id="sidechain"></a>
### `sidechain`
第一回合或其祖先是 sidechain(子智能体)记录,而不是父会话。

<a id="agent"></a>
### `agent`
对话祖先通过子智能体记录。

<a id="cycle"></a>
### `cycle`
日志的父链接形成一个循环。

<a id="identity"></a>
### `identity`
日志路径、其目录或文件未通过身份检查:符号链接、外部会话、错误的目录布局、不同的文件,或缩小的日志。重新运行命中相同的检查。

<a id="oversized-invocation"></a>
### `oversized_invocation`
当前的 `/autoplan` 调用需要完整读取的日志记录比 32 MiB 保留数据限制(`OWNED_RETAINED_MAX_BYTES`)更多。保护索引没有内容的每个记录,并且仅完整地重新加载它评估的内容:快照初始化结果、受保护和关闭数据包的读取、结构化 Bash 结果、错误结果、第一个关闭数据包读取之后的助手文本以及对活动计划的编辑。重新运行读取相同的记录;使用回退。

<a id="malformed"></a>
### `malformed`
完整的日志行不是有效的 JSON 或 UTF-8,或其记录相互矛盾。

<a id="journal-missing"></a>
### `journal_missing`
hook 命名的会话日志在 2 秒后不存在。

<a id="event-order"></a>
### `event_order`
日志的事件无法为本次会话按一致顺序排列。

<a id="tool-identity"></a>
### `tool_identity`
两条日志记录声称为同一工具调用标识。

<a id="current-mismatch"></a>
### `current_mismatch`
日志使用与 Claude Code 传递给 hook 的不同工具或输入记录当前调用。只有 `run_in_background`(Claude Code 的 fork-subagent schema 删除) 和 `file_path` 的拼写可能不同。

<a id="agent-key"></a>
### `agent_key`
reviewer 调度携带的键在 `prompt`、`description`、`subagent_type`(仅 `general-purpose`) 和 `run_in_background` 之外。不允许 `/autoplan` reviewer 调用的 model 覆盖。日志记录键名称,而不是其值。

<a id="cross-phase-batch"></a>
### `cross_phase_batch`
一条助手消息进入两个不同的 phase。进入同一 phase 的调用可以共享一条消息;另一 phase 需要在前一份报告之后的自己的消息。

<a id="init-mismatch"></a>
### `init_mismatch`
快照初始化返回的路径与自己的命令参数的标准形式不匹配,或其工件(还原点、活动计划标题)与此调用不匹配。保护以与 `init` 相同的方式对 `<active>` 和 `<restore>` 参数进行规范(参见 [符号链接的计划目录](#symlinked-plan-directories))然后进行比较,因此重新定位或伪造的 init 结果在此被拒绝。一次 init 的零结果或重复结果也会拒绝为 `init_mismatch`。不要重新运行同一调用:使用保护期望的路径运行 init。

<a id="snapshot"></a>
### `snapshot`
phase 快照、方法论或其他不可变工件缺失、被别名化、可写或已更改。

<a id="dispatch-prompt"></a>
### `dispatch_prompt`
reviewer 提示与快照的精确 `nativeDispatchPrompt` 不同。

<a id="evidence"></a>
### `evidence`
保护需要的工件无法解析。

<a id="publication-repeat"></a>
### `publication_repeat`
该报告已在此调用中单独发布一次,并且仍然无法验证,因此 Claude Code 没有及时将其写入日志。

## 纠正拒绝

<a id="publication-missing"></a>
### `publication_missing`
前一 phase 的关闭数据包读取之后不会出现已填充的报告。将其发布为你自己的助手文本,消息中唯一的工具调用是 `true autoplan-published <phase>`,然后在以后的消息中进入下一 phase。

<a id="publication-unflushed"></a>
### `publication_unflushed`
报告共享当前调用的消息,或者尚无日志跟随它。将其发布在带有 no-op 调用的自己的消息中,然后在以后的消息中进入下一 phase。同一调用中的第二次出现变为 [`publication_repeat`](#publication-repeat)。

<a id="close-required"></a>
### `close_required`
完成前一 phase 的关闭程序并读取其关闭数据包。

<a id="close-incomplete"></a>
### `close_incomplete`
读取当前关闭数据包的每一行。

<a id="close-stale"></a>
### `close_stale`
关闭数据包或其检查点不再与当前 phase 输入匹配。准备新的数据包并读取它。

<a id="close-edits"></a>
### `close_edits`
关闭读取后的计划以保护无法重建的方式进行了更改,或者它无法通过其 phase 检查。重复关闭程序。

<a id="init-required"></a>
### `init_required`
此调用没有成功的快照初始化。运行现有的 init 步骤。

<a id="init-own"></a>
### `init_own`
新的 `/autoplan` 回合需要自己的初始化(它可以回答 `reused: true`)。

<a id="init-failed"></a>
### `init_failed`
最近的快照初始化调用失败;重新运行具有相同三个路径的同一 init 命令。出错的 init 被跳过,因此稍后成功的 init 仍然绑定。仅当没有 init 曾经成功时,或者当出错的 init 跟在好的之后时,此拒绝才适用。

<a id="init-unbindable"></a>
### `init_unbindable`
Init 通过 shell 变量、替换、链接、管道或重定向运行。将其作为带有字面绝对路径的一个 Bash 调用重新运行。

<a id="phase-order"></a>
### `phase_order`
在以后的 phase 之前输入 Phase 1(CEO)。

<a id="hook-input"></a>
### `hook_input`
hook 无法读取其输入。恢复安装(`./setup`)。

<a id="installation"></a>
### `installation`
hook 意外失败。恢复安装(`./setup`)。

<a id="foreign-install"></a>
### `foreign_install`
phase 文件属于另一个 gstack 安装,而不是 hook。恢复此安装(`./setup`)。

## 瞬态拒绝

<a id="entry-pending"></a>
### `entry_pending`
来自早期消息的 phase-entry 调用尚无结果。等待它。

<a id="mutation-pending"></a>
### `mutation_pending`
关闭读取后对活动计划的编辑尚无结果。等待它。

## 已删除的代码

<a id="changing"></a>
### `changing`
在 2026 年 10 月 7 日发布之前,`changing` 在 Claude Code 在保护读取日志时追加到日志时拒绝调用。有界读取器读取打开时存在的字节并忽略仍在写入的尾随记录,因此追加永远不会使读取失败;更改的前缀是 [`rewritten`](#rewritten)。

<a id="current-missing"></a><a id="pending-read"></a>
在 2026 年 10 月 7 日发布之前,"Native parent evidence has not reached the journal yet"(无代码) 和 "Current native phase-entry identity is required before entering a new phase" 拒绝了 Claude Code 尚未记录到日志的调用。有效负载路径取代了两者。

## Claude Code 设置

`CLAUDE_CODE_FORK_SUBAGENT=false` 不再需要(保留它是无害的)。使用 fork-subagent gate,reviewer 调度可能在后台运行:调度返回启动收据,reviewer 稍后作为完成通知到达。在关闭 phase 之前等待该通知。后台 reviewer 保持调用 armed,因此在其运行时键入的消息不会结束 `/autoplan`。

### 后台 reviewer 之后的权限卡

skill 的 `allowed-tools` 仅适用于调用它的回合,后台 reviewer 的完成通知启动新回合。从那时起,Claude Code 的正常权限规则适用,因此会话工作目录之外的 Reads 可以显示卡,在 Manual 模式下,不是只读的 Bash 命令也会显示卡。Skill hook 在整个会话中保持注册,但不在子智能体内运行。

`/autoplan` 因此使自己的文件可读而无需卡:

- 还原点和每个 phase 工件(方法论、reviewer 输入、快照、关闭数据包)存在于项目的 `.gstack/tmp/autoplan/`,Step 1 将其添加到 `.git/info/exclude`。Claude Code 在工作目录中读取文件而无需卡,用于 `/autoplan` 及其 reviewer 子智能体。`git clean -x` 删除它们,包括还原点。
- 发布 hook 批准 `/autoplan` 自己的在工作目录之外的 Reads:此安装的 `autoplan/sections/*.md`,以及会话开始于存储库子目录中时 `<repo>/.gstack/tmp/autoplan/` 中的不可变工件。它不批准任何其他内容,也从不批准保护拒绝的调用。

在存储库子目录中启动的会话在 reviewer 读取其输入时仍会显示卡:子智能体在没有 skill 的 hook 的情况下运行。在存储库根目录启动会话,或使用"Yes, allow reading from ... during this session"回答卡。在 Manual 模式下,第一个后台 reviewer 之后的 `/autoplan` 的 Bash 命令也会显示卡;自动模式,在支持的 model 的当前 Claude Code 上是默认模式,将其发送给它的分类器。

## 支持的日志根

保护每个日志文件仅接受一个根:

- 没有父级的用户消息(没有 hook 的会话)。
- 一连串无消息的 `SessionStart` hook 附件(`hook_success`、`hook_additional_context` 等),以 null 父级开始,以第一个用户回合结束。任何 `SessionStart` hook 安装时,Claude Code 都会写入此内容,包括插件添加的 hook。它还在 `/clear` 之后写一个。
- 逻辑父级不在文件中的 `compact_boundary` 记录,但仅当它是文件的第一个记录时。使用 `--fork-session` 恢复压缩的会话时,Claude Code 会写入此内容。

`--continue`、`--resume` 和 `/compact` 追加到同一日志,保护通过其现有的父链接跟踪它们。这些规则检查的真实日志来自 Claude Code 2.1.284(`test/fixtures/claude-native-journal-roots-2.1.284.json`);周期性的 `autoplan-journal-drift` canary 重新检查固定的 Claude Code。

## 决策日志

每个受保护的决策将一行附加到 `<state root>/analytics/autoplan-guard.jsonl`(通常 `~/.gstack/analytics/autoplan-guard.jsonl`):`schema`、时间、`decision`(`allow` 或 `deny`)、`disposition`、`code`、`path`(`journal`、`payload` 或 `none`)和 Claude Code 版本。它永远不保存内容;`agent_key` 拒绝添加键名称,而无法识别的形状添加记录类型名称。文件一旦通过 256 KiB 就保留最新的 1,000 行。写入失败永远不会更改决定。

## Windows 路径

Claude Code 和 Git Bash 可以将一个路径拼写为 `C:\x`、`c:\x`、`C:/x` 或 `/c/x`。保护将这些视为同一路径。它不解析 `.` 或 `..`,因此包含它们的路径仍然被拒绝。

<a id="symlinked-plan-directories"></a>
## 符号链接的计划目录(macOS /tmp、/var)

保护按规范路径比较计划路径,而不是按 argv 拼写。在 macOS 上,`/tmp` 和 `/var` 是到 `/private/tmp` 和 `/private/var` 的符号链接,因此 `/tmp` 或 `/var/folders` 下的计划预期在 init JSON 中显示为 `/private/...`。这是设计:`init` 报告规范路径(使用 `realpath` 解析最深的现有祖先,并重新加入缺失的段,`bin/gstack-autoplan-snapshot.ts` 中的 `canonicalDestination`),保护以相同的方式对 init 命令的 `<active>` 和 `<restore>` 进行规范化。对活动计划的写入和编辑也按规范路径匹配,因此通过别名拼写编辑计划是被识别的。符号链接的计划目录是允许的;仅与其自身参数的标准形式不同的结果被拒绝(`init_mismatch`)。
