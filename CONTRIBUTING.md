# Contributing to gstack

感谢你愿意让 gstack 变得更好。无论你是在技能提示中修一个错别字,还是在构建一套全新工作流,本指南都能让你快速上手。

## Quick start

gstack 技能是 Claude Code 从 `skills/` 目录发现的 Markdown 文件。它们通常位于 `~/.claude/skills/gstack/`(你的全局安装)。但当你开发 gstack 本身时,你希望 Claude Code 使用*你工作树中*的技能——这样编辑无需复制或部署即可立即生效。

这就是 dev mode 所做的事。它把你的仓库符号链接到本地的 `.claude/skills/` 目录,让 Claude Code 直接从你的 checkout 读取技能。

```bash
git clone https://github.com/garrytan/gstack.git && cd gstack
bun install                    # install dependencies
bin/dev-setup                  # activate dev mode
```

> **完整克隆 vs 浅克隆。** README 中的用户安装使用 `--depth 1` 以加快速度。作为贡献者,请使用完整克隆(不要 `--depth` 标志)——你需要历史来进行 `git log`、`git blame`、`git bisect`,以及对照早期版本评审 PR。如果你已经有了按 README 克隆的 `--depth 1` 版本,用 `git fetch --unshallow` 提升为完整克隆。

### First free check (no API key, no browser)

```bash
bun install --frozen-lockfile
bun run typecheck        # expect no output and exit 0 (about a second)
bun run typecheck:test   # expect "test typecheck ratchet: N known diagnostics, none new."
```

`typecheck` 覆盖产品代码(`browse/src`、`lib`、`scripts`、`bin`、`hosts` 以及 `tsconfig.json` 中的其它条目),必须保持零错误。`typecheck:test` 让测试代码服从已提交的 `scripts/typecheck-test-baseline.json`:新增或重复的诊断会失败并指出文件名、TS 码与消息;修复诊断也会失败,直到你用 `bun run typecheck:test --write-baseline` 把更小的余量锁定进去。编辑 `lib/cso/*.ts`?在提交前跑 `bun run format:cso`;CI 跑 `format:cso:check`。这三者都在必需的 `free-tests` 检查中运行。

现在编辑任何 `SKILL.md`,在 Claude Code 中调用它(例如 `/review`),即可看到你的修改实时生效。当你开发完毕:

```bash
bin/dev-teardown               # deactivate — back to your global install
```

## Operational self-improvement

gstack 自动从失败中学习。每次技能会话结束时,代理会反思哪里出了问题(CLI 错误、错误方法、项目怪癖),并将运营学习记录到 `~/.gstack/projects/{slug}/learnings.jsonl`。未来的会话会自动浮出这些学习,因此 gstack 在你的代码库上会越来越聪明。

无需设置。Learnings 自动记录。通过 `/learn` 查看。

### The contributor workflow

1. **正常使用 gstack** ——运营学习会自动捕获
2. **检查你的 learnings:** `/learn` 或 `ls ~/.gstack/projects/*/learnings.jsonl`
3. **Fork 并克隆 gstack**(如果还没有)
4. **把 fork 符号链接进你撞见 bug 的项目:**
   ```bash
   # In your core project (the one where gstack annoyed you)
   ln -sfn /path/to/your/gstack-fork .claude/skills/gstack
   cd .claude/skills/gstack && bun install && bun run build && ./setup
   ```
   Setup 会为每个 skill 创建带 SKILL.md 符号链接的子目录(`qa/SKILL.md -> gstack/qa/SKILL.md`),并把每个 skill 的运行时资产链接到旁边(sections/、templates、checklists——除 SKILL.md、测试、构建输出和 `.tmpl` 源之外的一切),然后询问你的前缀偏好。传 `--no-prefix` 跳过提示,使用短名。
5. **修复问题** ——你的修改在这个项目里立即生效
6. **通过实际使用 gstack 来测试** ——做那个让你不爽的事,验证它已修复
7. **从你的 fork 开一个 PR**

这是贡献的最佳方式:在你做真实工作的项目里修 gstack,就在你真正感到痛的地方。

**CI 在 fork PR 上做什么。** GitHub 永远不会给 fork PR 这个仓库的 secrets,所以 CI 跑所有不需要 secrets 的:必需的 `free-tests` 检查(Linux 免费套件、typecheck、macOS 与 Windows gates)、Windows Free Tests、Skill Docs Freshness、Workflow Lint、Version Gate、Quality gate、make-pdf gate,以及你的变更触碰的任何 path-triggered gates。E2E Evals 仍会从你的 `Dockerfile.ci` 构建 CI 镜像而不发布它,并跳过需要 provider API keys 的付费 eval jobs。那个 skip 是预期的,且不会阻塞你的 PR。它也不是通过:维护者会在合并前从这个仓库的分支上跑付费 eval。第一次贡献者的运行会等待维护者批准。

### Session awareness

当你同时开着 3+ 个 gstack 会话时,每个问题都会告诉你项目、分支以及正在发生什么。再也不用盯着一个问题想「等等,这是哪个窗口?」格式在所有技能中保持一致。

## Working on gstack inside the gstack repo

当你在编辑 gstack 技能并想通过在同一仓库里实际使用 gstack 来测试它们,`bin/dev-setup` 会把这件事接好。它创建 `.claude/skills/` 符号链接(gitignored)指回你的工作树,所以 Claude Code 使用你的本地编辑而不是全局安装。

```
gstack/                          <- your working tree
├── .claude/skills/              <- created by dev-setup (gitignored)
│   ├── gstack -> ../../         <- symlink back to repo root
│   ├── review/                  <- real directory (short name, default)
│   │   └── SKILL.md -> gstack/review/SKILL.md
│   ├── ship/                    <- or gstack-review/, gstack-ship/ if --prefix
│   │   └── SKILL.md -> gstack/ship/SKILL.md
│   └── ...                      <- one directory per skill
├── review/
│   └── SKILL.md                 <- edit this, test with /review
├── ship/
│   └── SKILL.md
├── browse/                      <- /browse skill + gstack's own browser engine (the fallback)
│   ├── src/                     <- TypeScript source
│   └── dist/                    <- compiled binary (gitignored)
├── lib/
│   ├── aside-render.ts          <- local-HTML rendering: Aside first, browse engine fallback
│   └── design-catalog.ts        <- typed design anti-pattern catalog; review/design-checklist.md is generated from it
├── bin/
│   └── gstack-render.ts         <- the CLI skills call to render a local HTML file
└── ...
```

