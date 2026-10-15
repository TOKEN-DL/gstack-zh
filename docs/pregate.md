# Pre-gate:生成的文件、廉价的检查、问题链接

`/ship` 步骤 5 以 `gstack-pregate` 开头:多智能体 wave 一直付费远程 gate 以发现的免费检查(过时的生成文件、仓库根目录下的临时脚本、仍然断言旧常量的仅夜间测试、其 lane 未运行而合并的仅 Windows 更改、`| tail` 隐藏的失败)。计划:`docs/designs/MULTI_AGENT_WAVE_2026_10_10.md` C4、C6、C8。结果代码:[docs/troubleshooting.md](troubleshooting.md#pre-gate-generated-file-registry-gstack-pregate-issue-links)。

三个 bin,都在 `lib/` 之上:

| Bin | Pin 文件(从 `origin/<base>` 读取) | 作用 |
|---|---|---|
| `gstack-regen` | `.gstack/generated.json` | 生成的文件重新生成,永远不手动合并 |
| `gstack-pregate` | `.gstack/pregate.json` | 两阶段廉价 gate,`pregate.json` 记录,发布检查 |
| `gstack-issue-links` | — | `Fixes #N` 仅在显式关闭意图上 |

## 信任边界

Pin 文件命名命令。它的存在永远不会授权执行:两个工具都从 `origin/<base>` 上提交的副本读取(PR 不能将命令混入其自己的 gate),并且仅使用 `--allow-repo-commands` 运行命令,而 `/ship` 传递该命令。独立运行,`gstack-regen check` 列出命令并退出 3(`REPO_COMMANDS_NOT_ALLOWED`);`gstack-pregate` 将 `regen` 和 `lanes` 标记为 `incomplete`,永远不是 `pass`。`--policy-from head`(引入或更改 pin 文件的 PR)仅在 `--allow-repo-commands` 下被接受,并且记录显示 `policy head (unreviewed)`。不可解析的基础以退出 3 和 `POLICY_SOURCE_UNAVAILABLE` 退出。Globs 是仓库相对的(没有 `..`,没有绝对路径,每个列表最多 64 个);命令是 512 字节以下的单行。

<a id="generated"></a>
## `.gstack/generated.json`(generated-v1)

```json
{ "$schema": "https://github.com/garrytan/gstack/docs/pregate.md#generated-v1",
  "entries": [ { "id": "skill-docs", "command": "bun run gen:skill-docs --host all",
                 "outputs": ["*/SKILL.md", "*/sections/*.md"], "inputs": ["*/SKILL.md.tmpl", "hosts/**"] } ] }
```

- `gstack-regen check` 在临时 `git worktree`(从工作树植入,链接 `node_modules` 而不是复制)中重新生成每个条目(或 `--only <id>`,或 `--touched <paths>` 用于输入或输出相交的条目),并报告过时的输出(`REGEN_STALE`,退出 1)而不触及树。没有 `command` 的条目仅被跟踪(黄金由另一个条目重新生成)。
- `gstack-regen write` 重新生成到位:`gstack-restamp` 和步骤 3 冲突规则使用的内容。在已注册输出上发生冲突时,选择任一侧,然后运行 `gstack-regen write`;智能体永远不会手动合并生成的文件。
- `list`、`validate`、`init [--explain] [--force]`。

gstack 的条目:`skill-docs`(每个主机渲染、智能体摘要、llms.txt、doctor 组件表)和 `ship-goldens`(`bun scripts/gen-ship-goldens.ts`,`test/fixtures/golden/` 中的字节固定的 `/ship` 渲染)。

<a id="pregate-v1"></a>
## `.gstack/pregate.json`(pregate-v1)

| 键 | 含义 |
|---|---|
| `runner` | 带有 `{files}` 的本地测试命令(gstack:`bun test {files}`);没有它 `lanes` 是 `incomplete` |
| `dependencies` | glob → 子进程目标的测试文件、运行时读取和导入图看不到的生成输入;`[]` 声明"不需要 lane"(文档、许可证) |
| `selection` | `workflow.yml/job` → 打印 lane 文件列表的命令,用于规划器计算的矩阵(gstack:`scripts/test-free-shards.ts --list`) |
| `warn` | 降级为 warn 的检查 ID;`secrets` 和必需的 lane 永远不是 |
| `strays.allow` | 故意根文件或新的顶级目录 |
| `guards` | `[{ id, when: [globs], command, fix?, timeout_s? }]`:当触摸的文件匹配 `when` 时,车道必须传递的仓库命令(清单检查、导入图看不到的子进程黄金);参见 [guards](#guards) |
| `serial_patterns` | 测试文件 glob,`lanes` 检查在单元文件之外对每个文件运行一个进程(默认 `**/*.serial.test.*`) |
| `sealed` | 必须永远到达公共目的地的密封或评估输出;由 pre-push hook 和 `gstack-pregate sealed` 读取(参见 [sealed output](#sealed)) |
| `price_file` | 付费 gate 的调用计划数据文件的仓库相对路径(`GSTACK_EVAL_CALLS v1`);`gstack-eval-plan price-gate` 从中定价并且永远不会执行任何东西([docs/eval-calls.md](eval-calls.md)) |

仅尊重 `origin/<base>` 副本,因此降级在生效之前会被审查。`gstack-pregate validate` / `init` 涵盖工作树文件。

## `gstack-pregate`

```
gstack-pregate [--stage preflight|tests|all] [--only <check,..>] [--allow-repo-commands] [--policy-from base|head] [--explain] [--json]
gstack-pregate verify | clear --remote <workflow.yml/job> --run-url <url> | lanes | validate | init
```

两个阶段具有单独的时钟,都打印:

**阶段 1,机械预检**(两分钟承诺;gstack 测量 `regen` 约 2.5 秒,每个其他约 200 毫秒):

- `regen`:输入或输出被触及的每个注册表条目都是新鲜的(临时 worktree 中的 `gstack-regen check`)。
- `secrets`:在分支的添加行上的 `gstack-redact`(merge-base 到工作树,包括未跟踪的文件)。HIGH,或 `secret` 类别中的 MEDIUM 失败;其他 MEDIUM 发现警告。无法降级。
- `strays`:仓库根目录下的新文件、新的顶级目录或任何地方的临时名称(`zz-*`、`tmp*`、`scratch*`)。
- `literals`:对于每个更改的导出常量或命名超时/重试文本,整个测试树(包括仅夜间和仅 Heavy 文件)被搜索旧值;分支未更新的消费者失败,并显示 `file:line`。
- `guards`:`.gstack/pregate.json` 中每个其 `when` glob 匹配被触摸的文件(包括重命名)的 guard 在其自己的进程中使用已验证的启动 Bun 运行其 `command`,并具有自己的 `timeout_s`(默认为 `--timeout`)。非零退出失败 pregate,命名 guard 及其 `fix`(`PREGATE_GUARD_FAILED`);超时是 `incomplete`;没有 `--allow-repo-commands`,选定的 guard 被列出并且检查是 `incomplete`。Guard 从基础分支的副本中读取,无法降级为 warn。参见 [guards](#guards)。

**阶段 2,测试执行:**

- `lanes`:触摸的文件通过 `lib/pregate/import-graph.ts` 映射到测试(静态导入 ∪ 仓库路径字符串文字 ∪ `bin/<name>` → `test/<name>*.test.ts` 约定 ∪ `dependencies`),通过非测试文件传递;删除和重命名的文件选择命名它们的测试。没有 lane 的被触摸文件是 `FAIL lanes — <file> has no test lane`,永远不是 pass。Lanes 和平台 tier 来自 `.github/workflows/*.yml`(`gstack-pregate lanes` 打印它们):本地运行器在 `set -o pipefail` 下运行,在运行器摘要行上判断(零测试运行是失败,`| tail` 隐藏的退出仍然是退出);此机器无法运行平台 lane 成为 `requires-remote` 义务,命名 `workflow.yml/job`。`--explain` 打印每个测试的来源和每个不确定的映射(没有 `selection` 清单的规划器计算的矩阵保守地选择)。匹配 `serial_patterns`(默认 `**/*.serial.test.*`)的测试文件在单元文件之后对每个文件运行一个进程。运行器和每个 guard 首先在 PATH 上使用已验证的启动 Bun 运行:`$BUN_INSTALL/bin` 和 `~/.bun/bin` 中的第一个,其 Bun 满足 gstack 的和项目的 `engines.bun`(该行打印 `runtime: bun <v> from <dir>`),因此 Capy 主机的较旧镜像 Bun 无法翻转 verdict。

每个检查记录它运行的输入标识(前后的工作树指纹、选择、命令哈希);在检查期间移动的树、超时或缺少工具是 `incomplete`,永远不是 pass。记录是 `.gstack/tmp/pregate.json`(`gstack-artifact validate --as pregate`)。

**发布。** `gstack-ship-receipt write --pregate .gstack/tmp/pregate.json`(步骤 18)除非记录是当前用于发布的树(`PREGATE_STALE`:在最终标记后重新运行)、绿色(`PREGATE_LANE_FAILED`、`PREGATE_INCOMPLETE`)和每个 `requires-remote` 义务都带有收据(`PREGATE_REMOTE_UNCLEARED`:调度作业,然后 `gstack-pregate clear --remote <workflow.yml/job> --run-url <url>`),否则拒绝。然后 ship receipt 携带 `pregate` 和 `requires_remote`。`gstack-pregate verify` 独立运行相同的检查。

未知标志以退出 2 退出,每个动词都带有用法(被告知运行 gstack 缺少的标志的智能体永远不会得到它没有要求的绿色)。仍要完成的 Tier 2 检查(`ratchets`、`lint`、`links`、`hermetic`、`patches`、`integrator`)在 `--only` 的帮助中命名。

<a id="guards"></a>
## Guards 和 lane-done 规则

协调器下的 lane 在其分支上运行 `gstack-pregate run --allow-repo-commands`,然后报告完成,其交接携带 `PREGATE:` 行(协调器合约是这样逐字说的)。单元 tier 涵盖更改文件的测试;guards 涵盖导入图看不到的仓库机械 gate:

```json
{ "guards": [
    { "id": "serial-manifest", "when": ["test/**/*.serial.test.ts", "scripts/serial-files.tsv"],
      "command": "bun test test/scripts/serial-files.test.ts",
      "fix": "add the file's row to scripts/serial-files.tsv" } ] }
```

`dependencies` 仍将 glob 映射到 lanes 检查选择的额外 *tests*;guard 是具有自己的 fix 行的 *command*。较旧的 gstack 拒绝未知键(`PREGATE_CONFIG_INVALID`),因此仓库仅在读取它们的 gstack 发布后才提交 `guards`。已处理的 gbrain 配置位于 [gbrain worked configs](#gbrain-worked-configs)。

<a id="sealed"></a>
## 密封输出:pre-push hook 和 `gstack-pregate sealed`

```json
{ "sealed": {
    "paths": ["heldout/**"],
    "transcripts": ["runs/**/transcript*.jsonl"],
    "answer_markers": ["SEALED ANSWER:"],
    "seed_patterns": ["shuffle_seed=\\d+"],
    "local_paths": true } }
```

具有 `sealed` 键的仓库在托管的 pre-push hook(`gstack-redact install-prepush-hook`,无需重新安装)或独立预检运行时获得针对意外发布的配置本地检查:

```
gstack-pregate sealed [--range <a..b>] [--remote origin] [--base <branch>] [--json]
```

- **边界。** 推送发布的每个提交:`git rev-list <local sha> --not --remotes=<remote>`(以及 ref 存在时的远程提示);没有 refs 的目的地意味着可达的所有内容。`--range` 显式命名提交。
- **规则。** `sealed.path` 和 `sealed.transcript` 检查提交添加、重命名或复制的每个路径,无论其内容(二进制、空、在以后的提交中再次删除);`sealed.answer`、`sealed.seed` 和 `sealed.local_path`(`/home/<user>/`、`/Users/<user>/`、`C:\Users\<user>\`)检查每个提交的添加行。发现打印规则、`file[:line]` 和提交,永远不打印文本。
- **策略。** 目的地基础分支的 `sealed` 和推送的提交的并集:推送可以收紧它,永远不能放松它。格式错误或不可读的策略会被拒绝。仅来自基础分支的 `.gstack/sealed-allow.tsv`(`rule<TAB>path<TAB>sha<TAB>reason`)的豁免被接受,并在每次推送时列出。
- **可见性。** 来自基础分支的 `sealed.visibility`,否则是状态根下的缓存(`sealed-visibility.json`:`public` 答案为 24 小时,`private` 答案为 1 小时,然后才能将拒绝转为警告),否则 `gh repo view --json visibility`(5 秒)。未知、不可读或新创建的计为公共。公共拒绝(`SEALED_OUTPUT_REFUSED`,退出 1);私有警告。

Capy 的 pull-request 工具可能不运行本地 git hooks,因此智能体在任何具有 `sealed` 配置的仓库中的任何推送或 PR 创建之前运行 `gstack-pregate sealed`(协调器合约是这样说的)。公共仓库上的托管 CI 仅在上传后检测绕过。

<a id="regate"></a>
## `gstack-pregate regate`:gate 是否仍然覆盖 HEAD

```
gstack-pregate regate --against <gated-head> --bundle <evidence-bundle.json> [--require <lane,...>]
                      [--pr <n>] [--from <gate-ahead.json>] [--predecessor-pr <n> | --predecessor-ref <ref>]
                      [--base <branch>] [--json]
gstack-pregate --against <gated-head> ...      the same verb
```

它现在评估树收据(永远不是存储的)在 gated head 和 HEAD 之间,在基础分支的发布策略下,因此仓库的 `stamp_files` 和 `stamp_patterns` 决定什么是标记([ship-policy](ship-policy.md#tree-receipt))。重用需要在 gated 树上生成的证据 bundle、相同的执行标识,以及仍然是记录的前身 head(从 `--from` 读取,默认 `.gstack/tmp/gate-ahead.json` 当它命名此 gated head 时)。它打印每个不同的文件(具有第一行的非标记差异),然后:

```
STAMP_ONLY: yes|no files=<n> [changed=<n>]
LANE: <label> reusable|rerun|not-reusable (<reason>)      one per bundle lane and per --require lane
MIGRATION_COLLISION: none (...) | unknown (...) | <id> <file> vs <origin/main|PR #n> <file>
REGATE: not-needed | needed (<every reason>)
```

`REGATE: not-needed`(退出 0)仅当树仅为 stamp,每个 bundle 中的 lane 和 `--require` 都是可重用的(stamp-sensitive lane 重新运行,因此它需要 regate),前身 head 是已知的且未移动,以及迁移查找完成且没有冲突。任何其他内容都是 `needed`(退出 1);没有 bundle 打印 `REGATE: needed (no gate evidence for <head> ...)`。

迁移来自发布策略的 `migrations`(`dir`、`id` 正则表达式具有一个捕获组)。该分支在 `dir` 下添加的数字与基础分支的树以及每个其他打开的 PR(`gh pr list --state open --limit 1000 --json number,headRefOid,files`;PR 本身按 head sha 或 `--pr` 跳过)进行比较。列出 100 个文件的 PR 被 GitHub 截断,因此其完整列表使用 `gh api --paginate` 获取。失败的查询、列出的 1000 个 PR 或无法获取的文件列表是 `MIGRATION_COLLISION: unknown (...)`,它阻止 `not-needed`([`MIGRATION_COLLISION`](troubleshooting.md#migration-collision))。

<a id="gbrain-worked-configs"></a>
## gbrain worked configs

gbrain 在读取新键的 gstack 发布后在 master 上提交这些(较旧的 gstack 拒绝未知键)。其 `.gitignore` 当前忽略 `.gstack/*`,因此该更改还添加 `!.gstack/pregate.json` 和 `!.gstack/ship-policy.json`;这两个更改都是 gbrain 的。

`.gstack/pregate.json`:guards 用于导入图看不到的 gate。

```json
{
  "dependencies": {
    "test/**/*.serial.test.ts": ["test/scripts/serial-files.test.ts"],
    "scripts/serial-files.tsv": ["test/scripts/serial-files.test.ts"],
    "src/cli.ts": ["test/cli-goldens.test.ts"],
    "src/commands/**": ["test/cli-goldens.test.ts"],
    "src/core/operations*.ts": ["test/cli-goldens.test.ts"]
  },
  "guards": [
    { "id": "serial-manifest", "when": ["test/**/*.serial.test.ts", "scripts/serial-files.tsv", "scripts/run-serial-tests.sh"],
      "command": "bun test test/scripts/serial-files.test.ts",
      "fix": "add the serial file's row to scripts/serial-files.tsv" },
    { "id": "cli-goldens", "when": ["src/cli.ts", "src/cli/**", "src/commands/**", "src/core/operations*.ts", "src/core/ops/**"],
      "command": "bun test --timeout=60000 test/cli-goldens.test.ts test/cli-dispatch-phase.test.ts test/export-surface-golden.test.ts",
      "fix": "bun run regen:all, then justify each changed golden in the PR body" },
    { "id": "regen-check", "when": ["src/**", "skills/**", "templates/**", "plugin/**"],
      "command": "bun scripts/regen-all.ts --check",
      "fix": "bun run regen:all and commit the regenerated files" },
    { "id": "migration-order", "when": ["src/core/schema-migrations/**"],
      "command": "bun scripts/check-schema-migration-order.ts",
      "fix": "renumber the migration to the next free number on master" }
  ]
}
```

`.gstack/ship-policy.json`:标记规则和迁移。在从 0.60.188.0 到 0.60.189.0 的 gbrain master 树的纯重新标记上,树收据将所有 16 个更改的文件分类为该策略下的标记;`src/core/behavior-change-notice.ts` 中的添加行仍然是更改。

```json
{
  "bump": "patch",
  "queue_mode": "stamp-at-merge",
  "changelog": "CHANGELOG.md",
  "stamp_files": [
    "VERSION", "package.json", "CHANGELOG.md", "openclaw.plugin.json",
    ".claude-plugin/plugin.json", ".codex-plugin/plugin.json",
    "plugin-variants/*/.claude-plugin/plugin.json", "plugin-variants/*/.codex-plugin/plugin.json",
    "plugin-variants/*/README.md", "plugin/README.md",
    "BOOTSTRAP_FOR_AGENTS.md", "templates/bootstrap/template-repo/README.md",
    "src/core/behavior-change-notice.ts"
  ],
  "stamp_patterns": [
    "\"version\": \"{version}\"",
    "since: '{version}'",
    "<!-- gbrain-plugin-tree-stamp: {version} -->",
    "<!-- gbrain-template-stamp: {version} -->",
    "<!-- gbrain-runbook-stamp: {version} -->"
  ],
  "migrations": { "dir": "src/core/schema-migrations", "id": "^v(\\d+)-" }
}
```

gbrain 的 `bun run release:restamp` 获取 master,选择其自己的版本,拒绝脏树并提交,因此它不能作为 `restamp_command`,因为它存在;上面的标记规则让 gstack 自己的 restamp writer 完成工作。采用版本并仅写入标记的 gbrain 模式将使它成为一个。

gbrain-evals `.gstack/pregate.json`:密封路径。下面的 glob 是 guard 读取的形状的占位符;gbrain-evals 填充其自己的 held-out 和脚本布局。

```json
{ "sealed": {
    "paths": ["heldout/**"],
    "transcripts": ["runs/**/transcript*.jsonl"],
    "answer_markers": ["SEALED ANSWER:"],
    "seed_patterns": ["shuffle_seed=\\d+"],
    "local_paths": true } }
```

<a id="issue-links"></a>
## `gstack-issue-links`

```
gstack-issue-links [--base <ref>] [--issue <n>]... [--ci-run <id>]... [--out issues.json] [--json]
```

读取 `origin/<base>..HEAD`。`Fixes #N` 仅在显式关闭意图上:`Fixes`/`Closes`/`Resolves #N` 预告行或 `--issue N`。裸提及、部分修复、延迟询问以及每个跨仓库 `owner/repo#N` 都是 `Refs`。每个关闭问题通过 `gstack-issue-guard` 读取;无法读取的是 `Refs #N (unreadable …)`,带有 `ISSUE_UNREADABLE`(退出 1),永远不是 `Fixes`。每个 `--ci-run <id>` 在其主体命名它的打开问题中搜索,并呈现 `Flake: #N (run <id>)`。打印的行进入 PR 正文中的 `## Issues`;`--out` 写入 `gstack-ship-receipt write --issues @file` 嵌入的对象。
