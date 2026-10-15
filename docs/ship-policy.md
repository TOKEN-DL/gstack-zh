# 仓库发布策略:`.gstack/ship-policy.json`

`/ship` 为每个仓库读取一个机器可读的文件,该文件说明如何标记和合并发布:提升级别、队列模式、仓库自己的发布工具、版本提及被重写的路径、CHANGELOG、配对的镜像、历史策略和预注册的评估。没有文件,什么都不会改变:每个键都有一个再现今天行为的默认值。

该文件是**从基础分支上提交的副本读取**(`origin/<base>`),而不是从 PR 的工作树中,因此 PR 不能将命令混入其自己的 gate。**文件的存在永远不会授权执行:** `gstack-restamp` 仅在 `/ship` 下或使用 `--allow-repo-commands` 运行 `release_tool`;独立运行会列出命令并以退出 3 和 `repo commands not executed (pass --allow-repo-commands)` 退出。引入或更改策略的 PR 在评审的引导路径(`--policy-from head --allow-repo-commands`)下运行,其收据显示 `policy: head (unreviewed)`。

工具:`gstack-ship-policy init|validate|show-effective|check-history`、`gstack-restamp`、`gstack-tree-receipt`、`gstack-ship-receipt`、`gstack-evidence bundle|ancestor`。它们共享退出表(`0` ok,`1` fail,`2` usage,`3` refused or needs a flag),并以 [troubleshooting.md](troubleshooting.md#queue-aware-ship-policy-restamp-tree-receipt-ship-receipt) 中的结果代码结束每个错误行。

<a id="schema-v1"></a>
## Schema v1

`gstack-ship-policy init` 在其默认值下写入每个键;`init --explain` 打印此表。JSON 不带注释,因此示例存在于 `.gstack/ship-policy.example.json`。未知键是错误。

| 键 | 值 | 默认 | 含义 |
|---|---|---|---|
| `bump` | `patch` \| `ask` | `ask` | `patch` 跳过 `/ship` 步骤 12 和 `gstack-restamp --next` 中的 MINOR/MAJOR 提示,提升 PATCH;`ask` 保留提示。 |
| `queue_mode` | `claim-at-open` \| `stamp-at-merge` | `claim-at-open` | 参见 [stamp-at-merge](#stamp-at-merge)。`claim-at-open` 是今天的队列(打开 PR 时 `gstack-next-version` 声明一个槽位)。 |
| `release_tool` | 命令或 `null` | `null` | 仓库自己的发布命令(`bun run release:restamp`)。在隔离的暂存 worktree 中的 [release-tool contract](#release-tool-contract) 下运行。 |
| `release_outputs` | globs | `[]` | 发布工具可能写入版本文件、lockfiles、CHANGELOG 和 `stamp_paths` 之外的路径。任何其他内容都是 `RESTAMP_OUTPUT_OUTSIDE_ALLOWED`。 |
| `stamp_paths` | globs | `[]` | 在重新标记时,精确的 `(was X)`、`since: X` 和 `vX` 提及分支的暂定版本被重写的位置。 |
| `changelog` | 路径或 `null` | `CHANGELOG.md` | 在重新标记时重新设置标题:PR 的 `## [Unreleased]`(或旧版本)部分移至顶部为 `## [<version>] - <date>`。 |
| `mirror` | 相对路径或 `null` | `null` | 一个配对的 checkout 以相同方式重新标记(tier 2:今天已验证和限制,但尚未运行)。 |
| `history` | `merge-only` \| `squash-ok` | `squash-ok` | `merge-only` 使 `/ship` 拒绝 squash 或 rebase 计划;`gstack-ship-policy check-history` 报告重写的分支(`HISTORY_POLICY_VIOLATION`)。 |
| `preregistration_shas` | commit ids | `[]` | 每个都必须是推送前 HEAD 的祖先:`gstack-evidence ancestor --from-policy`;ship receipt 记录结果。 |
| `stamp_sensitive_gates` | evidence labels | `[]` | 读取 VERSION 或生成的字节的 Gates(gstack 自己的 `tests` lane 通过 agents-digest 新鲜度测试执行)。tree receipt 永远不会将它们标记为可重用。 |
| `pr_count` | `{ default_pr_count, split_requires_owner }` | `{1, true}` | plan reviews 中的 PR-count 行。 |
| `maintainers` | GitHub logins | `[]` | `/review` 贡献者模式的维护者集;空表示仓库推送权限。 |
| `stamp_files` | globs | `[]` | 仓库的标记规则:设置后,只有这些文件可能因标记而不同(列出版本文件、清单、插件树以及 PR 自己的 `since:` 行的文件)。它在每个其他标记规则之前应用,因此其外的文档 `vX` 提及是差异。 |
| `stamp_patterns` | 文本模板 | `[]` | 最多 16 个模板,每个最多 128 个字符,每个正好有一个 `{version}` 槽(`"version": "{version}"`、`since: '{version}'`、`<!-- x-stamp: {version} -->`);在版本边界按字面匹配,永远不作为正则表达式。restamp writer 和 tree receipt 将它们与内置形式一起使用。 |
| `restamp_command` | 命令或 `null` | `null` | 仓库自己的 restamp,在暂存 worktree 中首先运行;参见 [restamp_command](#restamp-command)。 |
| `migrations` | `{ dir, id }` 或 `null` | `null` | 迁移目录和具有文件名中数字的正好一个捕获组的正则表达式(`"^v(\\d+)-"`);`gstack-pregate regate` 报告 `MIGRATION_COLLISION`。 |

包含,由 `validate` 检查并在写入时再次检查:路径和 globs 是仓库相对的,没有 `..` 段,长度小于 256 个字符,每个列表最多 64 个,glob 的正则表达式扩展长度小于 512 个字符;`release_tool` 长度小于 512 个字符且没有换行;通过符号链接解析到仓库外部的匹配文件被拒绝(`POLICY_CONTAINMENT`)。镜像可以以 `..` 开头(同级 checkout),但内部没有 `..`;镜像写入仅限于该目录。

## gstack 自己的文件

gstack 的策略(`.gstack/ship-policy.json`,在否则被忽略的 `.gstack/` 中与 `generated.json` 和 `pregate.json` 一起跟踪)在 `stamp-at-merge` 上运行仓库:版本分配一次,当 PR 在队列中下一个时,每个提升都是 PATCH,`/review` 从文件读取维护者集。它命名一个发布工具 `gstack-version-bump --regen-digest` 用于硬编码:

```json
{
  "bump": "patch",
  "queue_mode": "stamp-at-merge",
  "release_tool": "bun scripts/gen-agents-digest.ts",
  "release_outputs": ["agents-digest/gstack-AGENTS.md"],
  "stamp_paths": [],
  "changelog": "CHANGELOG.md",
  "history": "squash-ok",
  "stamp_sensitive_gates": ["tests"]
}
```

`.gstack/ship-policy.example.json` 显示具有发布脚本、docs 提及和镜像的串行合并仓库将设置的每个键。

### gstack 的队列端程序

谁在 `stamp-at-merge` 下做什么,PR 作者和运行合并队列的线程:

1. **Open。** 分支携带基础版本加 PATCH(`gstack-version-bump write --version <base+patch> --regen-digest`)的暂定标记,并且其 CHANGELOG 部分直接在 `# Changelog` 下以该版本为标题;`## [Unreleased]` 也被接受。没有任何内容被声明,并且不咨询 `gstack-next-version`;暂定版本仅是使常见情况免费的猜测。CI 在此 head 上正常运行。
2. **Register。** 队列记录 PR 及其预期顺序。
3. **Stamp when next。** 当当前 head 上每个检查为绿色且 `origin/main` 位于队列假定的 SHA 时,PR 所有者(永远不是队列)在分支上运行:
   `gstack-restamp --next --base main --allow-repo-commands --pr <n> --expect-base <main-sha> --expect-order <pr,...>`。
   当基础未移动超过暂定标记时,它打印 `RESTAMP: X -> X (noop)` 和 `FILES: (none)`,并且绿色 head 是没有进一步 CI 的合并候选项。当基础移动时,它标记 base + PATCH,重新将 CHANGELOG 放在基础顶部条目的上方,重新运行发布工具,打印 `STAMP_VERSION: <X>` 和标记差异,并在 `--expect-base` 被读取后基础或顺序移动时以 `QUEUE_STALE` 拒绝(未写入任何内容)。所有者提交并推送标记 commit,让 CI 在其上运行,并报告 `READY #<n> <sha> STAMP_VERSION=<X>`。
4. **Merge。** 队列在该确切 SHA 上进行验证:`VERSION` 等于 `STAMP_VERSION`,CHANGELOG 的顶部标题是 `[<X>]`,其下方的标题是基础当前版本,该 SHA 上的每个检查为绿色,并且当推送标记 commit 时,它仅更改 `VERSION`、`package.json`、`CHANGELOG.md` 和发布输出(`gstack-tree-receipt --gated <green-head> --current <sha>` 打印每个文件的 verdict)。然后它使用合并 API 的 expected-head `sha` 设置为该 SHA 进行 squash 合并,因此服务器的延迟推送被拒绝。"VERSION 高于主分支"被"VERSION 等于重新标记的 STAMP_VERSION 在合并的 SHA 上,并且该版本在合并时为 base + PATCH"取代。
5. **Not next。** 等待另一个 PR 之后的 PR 可以在本地预先 gate,使用 `gstack-restamp --after <predecessor-pr>`;它不推送任何内容。当前身落地时,等待的 PR 合并 `origin/main`(永远不是 rebase),将其自己的 CHANGELOG 部分保留在新合并的部分之上,并运行步骤 3。

在 `origin/main` 上使用此策略之前,`gstack-restamp --next` 读取基础的 `bump: ask` 并回退到 MICRO bump;如果其基础早于此策略,则在翻转后第一次标记的 PR 上传递 `--bump patch`。

<a id="ship"></a>
## `/ship` 从中读取什么

`/ship` 步骤 3 运行 `show-effective --base <base>` 并遵循其 `ship:` 行,这些行携带 skill 散文原本会阐明的工作流后果:

| 行 | 效果 |
|---|---|
| `ship: history merge-only: …` | 永远不要 rebase、squash 或 force-push;`check-history --plan <merge\|squash\|rebase>` 拒绝重写计划(`HISTORY_POLICY_VIOLATION`,退出 3) |
| `ship: queue_mode stamp-at-merge: …` | 当 PR 是下一个时,步骤 12 项目 1-4 被 `gstack-restamp --next --base <base> --pr <n> --expect-base <sha>` 取代,当不是下一个时,被 `--after <predecessor-pr>` 取代;restamp 自己的 `next:` 行说明后续内容(NEW_VERSION、CHANGELOG 条目文本仅、合并的预期 head) |
| `ship: gate-ahead: …` | 使用 `.gstack/tmp/gate-ahead.json`,步骤 16 仅重用 `gstack-tree-receipt … --json > .gstack/tmp/tree-receipt.json` 标记为可重用的 lanes;stamp-sensitive 和 not-eligible lanes 实时运行;`--force-gate <reason>` 是协调者的调用 |
| `ship: bump patch: …` | 步骤 12 从不询问 MINOR 或 MAJOR;按大小的 PATCH 或 MICRO |
| `ship: preregistration: …` | 推送前,`gstack-evidence ancestor --from-policy --base <base>` 必须仅打印 `ANCESTOR: ok` |
| `ship: release_tool: …` | 该命令仅在 `gstack-restamp` 内运行(暂存 worktree,输出限于 `release_outputs`) |

PR 正文的第一个部分是 ship receipt(`gstack-ship-receipt write … --body`),在正文组合后由 `/ship` 的 pr-body 部分写入。

<a id="stamp-at-merge"></a>
## `queue_mode: stamp-at-merge`

在 `stamp-at-merge` 下,当 PR 打开时什么都不会被声明。当 PR 是下一个时,标记从那时基础分支的版本发生,通过仓库自己的发布工具,在每个标记位置一次完成:

```bash
gstack-restamp --next --base main --allow-repo-commands --pr 42 \
  --expect-base <sha of origin/main the queue assumed> --expect-order 40,41,42
```

打印 `ship, next in queue at <version>`、`STAMP_VERSION: <version>`、每个阶段一行、仅限于标记文件的差异、`MERGE_CONDITION: sha=<head>`(将其传递给合并 API 的 expected-head `sha`,以便 check-to-merge 竞争被服务器拒绝)和 `GSTACK_RESULT:` 行。`claims: ignored (stamp-at-merge)` 表示未咨询旧声明读取器。当基础或顺序移动时,结果是 `QUEUE_STALE`(退出 3)以及确切的 `recompute` 命令;未写入任何内容。

阶段,每个记录在 `.gstack/tmp/restamp-journal.json` 中:

0. **restamp_command**(仅当策略设置它时):仓库自己的 restamp,参见 [restamp_command](#restamp-command)。
1. **version**:在暂存 worktree 中(`VERSION`、清单、npm lockfiles)的 `gstack-version-bump write --version <X>`;当 `restamp_command` 拥有版本时跳过。
2. **release_tool**:策略命令,在下面的合约下。
3. **changelog**:PR 的 `[Unreleased]` 或旧版本部分移至 `<X>` 下的顶部,位于其前任条目的上方。重复的标题(`CHANGELOG_DUPLICATE_HEADING`)和不是 `<X>` 的顶部条目(`CHANGELOG_TOP_MISMATCH`)是错误。
4. **stamp_paths**:精确的 `(was X)`、`since: X` 和 `vX` 重写,以及每个 `stamp_patterns` 模板,在 `stamp_paths` 和 `stamp_files` 内。`X` 是分支自己的暂定标记(当它移动超过基础时的其 VERSION),否则 `--was <X>`,否则未标记分支将采用的下一个版本(`base + bump`);基础从未发布的版本的提及(在基础上不是 CHANGELOG 标题,不是标签)也被重写,因此将基础移动到分支下的队列重新排序仍会找到其 `since:` 字段。仅计算仓库自己的版本宽度,因此 4 位仓库中的 3 位依赖项 pin 永远不是候选。
5. **mirror**:tier 2;声明并验证,报告为未运行。
6. **apply**:每个更改的文件仅在所有阶段验证后写入分支,每个都针对指纹原始图像;同时编辑的文件是 `RESTAMP_CONFLICT`,应用的写入回滚(回滚拒绝覆盖字节与记录的后图像不同的文件)。

幂等:具有相同版本的第二次运行是 `noop`。`--dry-run` 打印完整差异并且不写入任何内容(甚至不写入日志)。`--version <X>` 精确标记 X;`--bump` 覆盖级别;`--queue-mode` 覆盖策略。

### Gate-ahead:`--after <pr>`

仅当 gate 可以在前身合并之前运行时,重新排序才便宜。`gstack-restamp --after <pr> --predecessor-ref <ref>` 在一次性 worktree(`.gstack/tmp/gate-ahead-worktree`)中合并 HEAD 和前身的 head,仅解析版本文件和 CHANGELOG-ordering 冲突,在那里相对于前身的版本标记 N+1,提交,并记录 `.gstack/tmp/gate-ahead.json`(`predecessor_head`、`branch_head`、`gated_head`、`gated_tree`、`version`、`evidence_branch_key`)。PR 分支的历史永远不会被触及,因此 squash-history PR 保持干净。gate 在该 worktree 中运行;`gstack-evidence bundle` 在那里导出证据。

<a id="release-tool-contract"></a>
### release-tool 合约

发布工具在从工作树的内容指纹(`gstack-wtree`)植入的隔离暂存 worktree 中运行,并链接仓库被忽略的 `node_modules`。其输出被枚举:版本文件、lockfiles、CHANGELOG、`stamp_paths` 和 `release_outputs`。在该集合之外的写入在任何内容到达分支之前拒绝整个 restamp(`RESTAMP_OUTPUT_OUTSIDE_ALLOWED`);非零退出是 `RESTAMP_RELEASE_TOOL_FAILED`。每次写入的原始图像和后图像都记录在日志中。固定命令字符串不会冻结其实现(`bun run release:restamp` 仍然运行 PR 的 package script),因此收据命名策略修订、命令、其工作目录和信任范围(`policy: base origin/main@<sha>` 或 `head (unreviewed)`)。

<a id="restamp-command"></a>
### `restamp_command`:仓库拥有标记

其自己的工具知道其标记位置的仓库设置 `restamp_command`。`gstack-restamp` 在其暂存 worktree 中作为第一阶段运行它,这是 HEAD 的干净 checkout,分配的版本作为最后一个参数,在 `GSTACK_RESTAMP_VERSION` 中。调用者的 HEAD、index 和 worktree 必须首先是干净的(`RESTAMP_CONFLICT`,未写入任何内容);通用路径保留其工作树指纹种子。命令在暂存 worktree 中暂存或提交的任何内容都会折回为普通编辑;然后输出限制适用于发布工具(`RESTAMP_OUTPUT_OUTSIDE_ALLOWED`),版本文件必须包含目标版本(`RESTAMP_VERSION_SOURCE`),非零退出是 `RESTAMP_RELEASE_TOOL_FAILED`。在任何失败中,没有任何内容到达分支。跳过 gstack 的版本 writer;CHANGELOG 和 `stamp_paths` 阶段仍然运行(当工具已经完成工作时是无操作的)。从基础分支读取,仅使用 `--allow-repo-commands` 运行,例如 `release_tool`。获取、合并基础分支或需要其自己选择的版本的工具无法在此处运行;给它一个采用版本并仅写入标记的模式。

<a id="tree-receipt"></a>
## tree receipt

```bash
gstack-tree-receipt --from .gstack/tmp/gate-ahead.json --predecessor-pr 41 --bundle evidence-bundle.json
gstack-tree-receipt --gated <sha> --current HEAD --json
```

比较两个**已提交的 head**(工作树指纹可以命名一旦 gate VM 消失就不可达的对象)并打印 gated 和 current head、两个 tree id、版本对、gate 假定的前身、仅限于标记文件的差异、每个文件一行及其规则和 verdict、每个 gate 一行,然后是两个 verdict 行:

```
TREE: same-modulo-stamps | changed (<residual paths>)
gate-reuse: eligible (reusable: tests; rerun: digest) | not-eligible (<what changed>) — fix: <rerun command> (GATE_REUSE_NOT_ELIGIBLE)
```

Hunks 按文件类型分类:版本文件(仅 old → new)、清单(仅 `version` 键)、npm 和 bun lockfiles(仅根 `version` 键,其中 `gstack-version-bump` 写入;依赖项条目是 `changed`)、CHANGELOG(标题移动或重新设置标题,作为顺序无关的多重集条目进行比较)、`stamp_paths` 和 `release_outputs`(每个删除/添加对必须是精确的 `(was X)` / `since: X` / `vX` 重写,其中 X 是 gated 版本,替换是当前版本,因此恰好等于旧版本的依赖项 pin 是 `changed`;`stamp_patterns` 模板也计为标记形式)。设置 `stamp_files` 时,其外的文件是 `changed (stamp_files: not in the repo's stamp_files allowlist)`,无论其 diff。任何其他字节都是更改。

等效性按 gate 决定。`stamp_sensitive_gates` 中的 lane 始终 `rerun`;仅当树是 same-modulo-stamps 且 bundle 中的执行标识等于当前标识时,验证的 lane 才 `reusable`:Bun 和 Node 版本、OS 和 arch、每个 lockfile 的标识(根版本规范化掉)、`.github/workflows` tree id、运行时 pin 文件和 `package.json#engines`,以及任一侧知道时的 runner 镜像。没有 bundle,标识未知,收据是 `not-eligible`。当记录的前身 head 不再是前身 PR 的 head(通过 `--predecessor-ref` 或 `gh pr view --predecessor-pr` 检查)时,收据打印 `PREDECESSOR MOVED <old> -> <new> — fix: gstack-restamp --after <pr> again (RESTAMP_PREDECESSOR_MOVED)` 并拒绝 `eligible`。

`/ship` 步骤 16 咨询收据;当它说 `eligible` 时,重复的完整 gate 需要 `--force-gate <reason>`,协调者进行调用。

### 证据 bundle

`gstack-evidence` 按设计是机器本地的,因此可移植的形式是 bundle:

```bash
gstack-evidence bundle --out evidence-bundle.json --label tests --label e2e --require tests,e2e \
  --selection tests='bun run test' --run-url tests=https://github.com/.../actions/runs/1
```

Bundle 绑定到已提交的 head(`repo.commit`、`repo.tree`)并携带比较材料(版本源、CHANGELOG 路径、`stamp_paths`、`release_outputs`)、上述执行标识、必需的 lane 清单,以及每个 lane 的命令、`cmd_sha256`、退出、时间戳、执行选择、运行 URL、日志哈希以及它是否 `verified`(由账本在此 head 的干净已提交树上生成)或不是(脏树、其他提交或 `--import`ed 断言)。新的协调器机器在 gate VM 消失后使用 `gstack-tree-receipt --bundle` 读取它;当 bundle 在另一棵树上生成或缺少标识时,收据拒绝重用(`EVIDENCE_IDENTITY_UNKNOWN`)。

<a id="ship-receipt"></a>
## ship receipt

交接记录是一个围栏块,标识符优先,根据 `gstack-artifact schema ship-receipt` 进行验证:

````
```gstack-ship-receipt
{ "schema_version": 1, "pr": 42, "head": "<sha>", "version": "1.2.7.0", "base": "main",
  "gated_tree": "<tree>", "predecessor": "<sha>", "tree": "same-modulo-stamps",
  "gate_reuse": "eligible (reusable: tests; rerun: digest)", "gate": { "tests": { "pass": 10, "fail": 0, "skip": 1 } },
  "ci": { "checks": 3, "failed": 0 }, "spend_usd": 0.42, "session_kind": "unattended",
  "artifacts_consumed": "1/1", "run": "<run id>", "thread": "<thread id>", "policy": "base origin/main@<sha>" }
```
````

`gstack-ship-receipt write --pr <n> --receipt <tree-receipt.json> --gate '<json>' --ci '<json>' --body <pr-body.md>` 将其设为 PR 正文的第一部分(`/ship` 步骤 18),并打印 `SHIP_RECEIPT: pr=… head=… version=…` 作为完成消息的第一行。`/land-and-deploy` 使用 `gstack-ship-receipt read --pr <n> --json` 读取它(`RECEIPT_MISSING`、`RECEIPT_INVALID`)。

### 采用普查(P7)

```bash
gstack-ship-receipt census --repo garrytan/gstack --since v1.91.70.0 --cohort threads.txt
```

通过 `gh` 读取自标签以来的合并 PR,解析每个收据,通过 `run` 和 `thread` 进行去重,并打印 `CENSUS: … unattended=<n> consumed=<n> adoption=<consumed threads>/<cohort> durable=yes`。队列文件每行列出一个线程 ID(从不调用 gstack 的线程计入分母);没有它,分母是收据命名的线程。

## 历史策略和预注册(C7)

在 `history: merge-only` 下,`/ship` 拒绝 squash 或 rebase 计划并合并。

[desktop write output truncated: original_bytes=27713; if the omitted remainder is needed, continue with read using offset=330 and the same path.]