Setup 在顶层创建真实目录(不是符号链接),其中带有 SKILL.md 符号链接,并链接到每个 skill 的运行时资产(sections/、templates、checklists)。别名 skill(`_gstack-command`、`connect-chrome`)作为重写的副本安装,绝不作为符号链接——编辑符号链接的别名会破坏生成的源。这确保 Claude 将它们发现为顶层技能,而不是嵌套在 `gstack/` 之下。名称取决于你的前缀设置(`~/.gstack/config.yaml`)。短名(`/review`、`/ship`)是默认。运行 `./setup --prefix` 如果你偏好命名空间名(`/gstack-review`、`/gstack-ship`)。

## Day-to-day workflow

```bash
# 1. Enter dev mode
bin/dev-setup

# 2. Edit a skill template (SKILL.md files are generated — edit the .tmpl)
vim review/SKILL.md.tmpl
bun run gen:skill-docs   # or: bun run dev:skill (watch mode, auto-regen on change)

# 3. Test it in Claude Code — changes are live
#    > /review

# 4. Editing browse source? Rebuild the binary
bun run build

# 5. Done for the day? Tear down
bin/dev-teardown
```

### Brain-aware blocks in a dev workspace (gbrain installed)

如果 gbrain 已安装且可用(`bin/gstack-gbrain-detect --is-ok` 退出 0),`bin/dev-setup` 会保持你 tracked 的 `SKILL.md` 文件规范,并把脑感知变体(`GBRAIN_CONTEXT_LOAD` / `GBRAIN_SAVE_RESULTS` 块)渲染到 `.claude/gstack-rendered/`(gitignored,每个 workspace)。然后它把 workspace 的 `SKILL.md` 符号链接重新指向该渲染,所以你的 Claude 会话获得完整的 gbrain 体验,同时 `git status` 保持干净。在底层,dev-setup 内联传入 `GSTACK_SKIP_GBRAIN_REGEN=1` 给嵌套的 `./setup`(所以它从不弄脏 tracked 源)并跑 `gen:skill-docs:user --out-dir .claude/gstack-rendered`,它仅重写 section-base 路径以指向该渲染。`bin/dev-teardown` 删除该渲染。要让这些块在*其他*项目的 Claude 会话中保持活跃,跑 `gstack-config gbrain-refresh`,它把这些块渲染到用户 render 目录(`${GSTACK_USER_RENDER_DIR:-~/.gstack/render/claude}`,仅在成功渲染时换入)并通过 `gstack-relink` 把已安装的 skill 重新指向那里——全局安装 checkout 保持 git-clean,且 refresh 受保护,绝不触碰符号链接或非 gstack 目录。

## Testing & evals

Codex evals 与 GPT benchmark 适配器默认使用 `gpt-6-astra`:显式 model > `GSTACK_CODEX_MODEL` > 默认。Claude capture 与裁判默认使用 `claude-fable-5-1`,通过 `lib/eval-model.ts` 解析:

- Claude 会话、PTY 与 Agent SDK eval runner 以及 Claude benchmark 适配器:显式 model > `EVALS_MODEL` > `GSTACK_EVAL_MODEL_CAPTURE` > `GSTACK_EVAL_MODEL` > 默认。
- 共享裁判调用(包括 benchmark 质量评分):显式 model > `GSTACK_EVAL_MODEL_JUDGE` > `GSTACK_EVAL_MODEL` > 默认。`EVALS_MODEL` 适用于 capture runner,而不是裁判。

Warmup 保持在 `claude-haiku-4-5`;distill 保持在 `claude-haiku-4-5-20251001`。显式测试与历史 benchmark model 选择仍然优先。这些评估默认值与 [runtime model tiers](docs/model-policy.md) 独立钉住;tier 新鲜度工作流不会自动发现或更改评估 model。下面显示的付费运行成本是本次默认变更之前的历史估算,不是新默认的实测值。

### Which command do I run?

