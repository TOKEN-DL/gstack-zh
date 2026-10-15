<!-- AUTO-GENERATED from measure.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## 测量-然后-修复循环(红色付费评估或失败的免费套件分片)

此处的每次重新运行都是**诊断性的,而不是判定**:它从不更改失败运行的记录判定,其制品标记为 `.context/ship-measure/`(永远不是评估历史)下的 `diagnostic`,通道判定仅来自最终的一次完整闸门运行。永不提高预算、降低阈值、添加判定重试或跳过案例以获得通过。
下面的 `ship-measure` 意味着 `bun run ~/.claude/skills/gstack/scripts/ship-measure.ts`。

**失败的免费套件分片(Step 5)。** 在将失败称为预先存在或推送之前,按照所说的那样重新运行该分片的精确文件列表(按运行顺序)`ship-measure free --files <f1,f2,...>`(gstack 本地运行: `--shard <i>`): 10 次重新运行,重新运行 1 单独作为基线,其余标记为并行,每个都有自己的 HOME 和状态根,不稳定重试关闭,在 10 分钟上限下。
`ship_rerun_backend=ubicloud` 设置 `UBICLOUD_API_KEY` 在一个 Ubicloud VM 上运行它们。如果运行程序无法获取文件列表,可以创建一个新的 — 如果运行程序无法获取文件列表,则重新运行失败的整个命令 10 次。报告通过率、完成计数(例如,上限处完成 10 次中的 6 次)和一次失败捕获,然后继续分类。

**红色付费评估(Step 6)。** 对于每个红色案例:

1. **分类。** 阅读红色行的原因、证据和 Expected/Received(`failure_cause`, `failure_detail`; gstack: `bun run eval:pass-rates --run <run> --case <id>`)。
   命名它:来自此分支的产品更改、检测器或参与者缺陷、夹具缺陷、提供程序事件或模型未命中。检测器修复在任何付费试用之前获得免费重播测试。基础设施红色(提供程序中断、运行程序丢失)可以跳过测量: `ship-measure skip --case <id> --reason "<why>"` 在 PR 正文中将其标记为 `unmeasured`。它永远不会被计为通过。
2. **展示计划。** 打印 `ship-measure table`(试验、条形图和每种预算)、从 `ship_measure_*` gstack 配置(从通过率历史或 "estimate unknown" 乘以试验的每次试验成本)。
3. **测量。** `ship-measure measure --case <id> --round baseline [--cost-per-trial <usd>]` 在其种类计数上,在并行,通过项目的文档化单案例评估命令,运行案例,`--command '<command> {case}'`(找到 Step 4 找到测试命令的方式;gstack 不需要,并接受判断 ids)。在没有文档化命令的情况下,询问一次并在报告中记录答案。退出 2 需要批准:用其原因询问一次,然后重新运行 `--approved`。
4. **按退出码决定**(MEETS 9/10、MEETS-qualified 8/10 每个红色都合格、EXTEND 7/10;行为 11/12、10/12、9/12):
   - 0: MEETS 或 MEETS-qualified;完成。
   - 5: needs-classify。从其捕获记录每个列出的红色:
     `ship-measure classify --case <id> --round <r> --trials <2,5> --class <class> --evidence "<path:line>"`。
     合格: `provider`(仅在试用本身提供程序错误上)、`judge-noise`、`model-miss`(引用行)。修复轮: `timeout`、`hang`、`regression`、`fixable`。`unclassified`: 已审查,无证据。
   - 6: EXTEND。`ship-measure cycle --case <id> --round <r>` 一次,在未更改的树上;绝不进行第三批。
   - 1: BELOW,一个修复轮。
5. **根据原因修复。** 从其捕获中对失败的试验进行根本原因 `.context/ship-measure/<case>/<round>/tNN/` 并更改原因。
6. **重新测量。** `ship-measure measure --case <id> --round round-<n> --fix "<cause and change>"`,然后项 4。仅在有命名因果更改和关闭测量的每个失败都被解释时调用案例已修复;报告 "observed k/n after fix at <cause>",永远不是已证明的速率。
7. **停止。** 退出 3 是命名红色(轮限制、预算或两个空批次):停止,不推送,并报告 `ship-measure report` 与每个试验。

**闸门一次。** 当每个红色案例以或高于目标测量时,运行完整闸门一次(Step 6 项 2)。当项目的 CI 在推送时运行该闸门时(gstack: 拉取请求的 `evals` 工作流),推送是闸门运行:推送并读取该运行,并且不启动闸门命令(gstack 的 `bun run eval:bg:pr` 在推送的头部上分派第二个付费运行)。将 `ship-measure report` 表(估计和实际支出/案例)放在 PR 正文中。