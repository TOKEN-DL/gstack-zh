<!-- AUTO-GENERATED from phase-close.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
当本阶段的评审工作结束时,重新读取本 section。使用本阶段退出时绑定的 phase、amendment checkpoint 与 methodology path。本流程将 readback、verification 与 publication 视作相互独立的不同操作。在把 section 内联的宿主上,每次退出都要重新读取这份安装版 Autoplan SKILL.md 中的 close 块;它们没有独立的 phase-close.md 文件。

1. **完成并保存评审。** 要求本阶段的全部 methodology/section 已 Read、强制输出已生成、写入成功、评审员给出终态 result。把已完成 native 评审的 INPUT 与其 voice snapshot 匹配。等待中的评审员会让本阶段继续开放。对失败的 native attempt 应用本阶段的失败策略;unavailable/disabled voice 不得获得完成度。记录 wall time:`~/.claude/skills/gstack/bin/gstack-autoplan-timing close --run <RUN_ID> --phase <PHASE> --out "$_AP" --outside-s <N> --native-s <N>`(写入 timing.json;ephemeral state root 通过一行打印跳过 analytics append)。
2. **调和已接受的需求。** 把每项已接受的 behavior、condition、test 和 manual checklist 记录进本阶段的 accepted 块。Taste 仍是临时性的;User Challenges 保留原始需求。`None` 记录必须解释实现为何保持不变。把 amendment checkpoint 在本次调用期间固定不变,包括 compaction 之后。
3. **准备本阶段的 close packet。** 以退出时的 phase/checkpoint 运行:
```bash
bun "<SNAPSHOT_TOOL>" prepare-close "<PHASE>" "<ACTIVE_PLAN>" "<AMENDMENT_CHECKPOINT>" "<RESTORE_PATH>" "<methodologyPath>"
```
本命令应用已接受的需求,并导出不可变的 packet,包含完整的当前实现、固定的 checkpoint、哈希与阶段专属 `report` 字段。盲评审员的输入保持不变。这些是步骤 4-6 的输入;准备工作不执行它们。
4. **读取完整的当前 packet。** 对每个返回的 `readRanges` 条目,以该条目给出的精确 `offset` 与 `limit` 读取 `closePacketPath`。把全部条目读到 EOF。只读取编辑后尾部不满足此步骤;先前的 snapshot 也不满足。如果结果被截断,补读其缺失范围。如果 Read 失败,修复并补读缺失范围。不得在没有结果的情况下推进请求。在最后一条成功的 Read 之后,在原位执行步骤 5。
5. **验证当前实现。** 把完整的当前实现与已接受的决定、source requirements、conditions、tests 与必填 outputs 比较。Retention check 通过字节证明;counts、hashes、关键字探针或保存的"Read-back"一句话都不足以完成此项语义审查。评审历史留在 Review record 中。复查步骤 1 的前提。任何前提未完成都让本阶段保持开放,把缺失工作补完。修复遗漏,然后以相同 checkpoint 重新生成 packet,并在发布前完整 Read 整个新 packet。任何后续的 implementation 或 accepted-decision 编辑(包含 compaction 后)都返回步骤 3。
6. **发布父报告。** 验证成功之后,立刻把填好的报告作为可见的父 agent 文本发出,使用实际 findings 与 voice statuses;它自己一条消息,唯一的工具调用是 Bash no-op `true autoplan-published <PHASE>`(无输出)。守卫仅在后续记录紧随其后时计一份报告;因此本消息必须在任何下一阶段工具调用之前发出。本阶段使用 packet 的 `report` 字段、本宿主真实的评审员名称,任一 review voice 缺失时使用 N/A;confirmed 计数要求两个 voice 都到位。仅当 `report.includeDxMetrics` 为 true 时,才包含 DX metrics 行。`report.next` 按 driver 适用的 scope/skip 规则解析。

**Phase <report.number> complete.**
[DX only: DX overall: <score>/10. TTHW: <observed> min → <target> min.]
Outside review: <completed: N concerns / unavailable / disabled>. Native subagent: <completed: N issues / unavailable>.
Consensus: <N/A (voice coverage missing) | X/<report.total> native+outside confirmed; Y disagreements → gate>.
Passing to <applicable report.next>.

7. **返回 driver。** 在发出实际的父报告之后,在同一轮继续。在后续消息中进行下一个受守卫的 `Read` 或 `Agent` 调用(下一阶段的 driver,或在任何 skip 消息之后的 Phase 4 tasks aggregator)。只有 driver 可以推进阶段并发出适用的 skip 消息;skip 永远不是完成。不要等待"continue"回复。

已发出的会话消息就是步骤 6 的输出。把它存进 ACTIVE_PLAN 或通过 Bash 打印都不算发布。Compaction 之后,把绑定的 packet 与实际已发出的消息对账:已验证但未宣告的阶段从步骤 6 续上;过期的输入返回步骤 3。Helper 结果或 Read 都不完成 verification 或 publication。