| I want to… | Command | Cost / time | Needs |
|---|---|---|---|
| 快速检查一次普通编辑 | `bun run test:quick` | 免费,约 1 分钟 | Bun 1.4.2 |
| 修复中跑单个免费测试文件 | `bun test <file>` | 免费,几秒 | 套件绝不要裸 `bun test` |
| 发布前跑完整免费验收 | `bun run test` | 免费,几分钟 | Bun 1.4.2 |
| 在小机器上跑完整免费套件 | `bun run test:ubicloud` | 16-vCPU 收费 VM 上的免费套件,约 5 分钟 | `UBICLOUD_API_KEY` |
| 跑策划的 Windows 安全子集 | `bun run test:windows` (CI: 6 个 `windows-latest` 任务) | 免费 | Windows,Git Bash |
| 刷新 Windows 时长种子 | `gh workflow run windows-free-tests.yml --ref <branch> -f record_durations=true`,然后提交 `free-test-durations-windows` 工件 | 免费 CI runner | 推送的分支,带 workflow 权限的 `gh` |
| 跑原生 Windows 或 Dia 资格活动 | `gh workflow run native-qualification.yml --ref <branch> -f mode=<mode>` ([modes](docs/TESTING_INTERNALS.md#windows-free-lane)) | 免费 CI runner | 推送的分支,带 workflow 权限的 `gh` |
| 预览我的 diff 选中的付费用例 | `bun run eval:select` (PR profile; `--profile full` 用于普通 touchfile 选择) | 免费 | — |
| 预览 CI 付费切片计划 | `bun run scripts/test-paid-shards.ts --tier periodic --list --slice-budget 420 --jobs 2` (diff-selected; `EVALS_ALL=1` 列出全部) | 免费 | — |
| 跑我变更的付费覆盖(代理:detached) | `bun run eval:bg:pr` (前台: `bun run test:pr`) | 仅所选用例的 API 花费;dispatch 到 CI 时约 10 分钟 | Dispatch: 干净、推送的 HEAD 与 `gh`。本地回退: `ANTHROPIC_API_KEY`、Claude Code CLI、纯终端 |
| 发布前跑完整 gate + periodic 普查 | `bun run eval:bg:release` (前台: `bun run test:release`) | 最大 API 花费;本地运行上限 4 小时 | 同上 |
| 跑单个付费 tier | `bun run test:gate:sharded` / `bun run test:periodic:sharded` (detached: `eval:bg:gate` / `eval:bg:periodic`) | 该 tier 的 API 花费 | 同上 |
| 跑单个付费用例(作其 CI 面板) | `bun run scripts/test-paid-shards.ts --tier <tier> --case <case-id> --trials 3` | 该用例的 API 花费 | 同上 |
| 在 CI 中验证分支上的 gate 用例 | `gh workflow run evals.yml --ref <branch> -f evals_all=true` | CI runner + API 花费 | 推送的分支,带 workflow 权限的 `gh` |
| 在 CI 中验证分支上的 periodic 用例 | `gh workflow run evals-periodic.yml --ref <branch>` (仅 periodic lane) | CI runner + API 花费 | 推送的分支,带 workflow 权限的 `gh` |
| 在分支上验证 periodic 用例加每周 gate 普查 | `gh workflow run evals-periodic.yml --ref <branch> -f include_gate_census=true` | CI runner + API 花费 | 推送的分支,带 workflow 权限的 `gh` |
| 跑 opt-in ML、gitleaks 与 Swift 检查 | `gh workflow run platform-qualification.yml --ref <branch>` | 免费 CI runner | 推送的分支,带 workflow 权限的 `gh` |
| 查看过去的本地 eval 运行 | `bun run eval:list` / `eval:compare` / `eval:summary` | 免费 | 本地 eval 历史 |
| 查看用例在最近每周运行中的通过率 | `bun run eval:pass-rates --case <case-id>` | 免费 | 带仓库读权限的 `gh` |
| 单个红用例对照 ship bar 测一次(MEETS 9/10) | `bun run scripts/ship-measure.ts measure --case <case-id> --round baseline` ([bar](docs/TESTING_INTERNALS.md#ship-measure)) | 10 次试验的 API 花费(behavior 12 次);每次试验高于 $2 时询问,在 $25/用例 停止 | 同付费覆盖 |
| 测量 main 上最 flaky 的 gate 用例(每周 sweep) | `gh workflow run eval-sweep.yml --ref main [-f k=5] [-f cap_usd=150] [-f dry_run=true]`,或本地 `bun run scripts/ship-measure.ts sweep --dry-run` ([sweep](docs/TESTING_INTERNALS.md#ship-measure-sweep)) | 每 7 天最多 $150 的 API 花费(`ship_measure_sweep_budget_usd`);`--dry-run` 免费 | 带仓库读权限的 `gh`;CI dispatch 需要 workflow 权限 |
| 查看审计成功度量与每周健康 | `bun run test:health [--since-days 7] [--json] [--enforce]` | 免费;5-20 分钟的 `gh` 读取 | 带仓库读权限的 `gh` |

旧命令名在 [Retired commands](#retired-commands) 下列出。

### Setup

开发和测试需要 Bun 1.4.2 或更新版本;CI 钉住并测试 1.4.2。较早的 Linux 版本在子进程垃圾回收期间会关闭不相关的 live 文件描述符,导致间歇性浏览器与 HTTP fixture 失败([upstream diagnosis](https://github.com/oven-sh/bun/issues/34785#issuecomment-5020318035))。

```bash
# 1. Copy .env.example and add your API key
cp .env.example .env
# Edit .env → set ANTHROPIC_API_KEY=sk-ant-...

# 2. Install deps (if you haven't already)
bun install
```

Bun 自动加载 `.env`——无需额外配置。Conductor workspace 自动从主 worktree 继承 `.env`(见下面的 "Conductor workspaces")。

### Test tiers

功能 QA 变更需要原生 fixture 证明以及提示检查。在本 checkout 之外、隔离的临时仓库中,添加已声明的 CLI 或 loopback API/worker 契约。练习成功与不利路径、耐久效果以及 setup 失败。仅报告的评估必须保留可变更工具的可用性,并独立检测禁止的写入,包括稍后恢复的编辑;干净的最终 diff 还不够。在付费运行之前,故意用坏的控制验证观察者。

对于探索性回归,保留实际的修复前失败、修复后通过、原始探针与邻近的快乐路径。自动调用者测试必须通过 review/ship 入口,而不是告诉代理运行被测试的组件。文档测试必须在发布前证明真实子进程已完成且父进程使用了它的结果;现有的 dispatch-only 测试是更窄的证据。在 touchfiles、tiers 与 PR profile 中注册新用例及所有消耗的 section/resolver 输入,让它们运行。报告问题时分享脱敏的复现命令与 fixture 证据,绝不分享凭据、私有 payload 或整个未审阅的代理转录。

| Tier | Runs through | What it tests |
|------|--------------|---------------|
| 1 — Static | `bun run test` (free) | 命令验证、snapshot 标志、Aside 契约钉子、render-wrapper 选项映射、SKILL.md 正确性、TODOS-format.md 引用、可观测性单元测试 |
| 2 — E2E | 分片付费 runner(`bun run eval:bg:pr`、`eval:bg:release`) | 通过 `claude -p`、PTY 与 Agent SDK 会话跑完整 skill |
| 3 — LLM eval | 同一 runner;`test/skill-llm-eval*.test.ts` | LLM-as-judge 对生成 SKILL.md 文档打分 |

从 [Which command do I run?](#which-command-do-i-run) 选命令。

PR 付费 gate 使用显式的短行为 profile。每个选定的质量裁判仍被包含;manifest 单独列出延后的行为与通过。未知的源依赖恢复完整 gate;job summary 命名每个导致它的文件及其修复([PR paid lane fallback](docs/TESTING_INTERNALS.md#pr-paid-lane-fallback))。没有注册覆盖的新提示会失败规划。已知的广义行为在它们的提示变更时仍可见地延后;它们不会静默获得 PR-pass 信用。完整 gate 与 periodic 普查在每周与手动 dispatch `evals-periodic.yml` 上跑新;`bun run eval:bg:release` 跑两者。因此一些广义行为失败会在 PR gate 之后被发现。

阻塞付费 lane(PR gate 与每周 periodic + gate 普查)目标中位约 10 分钟含 setup。规划者将记录的墙钟时间(`scripts/paid-test-durations.json`,按 tier)打包进它所需数量的 7 分钟(420 秒)runner,每个 runner 一个文件或紧密打包的一组;用例短但总时长长的文件每 runner 跑一个用例。矩阵大小与每个 runner 的 job 超时来自该计划,所以卡住的 runner 在其自身上限内失败。用 `bun run scripts/test-paid-shards.ts --tier periodic --list --slice-budget 420 --jobs 2` 免费预览它。完整的 start-to-finish 流程属于 `marathon` tier(`describeE2ETier('marathon')`),该 tier 仅在非阻塞的 `evals-marathon.yml` lane 中跑(每周与 dispatch 时),且从不 gate 一次合并。

裁决:付费 eval 从不重试。每个用例的 kind 在 `E2E_KINDS`(`test/helpers/touchfiles-data.ts`)中,从 `EVAL_POLICY`(`test/helpers/periodic-exclude-data.ts`)的常量中在运行前固定其试验次数:

- `rule`(默认):一次试验;任何失败的断言让用例失败。当没有随机性决定裁决时使用,或当裁决检查产品每次运行都必须满足的契约(plan 模式下无写入、决策前一个问题、skill 强制步骤、无泄漏的 secret)。
- `behavior`:一个 3 次独立试验的面板作为并行用例分片,无契约违规(`expectContract()`)时 ≥ 2 通过。仅当实时 model 决定裁决且偶尔偏差是可接受的产品行为时使用;one-line 理由放在 `BEHAVIOR_WHY` 中。
- `judge`:LLM 裁判对固定输入打分;同一提示的恰好 3 个样本,每个维度以其中位数为门槛(booleans 按多数;仅报告均值)。出错的样本让面板失败,从不重新采样。

超时、崩溃或基础设施失败的试验算作失败的试验,并连同其类别一起报告;缺失的试验使用例 INCOMPLETE,这会让 lane 失败。2/3 通过被报告为 `PASS 2/3` 并附失败试验的原因,而不是干净的通过。用例预算与门槛永不随此策略改变。隔离(`CASE_QUARANTINE`)与历史在 `docs/TESTING_INTERNALS.md` 中描述;`bun run eval:pass-rates --case <id>` 显示用例的每次试验通过率及其 Wilson 区间。

**普查变红了?** 按 [docs/evals/census-red.md](docs/evals/census-red.md) 走:读红行,用 `bun run eval:pass-rates --run <id>` 检查其证据,检查 `--reds` 与 `--headroom`,用免费回归测试修复,然后才用该行的 `after a repair:` 命令花一次付费运行。`main` 上的红普查在每周跟踪 issue 上报告;在分支上 dispatch 的普查将同一报告写入其 run summary 与 `census-report` 工件。

CI 对 16 个工作流质量裁判启用 24 小时内的已验证首次尝试复用。Cookie 工作流的自定义输入与其他 11 个质量用例保持新鲜。运行一次(无重试,因此通过可证明是首次尝试)的 PR-profile E2E 分片在每次消耗输入字节相同时复用同一 PR 上的通过:测试的 import closure、其注册用例的 touchfiles 引用的每个 tracked 文件、注册的全局 touchfiles、runner 与 workflow、子进程的 EVALS_/GSTACK_/CLAUDE_/ANTHROPIC_ 环境(仅 secret 存在性)、CI 镜像与 Claude CLI 版本(`scripts/e2e-shard-reuse.ts`)。计算的用例注册或匹配不到任何东西的 touchfile 模式让分片保持新鲜。每周普查、marathon 与发布 lane 从不复用。本地运行保持新鲜,除非提供完整的 scoped cache 与运行时配置。键包含完整的提示字节、生成的输入、fixtures、runner/rubric 代码、已安装依赖、模型设置与运行时。

当付费 eval 失败时,修复产品或 harness,并将捕获的用例作为一行添加到检测器的所有者测试中;`test/test-of-test-ratchet.test.ts` 在任何只导入 `test/` 代码并命名要扩展的所有者测试的新测试文件上失败。`CONTRIBUTING.md` "Test tiers" 给出一个例子。

### Retired commands

这些 package scripts 是一个版本的存根:每个打印其替代并以 exit 1 退出(`scripts/retired-command.ts`)。下个版本会删除它们。

| Retired | Use instead | Why it was retired |
|---|---|---|
| `test:evals` | `bun run eval:bg:pr` | 无 tier:跳过每个按 tier 门控的付费文件 |
| `test:evals:all` | `bun run eval:bg:release` | 无 tier:跳过每个按 tier 门控的付费文件 |
| `test:e2e` | `bun run eval:bg:pr` | 无 tier:跳过每个按 tier 门控的付费文件 |
| `test:e2e:all` | `bun run eval:bg:release` | 无 tier:跳过每个按 tier 门控的付费文件 |
| `test:gate` | `bun run test:gate:sharded` | 单进程扇出从未完成一次运行 |
| `test:periodic` | `bun run test:periodic:sharded` | 单进程扇出从未完成一次运行 |
| `test:codex` | `bun run test:periodic:sharded` | 未设 `EVALS_TIER`,所以两个 periodic-tier Codex 文件跑了零个用例 |
| `test:codex:all` | `bun run test:periodic:sharded` | 与 `test:codex` 相同 |
| `eval:bg` | `bun run eval:bg:pr` | detached 的是退役的 `test:evals` |
| `eval:bg:all` | `bun run eval:bg:release` | detached 的是退役的 `test:evals:all` |
| `eval:flake-rank` | `bun run eval:pass-rates` | 同一脚本的第二个名字 |
| `eval:watch` | tail `gstack-detach` 日志,或 `gh run watch <run-id>` | 读取只有未分片 runner 写的文件,所以对分片运行显示为空 |
| `test:audit` | `bun run test` | `test/audit-compliance.test.ts` 已经在免费套件中运行 |

### CI

一个 GitHub Action(`.github/workflows/skill-docs.yml`)在推送到 main 与 PR 上为所有 host 生成,然后拒绝 tracked 的差异与未忽略的未跟踪输出。生成错误也会让 job 失败。可选的被忽略的 host 缓存不与 Git 对比。

供应链 gates 与它一起跑:

- **Quality gate** (`.github/workflows/quality-gate.yml`,每个 PR 与 push) —— 用 gstack 自己的 redact 引擎(`.github/scripts/gate-secret-scan.mjs`)扫描 diff 的新增行。HIGH 发现让 job 失败;MEDIUM 发现作为建议计数浮出。如果扫描无法产生报告则 fail-closed。同时在 setup/build 边界上跑 ShellCheck。
- **Dependency review** (`.github/workflows/dependency-review.yml`) —— 在触及 `package.json` 或 `bun.lock` 文件的 PR 上审查依赖变更,并在 high 或 critical 公告上失败。它与每周 OSV 扫描是依赖 gate。
- **OSV scanner** (`.github/workflows/osv-scanner.yml`) —— 每周针对 OSV 数据库的漏洞扫描。配置位于 `.osv-scanner.toml` 并通过显式的 `--config` 标志加载(OSV 不会自动发现该文件名);每个 ignore 条目需要 reason 和 `ignoreUntil` 过期时间,由 `test/osv-config-wiring.test.ts` 强制执行。main 扫描失败时 upsert 跟踪 issue "OSV scanner: vulnerable dependency needs triage",干净扫描时关闭;`test/osv-ignore-expiry.test.ts` 在任何 `ignoreUntil` 之前 14 天失败,因此即将过期的抑制会在 PR 中浮出。
- **Weekly test health** (`.github/workflows/test-health.yml`,周一与 dispatch) —— 跑 `bun run test:health --since-days 7 --enforce`。当免费测试文件在至少 20 次 main 运行中超过 5% flake,或超过 5 个免费文件缺失时长种子时失败,upsert 一个跟踪 issue 在失败时,且仅当每个之前失败的检查都有证据并通过时才关闭。它无法读取的度量打印 "unavailable: <reason>; next: <step>" 且从不让运行失败。
- **Dependabot** (`.github/dependabot.yml`) —— 分组的依赖更新 PR。
- **OpenSSF Scorecard** (`.github/workflows/scorecard.yml`) —— 每周与 main push;结果在 Security 标签页与 api.scorecard.dev 中。

- **Platform qualification** (`.github/workflows/platform-qualification.yml`,dispatch 与季度) —— 跑没有其他 lane 有前置条件的 opt-in 测试:ML prompt-injection 分类器(`SECURITY_BENCH=1`)、针对真实 gitleaks 的内存摄入(`GSTACK_TEST_GITLEAKS`)以及 DebugBridge Swift 构建(`GSTACK_TEST_SWIFT=1`,macOS)。`test/platform-qualification-workflow.test.ts` 把每个 gate 钉到它的 job。

每个 workflow 把它的第三方 action 钉在 commit SHA(`test/workflow-action-pins.test.ts`)上。PR 模板(`.github/PULL_REQUEST_TEMPLATE.md`)要求证据——跑的测试、eval 输出——而不是承诺。

测试直接针对 browse 二进制运行——不需要 dev mode。需要 Aside 本身的任何东西(`test/skill-e2e-aside.test.ts`、Aside qa/design 用例、`test/aside-render.test.ts` 中的 live render)仅在 Mac 且 Aside app 打开时跑,在其它地方自跳;make-pdf 的 render gates 与 `/diagram` E2E 跑在解析出的任一引擎上,所以 CI 在 browse 二进制上跑它们(用 `bun run build:gates` 构建)。

## Editing SKILL.md files

SKILL.md 文件**由** `.tmpl` 模板**生成**。不要直接编辑 `.md`——你的变更会在下次构建时被覆盖。

```bash
# 1. Edit the template
vim SKILL.md.tmpl              # or browse/SKILL.md.tmpl

# 2. Regenerate for all hosts
bun run gen:skill-docs --host all

# 3. Check health (reports all hosts)
bun run skill:check

# Or use watch mode — auto-regenerates on save
bun run dev:skill
```

`skill:check` 渲染所有 host 到临时存储,使用规范内容路径与 host 默认,验证完整生成内容,并将预期的 tracked 工件与 checkout 对比。缺失、变更或未忽略的未跟踪输出会让检查失败。本地的被忽略 host 缓存(包括符号链接的缓存)不被触碰;checker 在没有它们时也能工作。生成失败无法产出部分输出的成功检查。

模板编写最佳实践(自然语言优于 bash-ism、动态分支检测、`{{BASE_BRANCH_DETECT}}` 使用)见 CLAUDE.md 的 "Writing SKILL templates" 一节。

技能中的浏览器步骤是遵循 `scripts/resolvers/aside.ts` 菜谱的 `aside repl` 脚本,每个都配一个回退引擎的 `$B` 等价物;在提交前让 Aside 形状对着 Aside CLI 跑过。要新增 browse 命令,加到 `browse/src/commands.ts`。要新增 snapshot 标志,加到 `browse/src/snapshot.ts` 的 `SNAPSHOT_FLAGS`。然后重新构建。

**通过 `lib/aside-render.ts` 渲染;不要在 skill 中捆绑 puppeteer/Chromium。** 需要把自己的 HTML/JSON 栅格化或打印的 skill(diagrams、cards、og-images、PDFs)从它的模板里调用 `bin/gstack-render.ts`(`--screenshot`、`--pdf`、`--eval JS --out FILE`),或从 TypeScript 中 import `render` from `lib/aside-render.ts`(`renderWithAside` / `renderWithBrowse` 是引擎特定的两半;`render` 在两者之间挑选并在 Aside 的 CLI 无法启动或中途丢失其 CDP 桥时,在 browse 引擎上重试一次)。包装器在它打开时通过 Aside 打印,未打开时通过 `browse` 守护进程打印(`newtab --json`、goto loopback URL、`js` readiness 轮询、`pdf --from-file`、`viewport` + `screenshot`、`js --out`、`closetab`)——每台 box 一个共享的 Chromium、同样的标志与 `OK <path>` 行,`ENGINE=aside|browse` 说出实际渲染的那个,`EVAL` / `PAGE_ERRORS` 行被围栏化为不受信任的 web 内容。loopback 服务器为每次渲染提供一个 secret URL,从不跟随跳出其目录的符号链接。尺寸化的截图在回退上为 1x(Aside 默认 2x);JPEG quality 与 `pageRanges`/`scale` 仅 Aside;`--landscape` 通过交换纸张尺寸模拟;`--wait-pagedjs` 映射到守护进程的 `toc` 等待。永远不要 `npm i puppeteer`,永远不要下载与版本不同步的第二个 Chromium,永远不要把渲染器指向一个网站。如果包装器缺一个你需要的选项,加到 `lib/aside-render.ts`(在 `test/aside-render.test.ts` 中钉住,如果是 CLI 标志也在 `test/gstack-render-cli.test.ts` 中钉住)以便每个调用者在两条路径上都获得它。导出的测试接缝:`pickEngine(fresh, deps)`(注入探查与二进制解析器)、`serveDir(root, nonce)`、`SAFE_TMP_DIR` 与 `PAGE_NUMBER_FOOTER`(make-pdf、`gstack-render` 与 browse `pdf` 命令共享的页面页脚)。

## Prompt audit at each frontier-model release

当发布新的 frontier Claude 模型时,审计 text 模型读取的内容中,新模型过度应用或不再需要的指令。审计是 Anthropic 的 `/claude-api prompt-audit`,一个你在自己的 Claude Code 会话中跑的 Claude Code skill;gstack 仅打印喂给它什么。

```bash
bun run audit:manifest   # slices of templates, resolvers, overlays, CLAUDE.md and wording-pinning tests (--json for a machine-readable list)
```

1. 在 Claude Code 中,把新模型作为 target,一次跑一个 slice 的 `/claude-api prompt-audit`,给它该 slice 的文件列表。`s01` 是每个 skill 都读取的共享文本(CLAUDE.md、model overlays、preamble resolvers);一个 skill 的模板与它的 sections 共享一个 slice。
2. 在模板与 resolvers 中修复发现,永远不要在生成的 SKILL.md 文件中,然后跑 `bun run gen:skill-docs --host all`。
3. 把安全规则视为已钉住的:仅在一个 eval 显示模型在变更前后都遵守的情况下才重写。
4. 最后处理「钉住 skill 措辞」的测试 slice。在散文变更的地方,用结构或意义级检查替换精确句子钉子(CLAUDE.md testing 一节的 prompt-bytes 规则)。
5. `test/archaeology-lint.test.ts` 让 issue 编号与事件故事不进生成的文本;它的失败会命名要修复的源文件。

## Jargon list (V1 writing style)

gstack 的 Writing Style 部分(注入每个 tier-≥2 的 skill 序章)按每个 skill 调用对首次出现的术语加注释。符合注释资格的术语列表位于 `scripts/jargon-list.json`——约 50 个精选的高频术语(idempotent、race condition、N+1、backpressure 等)。不在列表中的术语假定已经够浅显。

**添加或删除术语:** 开一个编辑 `scripts/jargon-list.json` 的 PR。在编辑后跑 `bun run gen:skill-docs`——术语在 gen 时被烘焙进每个生成的 SKILL.md,所以变更仅在重新生成后生效。无运行时加载;无用户端覆盖。仓库中的列表是真相之源。

适合添加的好候选:非技术用户在评审输出中无上下文遇到的高频术语(常见数据库/并发术语、安全 jargon、前端框架概念)。不要添加仅在一两个小众 skill 中出现的术语——成本-价值比不值得审阅开销。

## Multi-host development

gstack 从同一套 `.tmpl` 模板为 10 个 host 生成 SKILL.md 文件。每个 host 是 `hosts/*.ts` 中的一份类型化配置。生成器读取这些配置以产出 host 适配的输出(不同的 frontmatter、路径、工具名)。

**支持的 hosts:** Claude(主)、Codex、Factory、Kiro、OpenCode、Slate、Cursor、OpenClaw、Hermes、GBrain。

### Generating for all hosts

```bash
# Generate for a specific host
bun run gen:skill-docs                    # Claude (default)
bun run gen:skill-docs --host codex       # Codex
bun run gen:skill-docs --host opencode    # OpenCode
bun run gen:skill-docs --host all         # All 10 hosts

# Or use build, which does all hosts + compiles binaries
bun run build
```

### What changes between hosts

每个 host 配置(`hosts/*.ts`)控制:

| Aspect | Example (Claude vs Codex) |
|--------|---------------------------|
| Output directory | `{skill}/SKILL.md` vs `.agents/skills/gstack-{skill}/SKILL.md` |
| Frontmatter | 完整(name、description、hooks、version) vs 最小(name + description) |
| Paths | `~/.claude/skills/gstack` vs `$GSTACK_ROOT` |
| Tool names | "use the Bash tool" vs 相同(Factory 重写为 "run this command") |
| Hook skills | `hooks:` frontmatter vs 内联安全顾问散文 |
| Suppressed sections | GBrain blocks vs GBrain blocks 和 Review Army;Codex 保留 routed 到 Claude Code 的 outside-review sections |
| Model overlay | `claude` vs `gpt`(per-host `defaultModel`;`--model` 或在 setup 时,Codex `config.toml` model 覆盖) |

见 `scripts/host-config.ts` 中的完整 `HostConfig` 接口。

### Testing host output

```bash
# Run all static tests (includes parameterized smoke tests for all hosts)
bun run test

# Check freshness for all hosts
bun run gen-skill-docs --host all --dry-run

# Health dashboard covers all hosts
bun run skill:check
```

### Adding a new host

完整指南见 [docs/ADDING_A_HOST.md](docs/ADDING_A_HOST.md)。简短版:

1. 创建 `hosts/myhost.ts`(从 `hosts/opencode.ts` 复制),包括它的 `tier` 和 `capabilities`
2. 加到 `hosts/index.ts`
3. 把 `.myhost/` 加到 `.gitignore`
4. 跑 `bun run gen:skill-docs --host myhost`
5. 跑 `bun run test`(参数化测试自动覆盖它)

渲染不需要生成器代码。安装需要:一个可安装的 host 还需要一个 setup install arm、`gstack_host_tier`(`bin/gstack-install-registry.sh`)中的一行、README host-matrix 行,以及 conformance kit(`test/host-conformance.test.ts`)。在有日期的认证记录存在之前,它作为 `experimental` 发布;见 "Certify your host" 以及指南中的安装所有权规则。

### Adding a new skill

当你添加一个新 skill 模板时,所有 host 自动获得它:
1. 创建 `{skill}/SKILL.md.tmpl`
2. 跑 `bun run gen:skill-docs --host all`
3. 动态模板发现会拾取它,无需更新静态列表
4. 预算它:跑 `bun test/helpers/capture-context-budget.ts` 并提交刷新的 `test/fixtures/context-budget.json`——上下文预算 ratchet 在没有 ceiling 的 skill 上失败
5. 提交 `{skill}/SKILL.md`,外部 host 输出在 setup 时生成并被 gitignore

## Conductor workspaces

如果你使用 [Conductor](https://conductor.build) 并行运行多个 Claude Code 会话,`conductor.json` 自动接入 workspace 生命周期:

| Hook | Script | What it does |
|------|--------|-------------|
| `setup` | `bin/dev-setup` | 从主 worktree 复制 `.env`、安装依赖、符号链接 skills、非交互式跑 `./setup`,以及(如果 gbrain 已安装)把脑感知块渲染到 `.claude/gstack-rendered/` 而不弄脏 tracked 源 |
| `archive` | `bin/dev-teardown` | 删除 skill 符号链接、`.claude/gstack-rendered/` 渲染,并清理 `.claude/` 目录 |

当 Conductor 创建新 workspace 时,`bin/dev-setup` 自动运行。它检测主 worktree(通过 `git worktree list`),复制你的 `.env` 以让 API keys 跟随,并设置 dev mode——无需手动步骤。

`bin/dev-setup` 完全非交互式地跑 `./setup`(它传 `--plan-tune-hooks=prompt` 并关闭 stdin),所以转发的 Conductor TTY 永远不会被一个隐藏的 setup 提示挂住。它也永远不安装 plan-tune Claude Code hooks,意味着一个临时 workspace 不能把全局 `~/.claude/settings.json` 重写为指向一个临时 worktree 路径。要刻意安装 plan-tune hooks,跑 `./setup --plan-tune-hooks` 在 dev-setup 之外(或 `gstack-config set plan_tune_hooks yes`)。显式标志算作显式决定:setup 的 Conductor auto-opt-in for AskUserQuestion hooks 仅在真正的静默 fall-through(无标志、无 `GSTACK_PLAN_TUNE_HOOKS` env var、无 `plan_tune_hooks` key 字面上出现在 config 中,通过 `gstack-config has` 检查)时触发,所以它永远不能把 dev-setup override 成安装 hooks。一个说明的修复例外:setup 的 heal-first pass(`gstack-settings-hook prune-stale --repoint`)可以清理死的 gstack hook 条目并把现有的重新指向稳定的 `~/.claude/skills/gstack` 安装。这是严格收敛的修复,绝不是新注册,且注册本身仅是规范的,所以临时 tree 路径永远不会被烘焙进 settings.json。

**首次设置:** 把你的 `ANTHROPIC_API_KEY` 放在主仓库的 `.env` 中(见 `.env.example`)。每个 Conductor workspace 自动继承它。

**`GSTACK_*` env 前缀(Conductor 注入的 keys)。** Conductor 显式从每个 workspace 的进程 env 中剥离 `ANTHROPIC_API_KEY` 与 `OPENAI_API_KEY`。`.env` 复制路径也不能恢复它们——剥离发生在 env 继承之后。希望在 Conductor workspace 中让付费 eval、`/sync-gbrain` 嵌入或 `claude-agent-sdk` 调用工作的用户,必须在 Conductor 的 workspace env 配置中设置 `GSTACK_ANTHROPIC_API_KEY` 与 `GSTACK_OPENAI_API_KEY`;Conductor 原样传递这些。在 gstack 一侧,TS 入口作为副作用 import `lib/conductor-env-shim.ts`,它在规范名称为空时把 `GSTACK_FOO_API_KEY` 提升为 `FOO_API_KEY`,让任何看不到规范名称的子进程也能拿到它。如果添加新的 TS 入口,它们会用到付费 API,就在文件顶部加 `import "../lib/conductor-env-shim";`。今天该 shim 在 `bin/gstack-gbrain-sync.ts`、`bin/gstack-model-benchmark`、`scripts/preflight-agent-sdk.ts` 与 `test/helpers/e2e-helpers.ts` 中被 import。

## Things to know

- **SKILL.md files are generated.** 编辑 `.tmpl` 模板,而不是 `.md`。跑 `bun run gen:skill-docs` 重新生成。同一次运行也会从 `lib/design-catalog.ts` 与 `lib/dom-dump-script.ts` 生成 `review/design-checklist.md` 与 `lib/dom-dump.js`:编辑这些源,永不编辑生成文件(`test/design-checklist-sync.test.ts` 在漂移时失败)。
- **TODOS.md is the unified backlog.** 按 skill/组件组织,带 P0-P4 优先级。`/ship` 自动检测完成项。所有 planning/review/retro skill 会读它以获取上下文。
- **Browse、make-pdf、design 与 `lib/` 源码变更需要 rebuild。** 如果你触及 `browse/src/*.ts`、`make-pdf/src/*.ts`、`design/src/*.ts` 或 `lib/` 下的任何内容(`lib` 规范化的 `claude-bin.ts`、`error-handling.ts` 与 `aside-render.ts` 二进制会嵌入,加上 `design-catalog.ts`,其 `MOCKUP_NEVER_NAMES` 是设计二进制的 mockup prompt 嵌入的;`browse/src` 重新导出前三个),跑 `bun run build`。`./setup` 自己也会调用:当三个二进制任何一个缺失,或这些源、`package.json`、`bun.lock` 比 browse 二进制新时,会重建(`test/setup-needs-build.test.ts` 钉住该决定)。
- **Dev mode shadows your global install.** 项目本地 skills 优先于 `~/.claude/skills/gstack`。`bin/dev-teardown` 恢复全局的那一个。
- **Conductor workspaces are independent.** 每个 workspace 是它自己的 git worktree。`bin/dev-setup` 通过 `conductor.json` 自动运行。
- **`.env` propagates across worktrees.** 在主仓库中设置一次,所有 Conductor workspaces 都获得它。
- **`.claude/skills/` is gitignored.** 符号链接从不被提交。
- **Never write raw `ln -snf` in `setup`.** `setup` 中的每个链接点都必须经过 `IS_WINDOWS` 检测附近的 `_link_or_copy SRC DST` 辅助函数。该辅助函数在 Unix 上保留 `ln -snf`,在没有 Developer Mode 的 Windows 上切换为 `cp -R` / `cp -f`(那里裸 `ln -snf` 会产出在 `git pull` 时不刷新的冻结文件副本)。`test/setup-windows-fallback.test.ts` 用静态不变式强制——辅助函数体外的单个裸 `ln` 调用会让 CI 失败。
- **Synchronous subagent dispatches must state the flag.** Claude Code 默认在后台跑 Agent-tool 子代理(自 v2.1.198 起),所以任何派发子代理并消费其输出的模板步骤必须带 `run_in_background: false`。用 `{{FOREGROUND_DISPATCH_NOTE}}` 占位符(`scripts/resolvers/constants.ts`)代替手写该指引,并在同一 commit 中把生成的载体文件加到 `GENERATED_WITH_GUIDANCE` in `test/run-in-background-guidance.test.ts`——它的结构扫描器在缺失该 flag 的任何生成 dispatch imperative 上让 CI 失败。
- **Never delete or link over a skill entry `setup` cannot prove is gstack's.** `setup`(linker、alias installer、两个 prefix-flip cleanups)与 `bin/gstack-relink` 中的每个破坏性站点都经过所有权辅助函数(`setup` 中的 `_claude_entry_is_ours` / `_claude_entry_owned_strongly`,relink 中的 `_entry_is_ours` / `_entry_owned_strongly`)。退役-skill prune(`_prune_stale_generated`)通过其自己的门应用相同的强/弱拆分:当其 SKILL.md 携带生成 banner(`_owned_for_windows_refresh`)时,真实 host 目录才是候选;只有当它解析到 gstack 内部(`_gstack_target_is_ours`)时,host 符号链接才会被移除;带 banner 的真实目录通过 `_cleanup_weak_dir` 清理;render 树内部的符号链接永远不被跟随。指向 gstack 的符号链接或 `.gstack-owned` 标记证明整个目录;字节相同或带生成 banner 的 SKILL.md 仅证明那个文件,而一个不同的会先被移到 `~/.gstack/backups/skills/<ts>/` 下。`test/setup-link-ownership.test.ts`、`test/setup-cleanup-orphans.test.ts`、`test/setup-prune-stale-generated.test.ts` 与 `test/relink.test.ts` 钉住了它。该规则在两个脚本中重复,直到 TODOS.md 中 filed 的共享 helper 落地:同时改两个。host-render prune(`gstack_prune_host_renders` in `bin/gstack-host-renders.sh`,#1694)将相同的拆分应用到 checkout 不服务的 host 的 render 目录:指向 checkout 的链接走、被证明生成的移到 `~/.gstack/backups/host-renders/<ts>-<id>/` 下,其它保留,只有空目录被删除(`test/host-renders-prune.test.ts`)。

- **`./setup` never fails on Chromium.** Playwright bootstrap(setup 的 section `# 2`)是尽力而为且有界的:每次失败都会变成一个原因码(`skipped`、`chromium-install`、`chromium-install-timeout`、`chromium-install-locked`、`windows-no-node`、`windows-node-modules`、`post-install-launch`),在最终总结中与依赖浏览器的 skill 一起打印,且 skill 注册始终运行。`GSTACK_PLAYWRIGHT_INSTALL_TIMEOUT=<seconds>`(默认 600)限定下载;`GSTACK_SKIP_PLAYWRIGHT=1` 跳过它,适合无浏览器 box 或仅 setup 的测试循环。`GSTACK_SKIP_ASIDE=1` 让浏览器总结(像 skill 的探查和渲染器)把 Aside 当作缺席,这样总结从不承诺 bootstrap 没提供的回退(`test/setup-browser-hint.test.ts`)。你在 bootstrap 之后添加的任何东西必须保持独立于浏览器。`test/setup-playwright-best-effort.test.ts` 钉住该块。

## Testing your changes in a real project

**这是开发 gstack 的推荐方式。** 把你的 gstack checkout 符号链接进你实际使用它的项目,所以你的修改在做真实工作时立即生效。

### Step 1: Symlink your checkout

```bash
# In your core project (not the gstack repo)
ln -sfn /path/to/your/gstack-checkout .claude/skills/gstack
```

### Step 2: Run setup to create per-skill symlinks

单独的 `gstack` 符号链接不够。Claude Code 通过单独的顶层目录(`qa/SKILL.md`、`ship/SKILL.md` 等)发现 skills,而不是通过 `gstack/` 目录本身。跑 `./setup` 来创建它们:

```bash
cd .claude/skills/gstack && bun install && bun run build && ./setup
```

Setup 会问你想要短名(`/qa`)还是命名空间名(`/gstack-qa`)。你的选择被存到 `~/.gstack/config.yaml` 并在未来的运行中被记住。要跳过该提示,传 `--no-prefix`(短名)或 `--prefix`(命名空间)。

### Step 3: Develop

编辑一个模板,跑 `bun run gen:skill-docs`,然后下一次 `/review` 或 `/qa` 调用立即拾取它。无需重启。

### Going back to the stable global install

删除项目本地符号链接。Claude Code 回退到 `~/.claude/skills/gstack/`:

```bash
rm .claude/skills/gstack
```

每个 skill 目录(`qa/`、`ship/` 等)包含指向 `gstack/...` 的 SKILL.md 符号链接,所以它们会自动解析到全局安装。

### Switching prefix mode

如果你以一种前缀设置安装了 gstack 并想切换:

```bash
cd .claude/skills/gstack && ./setup --no-prefix   # switch to /qa, /ship
cd .claude/skills/gstack && ./setup --prefix       # switch to /gstack-qa, /gstack-ship
```

Setup 自动清理旧符号链接。无需手动清理。仅删除 gstack 创建的条目:你自己同名的 skill(比如手写的 `qa/`)被保留并在 setup 的最终总结中命名。

### Alternative: point your global install at a branch

如果你不想要每项目符号链接,你可以切换全局安装:

```bash
cd ~/.claude/skills/gstack
git fetch origin
git checkout origin/<branch>
bun install && bun run build && ./setup
```

这会影响所有项目。要恢复:`git checkout main && git pull && bun run build && ./setup`。

## Community PR triage (wave process)

当社区 PR 累积时,将它们分批为按主题的波:

1. **分类** —— 按主题分组(安全、功能、基础设施、文档)
2. **去重** —— 如果两个 PR 修了同一件事,选那个变更更少行的。用一条指向胜者的备注关闭另一个。
3. **Collector branch** —— 创建 `pr-wave-N`,合并干净的 PR,为脏的解决冲突,用 `bun run test && bun run build` 验证
4. **Close with context** —— 每个关闭的 PR 都得到一条评论解释为什么以及什么(如果有)取代了它。贡献者做了真实的工作;用清晰的沟通尊重它。
5. **Ship as one PR** —— 一个 PR 到 main,所有归属在 merge commit 中保留。包括一个已合并与已关闭内容的总结表。

参见 [PR #205](../../pull/205)(v0.8.3)作为第一次 wave 的例子。

## Upgrade migrations

当一个 release 改变了 `./setup` 自身无法修复的磁盘状态(目录结构、配置格式、过期文件),添加一个迁移脚本,以便现有用户得到干净的升级。

### When to add a migration

- 改变了如何创建 skill 目录(符号链接 vs 真实目录)
- 在 `~/.gstack/config.yaml` 中重命名或移动配置键
- 需要删除之前版本留下的孤儿文件
- 改变了 `~/.gstack/` 状态文件的格式

不要为以下情况添加迁移:新功能(用户自动获得)、新 skill(setup 发现它们)、或仅代码变更(无磁盘状态)。

### How to add one

1. 创建 `gstack-upgrade/migrations/v{VERSION}.sh`,其中 `{VERSION}` 与需要修复的 release 的 VERSION 文件匹配
2. 让它可执行:`chmod +x gstack-upgrade/migrations/v{VERSION}.sh`
3. 脚本必须是**幂等的**(安全地多次运行)与**非致命的**(失败被记录但不会阻塞升级)
4. 在顶部包含一个注释块,解释改了什么、为什么需要迁移,以及哪些用户受影响

例子:

```bash
#!/usr/bin/env bash
# Migration: v0.15.2.0 — Fix skill directory structure
# Affected: users who installed with --no-prefix before v0.15.2.0
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")/../.." && pwd)"
"$SCRIPT_DIR/bin/gstack-relink" 2>/dev/null || true
```

### How it runs

在 `/gstack-upgrade` 期间,`./setup` 完成后(Step 4.75),upgrade skill 扫描 `gstack-upgrade/migrations/` 并运行每个版本比用户旧版本新的 `v*.sh` 脚本。脚本按版本顺序运行。失败被记录但从不会阻塞升级。

### Testing migrations

迁移作为 `bun run test`(tier 1,免费)的一部分被测试。测试套件验证 `gstack-upgrade/migrations/` 中的所有迁移脚本都是可执行的,并且解析无语法错误。

## Shipping your changes

当你对你的 skill 编辑满意时:

```bash
/ship
```

它会跑测试、评审 diff、对 Greptile 评论分类(带 2 级升级)、管理 TODOS.md、bump 版本、打开 PR。完整工作流见 `ship/SKILL.md`。
