# gstack

> "我想我自从十二月以来基本就没自己敲过一行代码了,这是一个巨大的变化。" — [Andrej Karpathy](https://fortune.com/2026/03/21/andrej-karpathy-openai-cofounder-ai-agents-coding-state-of-psychosis-openclaw/),No Priors 播客,2026 年 3 月

当我听到 Karpathy 这么说,我开始想弄明白:一个人怎么像一支 20 人团队那样出货?Peter Steinberger 做了 [OpenClaw](https://github.com/openclaw/openclaw) — 247K GitHub stars — 几乎靠 AI agent 单干。革命已经来了。一个带着合适工具的单人 builder,能比传统团队还快地推进。

我是 [Garry Tan](https://x.com/garrytan),[Y Combinator](https://www.ycombinator.com/) 的总裁兼 CEO。我和几千家创业公司共事过 — Coinbase、Instacart、Rippling — 在它们只有一两个人在车库里的阶段。在 YC 之前,我是 Palantir 最早的工程/产品/设计之一,联合创办了 Posterous(后被 Twitter 收购),还搭建了 YC 内部的社交网络 Bookface。

**gstack 就是我的答案。** 我做产品已经二十年,而现在我出货的产品比我任何时候都多。过去 60 天:3 个生产服务、40+ 上线的特性,兼职,YC 全职的同时。年化逻辑代码变更 — 不是原始 LOC,AI 会把那个吹大 — 我的 2026 节奏 **~810× 我 2013 的速度**(11,417 vs 14 逻辑行/天)。截至 4 月 18 日,2026 年已经产出 **240× 整个 2013 全年**。跨 40 个公开 + 私有的 `garrytan/*` 仓库(含 Bookface),排除一个 demo 仓库后统计。AI 写了大部分。重点不是谁敲的,是 ship 了什么。

> 那些质疑 LOC 的人没说错原始行数会被 AI 吹大。他们错的是,归一化通胀之后,我没那么高产。我高产,而且高得多。完整方法、注意事项和复现脚本: **[On the LOC Controversy](docs/ON_THE_LOC_CONTROVERSY.md)**。

**2026 — 1,237 次贡献,还在增长:**

![GitHub contributions 2026 — 1,237 次贡献,1-3 月大幅加速](docs/images/github-2026.png)

**2013 — 那时我在 YC 搭 Bookface(772 次贡献):**

![GitHub contributions 2013 — 在 YC 搭 Bookface 的 772 次贡献](docs/images/github-2013.png)

同一个人,不同时代。差别在工具。

**gstack 就是我怎么做出来的。** 它把 Claude Code 变成一支虚拟工程团队 — 一个会重新思考产品的 CEO、一个锁定架构的工程经理、一个会抓 AI slop 的设计师、一个找到生产 bug 的审查者、一个打开真实浏览器的 QA 负责人、一个跑 OWASP + STRIDE 审计的安全官、一个出 PR 的发布工程师。23 个专家 + 8 个强力工具,全是斜杠命令,全 Markdown,全免费,MIT 协议。

这就是我的开源软件工厂。我每天都在用。我分享出来,因为这些工具该被所有人用上。

拿走它,改它,变成你的。如果你想喷免费开源软件 — 欢迎,但我更希望你先试一下。

**谁适合用:**
- **创始人和 CEO** — 尤其是还想继续自己 ship 的技术型
- **Claude Code 新手** — 用结构化的角色,而不是空白的 prompt
- **Tech lead 与 staff 工程师** — 在每个 PR 上做严格审查、QA 和发布自动化

## Quick start

1. 装上 gstack(约 30 秒 — 见下)
2. 跑 `/office-hours` — 讲清楚你在做什么
3. 在任何特性想法上跑 `/plan-ceo-review`
4. 在任何带改动的 branch 上跑 `/review`
5. 在 staging URL 或本地隔离的 API、CLI、job 或 webhook 上跑 `/qa`
6. 到此为止。你就知道它适不适合你了。

## Install — 大约 30 秒

`./setup` 在一台 4-vCPU Linux 云机上从 clone 到完成耗时 31 秒(v1.91.13.0,干净 HOME,含二进制构建和 Chromium 下载)。慢速网络和笔记本会更慢。

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[Git](https://git-scm.com/)、[Bun](https://bun.sh/) v1.4.2+(测试版本;setup 拒绝低于 1.3.3 的 Bun,那样编译出来的工具能读项目的 `.env`)、[Node.js](https://nodejs.org/)(仅 Windows)。**macOS 推荐:** [Aside](https://aside.com) 浏览器(macOS 15+)— 浏览器技能、`/make-pdf` 和 `/diagram` 优先驱动它,使用你的真实登录会话。如果没有,`./setup` 会构建 gstack 自带的浏览器,同样的技能就用它。`/cso` 额外需要带全部四个 `--no-compile-autoload-*` 构建标志的 Bun release 加原生工具链:Linux 上要支持静态链接的 C 编译器,macOS 上要 Xcode 命令行工具,Windows 上要 Visual Studio 2022 Build Tools 带 C++ 桌面开发负载。如果缺失,setup 会装好其它东西、清理过期 CSO 助手,`/cso` 报告 `not assessed` 并指出前置条件。

当合格的 CSO 运行时镜像发布后,setup 给每个自动预加载 30 秒窗口,加上声明 catalog 的有界 setup 预算。慢镜像可以设 `GSTACK_CSO_IMAGE_PULL_TIMEOUT_SECONDS=120`(允许范围 5–300 秒)。一张图超时不会消耗剩余图的窗口;setup 报告部分进度,之后再跑会从本地 Docker 已存在的精确摘要续上。整个预加载上限 1 小时。

### Step 1: 装到本机

打开 Claude Code,粘贴下面这段。Claude 完成其余步骤。

> Install gstack: run **`git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`** then add a "gstack" section to CLAUDE.md that says to use the /browse skill from gstack for all web browsing, never use mcp\_\_claude-in-chrome\_\_\* tools, and lists the available skills: /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review, /design-consultation, /design-shotgun, /design-html, /review, /deslop-shared-libs, /test-audit, /ship, /eval-plan, /land-and-deploy, /canary, /benchmark, /browse, /connect-chrome, /qa, /qa-only, /design-review, /scrape, /setup-browser-cookies, /setup-deploy, /setup-gbrain, /retro, /investigate, /document-release, /document-generate, /codex, /cso, /autoplan, /plan-devex-review, /devex-review, /careful, /freeze, /guard, /unfreeze, /gstack-upgrade, /learn. Then ask the user if they also want to add gstack to the current project so teammates get it.

### Step 2: Team mode — 共享仓库自动更新(推荐)

在 repo 内粘贴下面这段。切到 team mode,引导仓库让队友自动获取 gstack,然后提交:

```bash
(cd ~/.claude/skills/gstack && ./setup --team) && ~/.claude/skills/gstack/bin/gstack-team-init required && git add .claude/ CLAUDE.md && git commit -m "require gstack for AI-assisted work"
```

你仓库里不引入 vendored 文件,没有版本漂移,不用手动升级。每个 Claude Code 会话开始时做一次快速的自动更新检查(节流到一小时一次,网络失败安全,完全静默)。

把 `required` 换成 `optional`,如果你更想 nudge 队友而不是阻塞他们。

### OpenClaw

OpenClaw 通过 ACP 生成 Claude Code 会话,因此只要 Claude Code 装了 gstack,每个 gstack 技能就直接可用。粘贴到你的 OpenClaw agent:

> Install gstack: run `git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup` to install gstack for Claude Code. Then add a "Coding Tasks" section to AGENTS.md that says: when spawning Claude Code sessions for coding work, tell the session to use gstack skills. Include these examples — security audit: "Load gstack. Run /cso", code review: "Load gstack. Run /review", QA test a URL: "Load gstack. Run /qa https://...", build a feature end-to-end: "Load gstack. Run /autoplan, implement the plan, then run /ship", plan before building: "Load gstack. Run /office-hours then /autoplan. Save the plan, don't implement."

**装好后,直接和你的 OpenClaw agent 自然对话:**

| 你说 | 发生什么 |
|---------|-------------|
| "Fix the typo in README" | 简单 — Claude Code 会话,不需要 gstack |
| "Run a security audit on this repo" | 用 `Run /cso` 派生 Claude Code |
| "Build me a notifications feature" | 派生 Claude Code 跑 /autoplan → implement → /ship |
| "Help me plan the v2 API redesign" | 派生 Claude Code 跑 /office-hours → /autoplan,保存方案 |

高级分发路由以及 gstack-lite/gstack-full prompt 模板见 [docs/OPENCLAW.md](docs/OPENCLAW.md)。

### Native OpenClaw Skills(通过 ClawHub)

四个方法学技能,直接在 OpenClaw agent 里跑,不需要 Claude Code 会话。从 ClawHub 安装:

```
clawhub install gstack-openclaw-office-hours gstack-openclaw-ceo-review gstack-openclaw-investigate gstack-openclaw-retro
```

| Skill | What it does |
|-------|-------------|
| `gstack-openclaw-office-hours` | 用 6 个强制问题做产品拷问 |
| `gstack-openclaw-ceo-review` | 4 种 scope 模式下的战略挑战 |
| `gstack-openclaw-investigate` | 根因调试方法论 |
| `gstack-openclaw-retro` | 每周工程复盘 |

这些是对话型技能。你的 OpenClaw agent 直接在聊天里跑它们。

### Other AI Agents

gstack 不止跑在 Claude 上。裸的 `./setup` 只装 Claude Code。用 `--host <name>` 另选 agent,或用 `--host auto` 让 setup 检测本机所有 agent 一次性安装:

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/gstack
cd ~/gstack && ./setup --host auto      # 或: ./setup --host codex
./setup --status                         # 每个安装一行: host、tier、scope、version、path
```

显式 `--host X` 只装 X,不会改其它 agent 的安装。`/gstack-upgrade` 一次性刷新它登记的每个安装,每行一个 host,并说明哪些失败。

一个 checkout 只携带它安装出去的 agent 的技能。setup 把它们记到 `.gstack-installed-hosts` 且永不丢弃一个,所以之后再加 agent 就是 `./setup --host <name>`。为其它 agent 渲染出的副本会被剪枝:生成文件移到 `~/.gstack/backups/host-renders/`,gstack 没法证明是自己生成的文件保持原位。一个全局 Claude 安装从 632 个 `SKILL.md`(34.7 MB)减到自己的 63 个(2.5 MB),避免 Cursor-agent 在技能树上卡死(#1694)。在开发 checkout 里,`bun run build` 仍然渲染所有 agent;在安装里只渲染已登记的,`GSTACK_RENDER_HOSTS=all bun run build` 渲染全部。

层级:**full** 经过真实工作流运行认证(见 [Certify your host](docs/ADDING_A_HOST.md#certify-your-host));**experimental** 安装并通过 conformance 测试但尚未有认证运行;**instruction-only** 不安装任何东西,只打印要复制的内容。

| Agent | Tier | Install | First invocation | Safety skills | Outside review needs | Repair |
|-------|------|---------|------------------|---------------|----------------------|--------|
| Claude Code | full | `--host claude`(默认) | `/office-hours` | enforced(hook 阻断) | Codex CLI,已登录 | `./setup --host claude` |
| OpenAI Codex CLI | experimental | `--host codex` → `${CODEX_HOME:-~/.codex}/skills/gstack-*/` | ask for `gstack-office-hours` | advisory, not blocked | Claude Code CLI,已登录 | `./setup --host codex` |
| OpenCode | experimental | `--host opencode` → `~/.config/opencode/skills/gstack-*/` + `/gstack-*` commands | `/gstack-office-hours` | advisory, not blocked | Codex CLI,已登录 | `./setup --host opencode` |
| Cursor | experimental | `--host cursor` → `~/.cursor/skills/gstack-*/` | ask for `gstack-office-hours` | advisory, not blocked | Codex CLI,已登录 | `./setup --host cursor` |
| Factory Droid | experimental | `--host factory` → `~/.factory/skills/gstack-*/` | ask for `gstack-office-hours` | advisory, not blocked | Codex CLI,已登录 | `./setup --host factory` |
| Kiro | experimental | `--host kiro` → `~/.kiro/skills/gstack-*/` | ask for `gstack-office-hours` | advisory, not blocked | Codex CLI,已登录 | `./setup --host kiro` |
| GitHub Copilot CLI | experimental | `--host copilot` → `~/.copilot/skills/gstack-*/` | `/gstack-office-hours` | advisory, not blocked | Codex CLI,已登录 | `./setup --host copilot` |
| Antigravity CLI | experimental | `--host agy`(或 `--host antigravity`)→ `~/.gemini/antigravity-cli/skills/gstack-*/` | `/gstack-office-hours` | advisory, not blocked | Codex CLI,已登录 | `./setup --host agy` |
| Slate | instruction-only | `--host slate`(指向 Claude 安装;Slate 读 `.claude/skills`) | `/office-hours` via the Claude install | advisory, not blocked | — | `./setup --host claude` |
| OpenClaw | instruction-only | `--host openclaw`(打印摘要路径;ACP 派生 Claude Code — [docs/OPENCLAW.md](docs/OPENCLAW.md)) | "Load gstack. Run /review" | advisory, not blocked | — | 升级后重新复制摘要 |
| Hermes | instruction-only | `--host hermes`(打印摘要路径和 `gen:skill-docs --host hermes`) | 复制摘要,或自己渲染技能 | advisory, not blocked | — | 升级后重新复制摘要 |
| GBrain (mod) | instruction-only | `--host gbrain`(brain-aware 变体从 GBrain 仓库发布) | — | advisory, not blocked | — | — |

Copilot 用前缀名(`/gstack-review`)调用 gstack 技能,因为 `/review` 是 Copilot 内建。Copilot 忽略 skill hook,所以 `/careful` 和 `/freeze` 在那里只是 advisory,且 `COPILOT_HOME` 还不支持非 `~/.copilot`(setup 拒绝并保持不变)。

Antigravity CLI 也用前缀名(`/gstack-review`)调用技能。setup 只写 `~/.gemini/antigravity-cli/skills/gstack*`:Gemini CLI 的 `~/.gemini/skills` 和配置从不被改动,`--host auto` 只在 `PATH` 上有 `agy` 或存在 `~/.gemini/antigravity-cli` 时选 Antigravity,绝不因仅 `~/.gemini` Gemini-CLI-only 而选它。Antigravity 2.0 app 和 IDE 读 `~/.gemini/config/skills`,本 host 不安装。

外部审查需要所选 CLI 已安装并登录:Claude Code 在 Codex 上用 gstack,或 Codex 在其他 harness 上用。外部 harness 把这些命令暴露为 `/gstack-claude-code` 和 `/gstack-codex`;每个 harness 省略自身的 wrapper。显式 provider 请求保持该 provider。现有 `codex_reviews` 设置控制自动外部审查(在支持处),与所选 provider 无关。`/codex` 和 `/claude-code` 在你点名要它们时仍然会跑。

`/claude` 已重命名为 `/claude-code`。重跑 `./setup --host <name>` 来迁移受管的安装,包括共享 checkout 的其他 harness。Setup 在替换生成或安装失败时保留之前安装并打印修复说明。

**Instruction-only 层级(任何读规则的 agent — Zed、Amp、Jules、side project):**
把 [`agents-digest/gstack-AGENTS.md`](agents-digest/gstack-AGENTS.md) 里 2KB 的摘要复制到你的 agent 读得到的位置(例如追加到项目 `AGENTS.md`)。它带 gstack 的 ethos、reuse ladder 与 voice 规则 — 不需要安装。摘要首行显示其 gstack 版本;升级后重新复制。

对 Codex,setup 从 `${CODEX_HOME:-~/.codex}/config.toml` 读顶层 `model`,生成匹配的行为画像;若没可用模型配置则回退 `gpt-6-astra`。`gpt-5.6-sol` 自动获得有界范围指令,完成请求的 lake 而不扩散到周边清理或推测性硬化。Sol 画像是精确匹配:日期快照和其它 5.6 变体走通用 GPT 画像,setup 对近失配如 `gpt-5.6-sol-2026-08-01` 发出警告。用 `./setup --host codex --model <id>` 覆盖检测(本次生效);在 Codex `config.toml` 设 `model` 使其跨升级保持。改完 Codex 模型后重跑 `./setup --host codex` 重新生成技能。

**Plan-review 与实现层级。** Plan 工作流使用独立 `frontier` 审查者:Fable 5.1(来自 Codex)或 GPT-6 Astra(来自 Claude Code)。实现交接推荐 `smart` 层级:Opus 5.5 或 GPT-6.1 Sol。可选别的层级,固定任一 provider 的模型,或用 `host` 模式保留原生模型设置。只读的 `gstack-models` 命令解释有效选择;改这些设置不需要重新生成技能也不切换当前会话的运行模型。每周维护检查标记变更的官方建议和退役通知,不自动改模型。详见 [model policy: setup, overrides and freshness](docs/model-policy.md)。

**其它 Codex 调用保持原选。** 没有显式 `plan-review` 角色(含普通 review 与 ship 对抗 pass)时,gstack 按这个顺序挑模型:你为该请求指定的、`GSTACK_CODEX_MODEL`、Codex `config.toml` 里的 `model`(原生 `codex review` 时优先 `review_model`;自定义 `CODEX_HOME` 也尊重),最后才是 gstack 默认 `gpt-6-astra`。在花任何钱之前它会打印所选和来源,例如 `CODEX_MODEL: gpt-5.6-terra (exec; source: ~/.codex/config.toml model)`。若选择无效或账号不能用,gstack 停下给出修复消息并报告外部审查 unavailable。它从不悄悄切回默认。嵌套 Codex 审查也运行隐藏已装技能(`-c skills.include_instructions=false`),这样 review 不会变成一整个嵌套技能运行。运行时模型选择与上面的 setup-time 行为画像分开。没有 plan-review 角色时,/claude-code(`gstack-claude-code` 在 Codex 上)保留 Claude 配置的模型。设 `GSTACK_CLAUDE_MODEL=<model>` 或在请求中指定模型,以覆盖该次调用的模型,包括恢复的会话。捕获、judge 和基准模型选择见 [eval defaults and overrides](CONTRIBUTING.md#testing--evals)。

设计二进制(`$D`)默认用 OpenAI 的 `gpt-5.5`,图片生成通过 `gpt-image-2`。设 `GSTACK_DESIGN_MODEL=<model>` 用另一个 gpt-5-class 模型做图片生成和截图分析;若 OpenAI 拒绝,错误会指明 `GSTACK_DESIGN_MODEL`。设 `GSTACK_DESIGN_IMAGE_MODEL=<gpt-image model>` 改图片工具模型;非 gpt-image 模型名称的值在发请求前就拒绝。用 `bun run design/scripts/live-model-check.ts` 对照默认检查一个 key,总是测默认模型并忽略两个覆盖;每周定期普查跑同样检查。设 `OPENAI_BASE_URL` 把所有 `$D` 调用送到 OpenAI 兼容网关而不是 `api.openai.com`;egress 收据记录网关主机。在 Codex 上,/design-shotgun 改用 Codex 自带的 `$imagegen` 技能生成 mockup,因此不需要 `OPENAI_API_KEY`;`$D` 仍然构建并服务对比板。

**要加对新 agent 的支持?** 见 [docs/ADDING_A_HOST.md](docs/ADDING_A_HOST.md)。渲染一个新 agent 是一个 TypeScript 配置文件;装它还需要一个 setup arm 和 conformance 工具包,起步就是 experimental。

## See it work

```
You:    I want to build a daily briefing app for my calendar.
You:    /office-hours
Claude: [asks about the pain — specific examples, not hypotheticals]

You:    Multiple Google calendars, events with stale info, wrong locations.
        Prep takes forever and the results aren't good enough...

Claude: I'm going to push back on the framing. You said "daily briefing
        app." But what you actually described is a personal chief of
        staff AI.
        [extracts 5 capabilities you didn't realize you were describing]
        [challenges 4 premises — you agree, disagree, or adjust]
        [generates 3 implementation approaches with effort estimates]
        RECOMMENDATION: Ship the narrowest wedge tomorrow, learn from
        real usage. The full vision is a 3-month project — start with
        the daily briefing that actually works.
        [writes design doc → feeds into downstream skills automatically]

You:    /plan-ceo-review
        [reads the design doc, challenges scope, runs 10-section review]

You:    /plan-eng-review
        [ASCII diagrams for data flow, state machines, error paths]
        [test matrix, failure modes, security concerns]

You:    Approve plan. Exit plan mode.
        [writes 2,400 lines across 11 files. ~8 minutes.]

You:    /review
        [AUTO-FIXED] 2 issues. [ASK] Race condition → you approve fix.

You:    /qa https://staging.myapp.com
        [opens real browser, clicks through flows, finds and fixes a bug]

You:    /ship
        Tests: 42 → 51 (+9 new). PR: github.com/you/app/pull/42
```

你说 "daily briefing app"。agent 说 "你在搭 chief of staff AI" — 因为它听的是你的痛,不是你的功能请求。8 条命令,从想法到上线。那不是 copilot,那是一支团队。

## The sprint

gstack 是流程,不是工具集合。技能按 sprint 的顺序跑:

**Think → Plan → Build → Review → Test → Ship → Reflect**

每个技能喂给下一个。`/office-hours` 写的设计文档 `/plan-ceo-review` 读。`/plan-eng-review` 写的测试计划 `/qa` 接。`/review` 抓的 bug `/ship` 验证修。一个不掉链,因为每一步都知道前一步做了什么。

| Skill | Your specialist | What they do |
|-------|----------------|--------------|
| `/office-hours` | **YC Office Hours** | 从这开始。6 个强制问题,在你写代码前重新框定你的产品。推翻你的框架,挑战前提,生成多个实现方案。设计文档喂给所有下游技能。 |
| `/plan-ceo-review` | **CEO / Founder** | 重新思考问题。在请求里找出 10 星产品。4 种模式:Expansion、Selective Expansion、Hold Scope、Reduction。 |
| `/plan-eng-review` | **Eng Manager** | 锁定架构、数据流、图、边界情况和测试。把隐藏假设暴露出来。 |
| `/plan-design-review` | **Senior Designer** | 每个设计维度打 0-10,说明 10 分长什么样,然后改 plan 到那里。AI Slop 检测。交互式 — 每个设计选择一次 AskUserQuestion。 |
| `/plan-devex-review` | **Developer Experience Lead** | 交互式 DX review:探索开发者画像,与竞品 TTHW(time-to-hello-world)对标,设计你的 magical moment,逐步追踪摩擦点。3 种模式:DX EXPANSION、DX POLISH、DX TRIAGE。20-45 个强制问题。 |
| `/design-consultation` | **Design Partner** | 从零搭建完整设计系统。调研格局,提出有创造性的风险,生成现实的产品 mockup。以开放 DESIGN.md 格式写 `DESIGN.md`,让 impeccable、Google Stitch 和任何读它的工具共享同一文件。 |
| `/review` | **Staff Engineer** | 找出过 CI 但在生产爆掉的 bug。自动修复明显的。标记完整性缺口与 gate integrity:每个被削弱的测试、timeout、snapshot、suppression 都被点名叫到人。咨询性 simplification lens 标记过度构建的代码 — 从不阻断,从不自动应用。 |
| `/deslop-shared-libs` | **Shared Code Reviewer** | 在近期工作里找值得抽取的共享代码。比较多达 5 个机会并推荐最好的 3 个,带源证据、可靠性收益和总代码节省。只给建议。 |
| `/test-audit` | **Test Auditor** | 扫现有测试,找出成本大于保护价值的:源码 grep、重复、无断言探测、仅测试导出。每个候选带证据卡;只报告,除非你批准一批。 |
| `/investigate` | **Debugger** | 系统化根因调试。铁律:未调查不修。追踪数据流,测试假设,3 次修复失败后停。 |
| `/design-review` | **Designer Who Codes** | 与 /plan-design-review 同审,然后修它找到的。原子提交,前后对比截图。如果你装了 impeccable,它的引擎先跑,每个机械发现都带规则 id 标签。 |
| `/devex-review` | **DX Tester** | 现场开发者体验审计。真的测你的 onboarding:导航文档,试 getting started,计时 TTHW,给错误截图。与 /plan-devex-review 分数对比 — 回旋镖,看你的 plan 是否对得上现实。 |
| `/design-shotgun` | **Design Explorer** | "给我看选项。" 生成 4-6 个 AI mockup 变体,在浏览器开对比板,收集反馈,迭代。品味记忆学到你喜欢什么。重复到爱上,然后交给 /design-html。 |
| `/design-html` | **Design Engineer** | 把 mockup 变成真的能用的生产 HTML。Pretext 算出的布局:文本回流,高度自适应,布局是动态的。30KB,零依赖。检测 React/Svelte/Vue。按设计类型智能路由 API(landing page vs dashboard vs form)。过一次 impeccable 引擎 slop-gate,如果你装了它。输出可上,不是 demo。 |
| `/qa` | **QA Lead** | 探索 browser、API、CLI、job 与 webhook 行为。复现 bug,写失败的回归,修根因并重新验证再提交。 |
| `/qa-only` | **QA Reporter** | 探索并报告,带可重放证据。提出回归用例,不修改产品代码或测试。 |
| `/pair-agent` | **Multi-Agent Coordinator** | 把 gstack 自己的浏览器共享给任意 AI agent。一条命令,一段粘贴,连上。可与 OpenClaw、Hermes、Codex、Cursor 或任何能 curl 的 agent 配合。每个 agent 拿自己的 tab。自动起 headed 模式让你看到一切。自动起 ngrok 隧道给远程 agent。scoped token、tab 隔离、限速、活动归属。(跑在自带浏览器上 — fallback 引擎;用 Aside 的 agent 自己开 tab。) |
| `/cso` | **Chief Security Officer** | 带应用模型的安全审计,带支持的发现、独立的挑战、显式的 coverage。无 catalog profile 也可用静态评估。匹配合格 profile 时,comprehensive 模式对 Node/Bun、Python、Rails 增加受限运行时/scanner 执行和可审查的修复候选;目前还没有合格的运行时或 scanner profile,所以审计只跑静态评估。运行时测试的 bundle 单独鉴权外部断言。Project-test 完成仍然是 `self_reported`,因为目标代码控制测试过程;`tested` 留给未来目标无关的完成见证。 |
| `/ship` | **Release Engineer** | 同步 main,跑测试,探索改动行为,审计 coverage、gate integrity 和文档,然后验证、推送、开 PR。 |
| `/land-and-deploy` | **Release Engineer** | 合 PR,等 CI 与部署,验证生产健康。从 "approved" 一条命令走到 "verified in production"。 |
| `/canary` | **SRE** | 部署后监控循环。观察 console 错误、性能回退和页面失败。 |
| `/benchmark` | **Performance Engineer** | 页面加载时间、Core Web Vitals、资源大小的基线。每个 PR 上前后对比。 |
| `/eval-plan` | **Eval Planner** | 在首次付费调用前对 paid eval 做 preregister:bars、arms、decoys 与停机规则、跨真实输入的 $0 stub-model dry-run、三个 priced item 的小规模试点,推断与花费表 gate 审批。 |
| `/document-release` | **Technical Writer** | 在每个 ship 上、最终验证和发布前,审计改动行为对项目文档的影响。也能独立跑。展示已更新、已审/现行或被阻塞的文档以及任何剩余缺口。 |
| `/document-generate` | **Documentation Author** | 用 Diataxis 框架从零生成缺失文档。先研究代码库,然后写真实匹配代码的 reference / how-to / tutorial / explanation 文档。可独立调用或从 /document-release 链入当 coverage map 发现缺口时。了解更多:[tutorial](docs/tutorial-document-generate.md) • [how-to](docs/howto-document-a-shipped-feature.md) • [why Diataxis](docs/explanation-diataxis-in-gstack.md)。 |
| `/retro` | **Eng Manager** | 团队感知的每周复盘。每个人的细分、出货 streak、测试健康趋势、增长机会。`/retro global` 跨你所有项目和 AI 工具(Claude Code、Codex、Gemini)跑。 |
| `/browse` | **QA Engineer** | 给 agent 眼睛。优先驱动你的 [Aside](https://aside.com) 浏览器 — 你的真实会话、真实点击、真实截图 — 通过确定性 `aside repl` 脚本。没有 Aside 就回落到 gstack 自带的 Chromium:真实点击,~100ms 每条命令,`/open-gstack-browser` 让你看到 headed 模式带 sidebar、反爬隐身和自动模型路由。所有其它浏览器技能都站在它之上。 |
| `/scrape` | **Data Extractor** | 从网页里抽结构化数据 — 表、列表、价格 — 在你的 Aside 浏览器里用页面的真实登录态。在 fallback 浏览器上,`/skillify` 把流程变成永久 browser-skill,下次跑 ~200ms。 |
| `/setup-browser-cookies` | **Session Manager** | 从 Chrome、Chromium、Brave、Edge、Windows-only Opera 和 Opera GX、或 macOS-only Comet、Arc、Dia 把选中的 cookie 复制到 gstack 自带浏览器。选你的 profile 和 domain;复制与登录验证分两步。仅在 fallback 路径上需要 — Aside 已经有你的会话。 |
| `/autoplan` | **Review Pipeline** | 一条命令,完整审过的 plan。自动跑 CEO → design → DX → eng review(eng 永远最后,所以出货 gate 审最终修订后的 plan),带编码的决策原则。只把 taste 决策亮给你批准。 |
| `/spec` | **Spec Author** | 把模糊意图变成精确可执行 spec,五阶段(why、scope、带强制读代码的 technical、draft、file)。发布前外部审查质量门(Claude Code 在 Codex 上;Codex 在其它 harness 上;7/10 以下 block)、fail-closed 秘密 redact、与已有 issue 去重、归档到 `$GSTACK_STATE_ROOT/projects/$SLUG/specs/` 做团队语料召回。`--execute` 在新 worktree 派生 `claude -p`;/ship 在 merge 时自动关闭源 issue。Plan-mode 感知。 |
| `/learn` | **Memory** | 管理 gstack 在跨会话中学到的东西。审、搜、剪、导出项目专属的模式、坑、偏好。Learnings 跨会话叠加,所以 gstack 在你的代码库上越用越聪明。 |
| `/make-pdf` | **Publisher** | Markdown 进,出版品质文档出。Mermaid 与 excalidraw 栅栏渲染成矢量图,完全离线。图片按页面缩放且绝不截断;宽图单独给横版页。`--to html` 输出一个自包含文件,`--to docx` 输出 Word 文档。 |
| `/diagram` | **Diagram Maker** | 英语进,可编辑的图出。输出三元组:mermaid 源码、`.excalidraw`(可在 excalidraw.com 上打开编辑,手绘风),以及渲染好的 SVG/PNG。零网络。把源码嵌入 markdown,`/make-pdf` 渲染它。 |

### QA without a webpage

对 browser 和非 browser 软件用相同的命令。从仓库的文档化原生命令加一个隔离的本地 fixture 起;指明目标和要查的行为。例如:

```text
/qa-only Test this repo's CLI using its documented local fixture. Check valid and invalid input, exit codes, stdout/stderr, and cancellation. Report only; do not change code or tests. For each finding, include the exact command, expected and actual results, and which checks remain untested. Keep requests inside the fixture; ask before contacting an external service.

/qa Test this repo's local webhook and worker fixture. Explore duplicate deliveries and recovery after a partial failure. Keep all effects inside the fixture; preserve reproduced bugs in native regression tests before repairing them.
```

QA 首先告诉它会用哪个表面、工具和写权限。CLI 或 API 不需要 browser。browser 目标保留真实 browser 测试;DX 审计只在 onboarding、安装或人体工学在 scope 内时加载。如果原生工具或安全 fixture 不可用,报告点出阻塞点和未测的契约,而不是编造一个 pass 或另装一个框架。

探索意味着从每个结果里学习并选下一个有用的挑战,而不是随机跑命令。下一次发现性探测前,QA 写一个短的 `exploration-NNN.json` 证据笔记,记录上次结果、被测的假设和下一条命令。最终报告链上这些笔记;它们不需要探测之间多一条聊天消息。发现的 bug 必须可复现,它的新测试必须在修 bug 之前 fail、修完之后 pass。单元测试保护逻辑;集成和端到端测试保护真实边界,那些 mocks 会藏起来。`/qa-only` 提出这些测试但不写。

普通 `/review` 和 `/ship` 自动在改动行为和附近风险上跑一个有界版本,包括无 plan 或 web server 的小 diff。已有的 fix/test 审批规则仍然适用。缺依赖、被拒的动作和时间限制仍然是可见的 coverage 缺口;短的 smoke pass 绝不等于穷尽 QA。生产访问和破坏性或外部效果需要明确许可。

有界探索用可执行 deadline 守护,而不是估算时钟:它拒绝迟到的探测并在到限时停下自有前台工作。未完成的检查在报告里保持可见。要求的 plan 检查仍在 review/ship smoke 预算之外。见 [QA deadlines](docs/reference-qa-deadlines.md) 的命令、平台和清理限制。

每个 ship 也跑既有的文档审计,包括重复 ship 和已有 PR 更新。明确的事实更正加入最终检出的改动;有风险的改写需要批准。失败的审计停下做恢复或显式接受命名风险,而不是默默丢掉结果。Ship 拥有 versioning、Git 与 PR 发布;docs 助手不会独立 commit 或 push。

### Which review should I use?

| Building for... | Plan stage (before code) | Live audit (after shipping) |
|-----------------|--------------------------|----------------------------|
| **End users** (UI, web app, mobile) | `/plan-design-review` | `/design-review` |
| **Developers** (API, CLI, SDK, docs) | `/plan-devex-review` | `/devex-review` |
| **Architecture** (data flow, perf, tests) | `/plan-eng-review` | `/review` |
| **All of the above** | `/autoplan` (自动跑 CEO → design → DX → eng,自动检测哪些适用;eng 永远最后) | — |

### Power tools

| Skill | What it does |
|-------|-------------|
| `/codex` | **Second Opinion** — 来自 OpenAI Codex CLI 的独立代码审查。review、challenge 和 consult 模式。每个 harness 上都有,除了 Codex。 |
| `/claude-code` | **Second Opinion** — 来自 Claude Code 的独立代码审查。review、challenge、consult 模式,consult 带会话连续性。每个 harness 上都有,除了 Claude Code。 |
| `/careful` | **Safety Guardrails** — 在破坏性命令前告警(rm -rf、DROP TABLE、force-push)。说 "be careful" 激活。可覆盖任何 MEDIUM 告警;根/家目录递归删除与默认分支 force-push 是硬拒绝。 |
| `/freeze` | **Edit Lock** — 把文件编辑限制在一个目录。调试时防止意外改出范围。 |
| `/guard` | **Full Safety** — 一条命令同时 `/careful` + `/freeze`。生产工作最大安全。 |
| `/unfreeze` | **Unlock** — 解除 `/freeze` 的边界。 |
| `/open-gstack-browser` | **GStack Browser** — 把 gstack 自带浏览器起成 headed,带 sidebar、反爬隐身、自动模型路由(动作用 Sonnet,分析用 Opus)、一键 cookie 导入、与 Claude Code 集成。清理页面、智能截图、编辑 CSS,把信息回传给终端。Fallback 引擎的可视面孔;Aside 打开时你在那里看 agent 的 tab。 |
| `/setup-deploy` | **Deploy Configurator** — `/land-and-deploy` 的一次性 setup。检测平台、生产 URL 和部署命令。 |
| `/setup-gbrain` | **GBrain Onboarding** — 从零到跑起来 gbrain 不到 5 分钟。PGLite local、Supabase 已有 URL 或通过 Management API 自动新建一个 Supabase 项目。Claude Code 的 MCP 注册 + 每个 repo 的 trust 三元组(read-write/read-only/deny)。[完整指南](USING_GBRAIN_WITH_GSTACK.md)。 |
| `/sync-gbrain` | **Keep Brain Current** — 把这个 repo 的代码通过 `gbrain sources add` + `gbrain sync --strategy code` 重新索引到 gbrain,刷新 CLAUDE.md 里的 `## GBrain Search Guidance` 块,在能力检查失败时自动移除该 guidance。`--incremental`(默认)、`--full`、`--dry-run`。幂等;可安全重跑。 |
| `/gstack-upgrade` | **Self-Updater** — 把 gstack 升到最新版。检测全局 vs vendored 安装,同步两边,显示变更。 |
| `/ios-qa` | **iOS Live-Device QA (v1.43.0.0+)** — 通过嵌入式 `StateServer` 在 app 内通过 USB CoreDevice 驱动真实 iPhone 或 iPad。读 Swift 源,codegen 类型化的 `@Observable` accessor,跑 agent 循环。可选 `--tailnet` 标志把设备暴露给 OpenClaw 或任何 Tailscale tailnet 上支持 HTTP 的 agent,让远程 agent 跑 iOS QA 而无需碰硬件。能力层级 allowlist(observe/interact/mutate/restore)、每设备 session 锁、审计日志。 |
| `/ios-fix`, `/ios-design-review`, `/ios-clean`, `/ios-sync` | iOS bug-fix 循环、设计者视角的 HIG 审计、debug-bridge 清理、accessor 重新同步。见 `docs/skills.md`。端到端演练:[docs/howto-ios-testing-with-gstack.md](docs/howto-ios-testing-with-gstack.md)。 |

### Standalone binaries

除斜杠命令技能外,gstack 也为不属于会话的工作流发布独立 CLI:

| Command | What it does |
|---------|-------------|
| `gstack-model-benchmark` | **Cross-model benchmark** — 用同样的 prompt 跑 Claude、GPT(经 Codex CLI)、Gemini;对比延迟、token、成本、以及(可选)LLM-judge 质量分。按 provider 检测 auth,不可用的 provider 干净跳过。输出表格、JSON 或 markdown。`--dry-run` 验证 flag + auth,不花 API 调用。 |
| `gstack-taste-update` | **Design taste learning** — 把来自 `/design-shotgun` 的批准和拒绝写进持久化的 per-project taste profile。5%/周衰减。反馈到未来变体生成,让系统学到你实际选什么。 |
| `gstack-egress` | **Egress receipt auditor** — 每个 gstack 发起的离机发送在发送前向 `~/.gstack/security/egress.jsonl` 写一个防篡改的、hash 链式收据。`list` 显示 gstack 尝试发了什么、到哪个 host;`grants` 显示长期同意设置加上撤销每条的具体命令;`verify` 重新算 hash chain,在篡改时退出 3(捕捉编辑、重排、链中删除;截断或删除账本本身不在范围内 — 它是取证日志,不是防篡改存储)。 |
| `gstack-context-bill` | **Token bill-of-materials** — 只读、离线的 token 审计:已装技能树成本:每会话必付的 always-on frontmatter vs per-invocation SKILL.md + forced references。`--diff` 比较两棵树,`--budget` 设上限,`--exact` 选 Anthropic `count_tokens`(发文件文本到主机;先写 egress 收据,若写不了回退到离线估算)。 |
| `gstack-code-intelligence` | **Code-intelligence provider picker** — 把 GBrain、Sourcebot、Graphify 装在一个接口背后:`options`/`status` 看可用的,`select` 选一个,`index`/`search` 用它,`suggest` 检查要不要在这里触发一次性索引 offer。offer 在大 repo(1,000+ tracked file)触发;拒绝被持久化)。非本地 provider 在你记录 per-repo 同意前拒绝 index 或 search(`consent <repo> yes\|no` — 查询文本是 repo 衍生内容),per-repo 信任策略的 deny 和 read-only 层级否决 write-class 操作不顾同意,每次离机发送写一条 egress 收据。完全可选 — 没选时 gstack 回退 grep。 |
| `gstack-verify-gate` | **Verification stop hook (opt-in)** — 在项目声明的 verify 命令通过前阻断 Claude Code 的 turn 结束(3 次被阻断重新进入后它用响亮的 still-RED 警告让步而不是无限循环)。在 CLAUDE.md 一行声明:`<!-- gstack:verify: bun test -->`。Hook 绕过权限系统,所以声明的命令在你每个 repo 信任一次之前不跑(`gstack-verify-gate --trust`);编辑该命令会让信任失效直到重新授权,每次授权都进审计日志。`./setup` 不会为你注册 — 用 `gstack-settings-hook add-event --event Stop --command ~/.claude/skills/gstack/bin/gstack-verify-gate --source verify-gate` opt-in,`gstack-settings-hook remove-source --source verify-gate` 移除。 |
| `gstack-memorable` | **Memorable recall bridge (opt-in, third party, Claude Code only)** — 通过 gstack 把 Claude Code 接到外部 [Memorable](https://memorable.sh) CLI,而不是用 vendor 自家的安装器,所以 hook 拿到 gstack 的保证:显式同意 key(`memorable_recall`,默认关闭,被 `gstack-egress grants` 列出)、每次 prompt 移交 fail-closed egress 收据(`gstack-egress list --sink memorable-recall`)、HIGH 层级密钥预扫、信任信封和返回内容的 8 KiB 上限,vendor 进程的 allowlist 环境与进程组隔离,以及干净移除。`enable` 在稳定安装点用 5 秒超时注册 hook 且绝不跑 vendor 自己的同意命令;`disable` 先撤销 gate 再按身份移除条目,即使 Claude Code 剥掉 tag;`status` 只读。gstack 从不安装 Memorable,其二进制发了什么是 vendor 的说辞而非 gstack 的。Windows 上暂不可用。[完整指南](docs/memorable-workflow-memory.md)。 |
| `gstack-wtree` | **Working-tree fingerprint** — 打印磁盘上真实内容的 content hash(从 stat 缓存播种的临时索引,比完整重 hash 便宜约 40 倍;未跟踪的源被数,gitignored 的 scratch 不数)。相同内容跨 commit、rebase、amend、squash 指印相同 — 这是把 review 和测试证据绑到内容而不是 commit SHA 上的东西。 |
| `gstack-review-log` | **Review-pass receipts** — `--start <skill>` 在 diff review 前捕捉工作树指印;`'<JSON>' --finish <token>` 消费这个一次性、repo/branch/skill 范围的收据。绑定要求匹配的 start/end 内容和审查者报告的 `completed:true` 与 `converged:true`;它不是模型读源码的独立证明。`--check-shared-libs <token>` 从 stdin 读一个当前 finding 为 JSON,检查之前 Skip 决定是否对得上真实 branch、capture 与合格原始源码 blob。证明缺失或不安全时返回 `reusable:false`,包括没有 logger-versioned coverage 的旧记录。最终 review 日志算共享代码指印与 coverage,而非相信提供的证明。 |
| `gstack-review-read` | **Review freshness** — 输出 review 记录,带计算出的 `review_freshness.status` 与 `reason`:CURRENT、STALE 或 UNVERIFIED(用于 diff review)。`/ship` 和 `/land-and-deploy` 用同一等级;仅匹配的 commit 从不为 diff review 出证书。[Dashboard rules](docs/skills.md#review-readiness-dashboard)。 |
| `gstack-evidence` | **Verification-evidence ledger** — `run --label <lane> -- <cmd>` 透明包装任何测试命令(子进程退出码始终透传)并记录什么跑过哪个工作树指印;`check` 按 FRESH/STALE/MISSING 给每个 label 评级,用 `--expect-cmd`、`--max-age` 与 `--allow-paths` 绑定。/ship 与 /land-and-deploy 引用 fresh evidence 而非重跑套件。每次运行日志权限 0600,2MB 上限,30 天后清理;账本与日志按设计只在本机。 |
| `gstack-issue-guard` | **Tracker-text trust envelope** — 拉取 GitHub issue/PR 文本(`issue <n>`、`pr-body`、`pr-comments` 或 `--stdin`)并包在带标签的 envelope 里,这样 agent 把它当数据:注入形状的行被标,即使穿过全角和不可见字符的规避,伪造的 envelope 横幅也被解除。gstack 里每条 tracker-text 入口都走它,由 CI 扫描器强制。 |
| `gstack-ios-qa-daemon` | **iOS QA daemon** — Mac 端在 agent 与通过 USB CoreDevice 连接的 iPhone/iPad 之间的 broker。默认 loopback;`--tailnet` 打开面向 Tailscale 的监听器,带身份门控的能力层级。单实例通过 `~/.gstack/ios-qa-daemon.pid` 的 flock 守护。见 [docs/howto-ios-testing-with-gstack.md](docs/howto-ios-testing-with-gstack.md)。 |
| `gstack-ios-qa-mint` | **iOS allowlist manager** — tailnet allowlist 的 owner-grant CLI。对 `~/.gstack/ios-qa-allowlist.json`(mode 0600)做 `grant`/`revoke`/`list`。远程 agent 永不自动加入 allowlist;这是显式意图路径。 |
| `gstack-ios-qa-regen` | **iOS bridge regenerator** — 确定性安装规范的 DebugBridge 包,生成类型化的 state accessor,记录已装的 gstack 版本。源码变更或升级后安全重跑。 |

私有的付费 CSO 评估 producer 是一个打包合约,不是普通的 `bun run build` 产物。在 macOS 或 Linux 上,发布 operator 在与 `bun run build:cso` 同一个干净 build 会话里用文档化的加固 Bun 标志编译 `cso-eval-producer`,然后把它装到 `gstack-cso-launcher`、`gstack-cso-core`、`gstack-cso-watchdog` 和隐藏的 `.gstack-cso-generation` manifest 旁边,作为一个 root-owned、不可写的五工件单元。producer 拒绝 root 执行、可写/symlinked/不完整的安装,以及未审过的 provider CLI 版本;每条收据绑定全部五个工件 hash,收集时拒绝来自不同单元身份的收据。见 [clean producer procedure](test/fixtures/cso-eval/README.md#trusted-five-artifact-producer-unit)。付费 producer 评估在 Windows 上仍不可用,因为 detached watchdog 没有 Windows 构建。

每个 producer 拿到一个策划的 one-cell 源拷贝,目录权限 `0555`、文件 `0444`,加 pre/post content、Git、模式检查。Claude 收不到源 add-directory,因为 add-directory 是 Claude Code 的 bubblewrap 留空 `.mcp.json` mount point 的沙箱写根。它的 Bash 跑在 Claude Code 沙箱里,但有一个例外:`--allowed-tools` 指名的精确绝对 launcher 路径,它在沙箱外跑,对用户也是这样。launcher 是边界。它把环境换成固定的 allowlist,只用本地 Docker Unix socket,在加固容器里跑每个 target。producer 通过 `--settings` 传这个策略(沙箱启用、不可用就失败、不要非沙箱回退、不要 auto-allow,`excludedCommands` 只指 launcher)。除非生成参数与环境携带精确这个策略,否则它拒绝启动 agent(`CLAUDE_PRODUCER_SANDBOX_POLICY_MISMATCH`)。`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` 保持不设,因为它会把 launcher 拉回沙箱。`--allowed-tools` 拒绝其它 Bash 命令,Write 限制在 provider 工作目录与 `cso-home`。Gemini 收不到源工作目录或 include-directory。Codex 技术上收到对精确策划源根的只读文件系统访问,因为受信 helper 继承 Codex 权限 profile;它私有的 provider 工作目录与精确 `cso-home` 产物目录是唯一的写根,其它根路径都保持被拒,producer prompt 要求通过 helper 访问源。这是对不可变公共 fixture 的评估隔离,不是说 Codex 不能直接读那个 fixture。

把每个已完成 cell 的收据连同 `state/cso-home/security/cso/` 复制到受信裁决主机。收据条目是相对该目录的排序路径,绑定每个保留文件的大小与 SHA-256 加一个汇总清单 hash。Provider home、settings、session 与凭证在单独的 disposable 目录下,在 cell 后移除;它们从来不是保留产物树的一部分。裁决者必须基于收据对传输过来的树重新 hash 再信任报告或修复 bundle。

付费 producer 资格使用这个明确的 host 矩阵:

| Producer host | Daily/static cells | Comprehensive target execution |
|---|---|---|
| Codex CLI 0.153.4 | 在自定义权限 profile 下支持 | 因为 Docker/socket 访问没被授权给模型命令沙箱而 fail-closed;报告 setup 缺口为 partial |
| Claude Code 2.1.263 | 在受限 safe mode 下支持 | 受限 launcher 子进程无法触达 Docker 时 fail-closed;报告 setup 缺口为 partial |
| Gemini CLI 0.59.0 | 在隔离 home/settings 下支持 | 私有发布资格主机:精确的 `run_shell_command(<launcher>)` 策略可以在不暴露通用 shell 的情况下跑受信 helper |

每个被 setup 阻断的 comprehensive cell 在 release-gate 分母里仍然是 miss。评估者不会在它的隔离策略阻止所需 setup 时,悄悄把 host 或工作流算作支持。

`./setup` 还在 `~/.claude/settings.json` 注册一个默认开启的 Stop hook:`gstack-timeline-stop`(在会话被打断时关闭悬空的 session-timeline 条目;fail-open — 2 秒内部预算,总是退出 0,永远不阻塞会话)。用 `./setup --no-timeline-stop-hook` 持久 opt-out — 选择落到 `timeline_stop_hook` 配置 key,跨升级存活,显式 "no" 移除已有注册。`GSTACK_TIMELINE_STOP_HOOK=no` 与 `gstack-config set timeline_stop_hook no` 也行(flag > env > config)。`./setup --no-team` 对那次运行跳过,`gstack-settings-hook remove-source --source gstack-timeline-stop` 手动移除,`gstack-uninstall` 也会移除。

Hook 注册是 canonical-only:每个 hook 命令指向稳定 `~/.claude/skills/gstack` 安装,而不是 setup 跑的那个 tree,所以删 worktree 或 Conductor workspace 不会留下死 hook 在你的会话里报错。每次 `./setup` 也先 heal:`gstack-settings-hook prune-stale --repoint` 移除死 gstack hook 条目,把过期的重指到稳定安装,折叠重复,只在改了东西时打印一行(并备份写在文件旁边)。

### Domain skills + raw CDP escape hatch

gstack 自家引擎里的两个浏览器原语(Aside 不在时的 fallback 路径)让 agent 随时间叠加:

- **`$B domain-skill save`** — agent 保存一个 per-site 笔记(如 "LinkedIn 的 Apply 按钮在 iframe 里"),下次访问该 hostname 时自动触发。quarantine → 3 次成功使用后 active → 可选通过 `$B domain-skill promote-to-global` 做跨项目升级。存储在 `/learn` 的 per-project learnings 文件旁边。完整参考:**[docs/domain-skills.md](docs/domain-skills.md)**。
- **`$B cdp <Domain.method>`** — 罕见情况下绕过 curated commands 的原始 Chrome DevTools Protocol 后门。默认拒绝:方法必须显式加进 `browse/src/cdp-allowlist.ts` 并附一行说明。两层 mutex 把 browser-scoped CDP 调用与 per-tab 工作串行。数据外泄方法的输出包在 UNTRUSTED 信封里。

> 想用没围栏、没 allowlist、没 daemon 的裸 CDP — 只是从 agent 到 Chrome 的薄传输? [browser-use/browser-harness-js](https://github.com/browser-use/browser-harness-js) 是一种不同的哲学(agent-authored helpers vs gstack 的 curated commands),适合你不想用 gstack 安全栈的场景。两者可以共存:gstack 的 `$B cdp` 与 harness 都能通过 Playwright 的 `newCDPSession` 接到同一个 Chrome。

**[每个技能的深度解析、例子和哲学 →](docs/skills.md)**

### Karpathy's four failure modes? Already covered.

Andrej Karpathy 的 [AI coding rules](https://github.com/forrestchang/andrej-karpathy-skills)(17K stars)点出四种失败模式:错误假设、过度复杂、正交编辑、命令式而非声明式。gstack 的工作流技能强制所有四种。`/office-hours` 在写代码前把假设暴露出来。Confusion Protocol 阻止 Claude 在架构决策上瞎猜。`/review` 抓不必要的复杂性和顺手编辑。`/ship` 把任务变成可验证的目标,test-first 执行。如果你已经在用 Karpathy 风格的 CLAUDE.md 规则,gstack 是让它们跨整个 sprint(而不仅是单个 prompt)落地的 workflow enforcement layer。

## Parallel sprints

gstack 单 sprint 就很好用。10 个并行就变有趣了。

**Design is at the heart.** `/design-consultation` 从零搭建你的设计系统,调研现有的,提出有创造性的风险,写 `DESIGN.md`。但真正的魔法是 shotgun-to-HTML 流水线。

**`/design-shotgun` 用来探索。** 你描述想要什么。它用 GPT Image 生成 4-6 个 AI mockup 变体。然后在浏览器开一个对比板,所有变体并排。你挑喜欢的,留反馈("留白多些"、"标题更大胆"、"去掉渐变"),它就生成下一轮。重复到你爱上。品味记忆几轮后开始偏向你实际喜欢的东西。不用再用文字描述愿景然后希望 AI 懂。你看选项,挑好的,可视化迭代。

**Works with impeccable.** 如果你也用 [impeccable](https://impeccable.style),gstack 不跟它打架。当你有它时,gstack 在 `/design-review`、`/review`、`/ship` 和 `/design-html` 里把 impeccable 的确定性引擎作为 pre-pass 跑(gstack 从不跑 impeccable 的 installer 或 launcher;首次设计技能发现没有引擎时,只问一次是否下载引擎二进制,checksum-pinned 并记日志,放到 `~/.impeccable`,记住你的回答),用自己的声音说同样的 61 个 rule id,读 `PRODUCT.md`,以两个工具都读的开放 DESIGN.md 格式写 `DESIGN.md`,把延后的 finding 交给 `/impeccable <command>`。说不就什么都不变:不催,不漏步骤。`gstack-config set design_detector off` 关掉 pre-pass。gstack 从 impeccable 与 DESIGN.md spec 派生的材料归属见 `NOTICE.md`。

**`/design-html` 让它落地。** 把那个已批准的 mockup(从 `/design-shotgun`、CEO plan、design review,或只是一段描述)变成生产级 HTML/CSS。不是那种在一个视口宽度看着好、其它地方全坏掉的 AI HTML。它用 Pretext 做计算式文本布局:文本真的随 resize 重排,高度自适应内容,布局是动态的。30KB 开销,零依赖。检测你的框架(React、Svelte、Vue)输出对应格式。智能 API 路由根据是 landing page、dashboard、form 还是 card 布局选不同 Pretext 模式。输出是你真会 ship 的东西,不是 demo。

**`/qa` 是个大解锁。** 它让我从 6 个并行 worker 走到 12 个。Claude Code 说 *"我看到问题了"* 然后真修,生成回归测试,验证修复 — 改变了我的工作方式。Agent 现在有眼睛了。

**Smart review routing.** 就像在一个运行良好的 startup:CEO 不用看基础设施 bug 修复,后端改动不需要 design review。gstack 跟踪跑了哪些 review,算出哪些合适,然后直接做对的事。Review Readiness Dashboard 告诉你在 ship 前你处在哪个位置。

**Test everything.** `/ship` 在你项目没有测试框架时从零引导出一个。每个 `/ship` 跑出一份 coverage 审计。`/qa` 在有基础设施时创建原生回归并显式报告缺失的测试 coverage;仅 CSS 修复可以用浏览器证据代替。100% 测试覆盖是目标 — 测试让 vibe coding 安全,而不是 yolo coding。

**`/document-release` 是你从未有过的工程师。** 它在最终验证前审计相关作者文档对发布 diff,纠正事实漂移。有风险的改动返回批准。`/ship` 每次跑都调用它并拥有 TODOS、发布元数据、生成与发布;子进程报告 updated、current 或 blocked 的文档。

**Aside is the browser gstack drives first.** 对于 browser 表面,在 macOS 上 [Aside](https://aside.com) AI 浏览器打开时,`/qa`、`/qa-only`、`/design-review`、`/canary`、`/benchmark`、`/scrape` 和 `/browse` 都在那里跑 — 你的真实浏览器、真实登录会话、在 agent 为自己开并用完关掉的 tab 里。无需 cookie import、无 "打开浏览器" 步骤、无 CAPTCHA 接力:撞上登录墙,在 Aside 里登录,说 "done",agent 继续。页面返回的任何东西都视为不可信内容 — agent 从中拿语法,不拿指令。`/make-pdf`、`/diagram` 和设计预览也通过 Aside 打印与截图(从本机 loopback 服务,每个脚本一份渲染),规划技能在做 web research 时也优先走 Aside 自己的 agent 而不是 search tool。

**When Aside isn't there, gstack's own browser takes over — automatically.** Linux、Windows、或关闭 Aside 的 Mac:同样的技能用 `./setup` 构建的 bundled headless Chromium,产出同样的证据,并点亮下面那些只在 browser 是 gstack 的而不是你的时候才有意义的功能。

**Real browser mode.** `/open-gstack-browser` 启动 GStack Browser,一个 AI 控制的 Chromium,带反爬隐身、自定义品牌与烤进去的 sidebar 扩展。Google、NYTimes 之类无需 captcha。菜单栏写 "GStack Browser" 而不是 "Chrome for Testing"。你的常规 Chrome 不受影响。所有现有 browse 命令不变。`$B disconnect` 回到 headless。浏览器在窗口打开期间一直在 — 不会有闲置超时在你工作时杀掉它。

**Sidebar agent — your AI browser assistant.** 在 Chrome 侧栏输入自然语言,一个 child Claude 实例执行它。"导航到设置页并截图。""用测试数据填这个表单。""遍历列表里的每一项并抽出价格。"sidebar 自动路由到对的模型:Sonnet 跑快动作(click、navigate、screenshot),Opus 跑阅读与分析。每个任务最多 5 分钟。sidebar agent 跑在隔离会话,不会干扰你的主 Claude Code 窗口。Sidebar 底部一键 cookie 导入。

**Personal automation.** Sidebar agent 不只是给开发工作流用。例子:"浏览我娃学校的家长门户,把其他家长的姓名、电话和照片都加进我的 Google Contacts。"两种认证方式:(1) 在 headed 浏览器里登录一次,会话持久化,或 (2) 点 sidebar 底部的 "cookies" 按钮从你真实的 Chrome 导入 cookie。一旦认证,Claude 导航目录,抽数据,创建联系人。

**Prompt injection defense.** 敌对网页试图劫持你的 sidebar agent。gstack 带一层防御:每次页面读取的内容过滤器(datamarking、hidden-element stripping、ARIA scrubbing、URL blocklist),加上 22MB ML 分类器在 sidecar 子进程里本地跑,在 agent 看到页面衍生内容前扫描,verdict combiner 要求分类器一致才阻断(防止 Stack Overflow 风格指令页上的单模型假阳)。一切都在你本机跑,无网络。紧急 kill switch:`GSTACK_SECURITY_OFF=1`。完整栈见 [ARCHITECTURE.md](ARCHITECTURE.md#prompt-injection-defense-sidebar-agent)。

**Browser handoff when the AI gets stuck.** 撞上 CAPTCHA、auth 墙或 MFA 提示?`$B handoff` 在同一页面打开一个可见的 Chrome,所有 cookie 和 tab 都完好。解决问题,告诉 Claude 你搞定了,`$B resume` 从刚才的位置继续。agent 在连续 3 次失败后甚至会自动建议。

**`/pair-agent` 是跨 agent 协调。** 你在 Claude Code 里。你还有 OpenClaw 在跑。或 Hermes。或 Codex。你想它们俩看同一个网站。敲 `/pair-agent`,选你的 agent,一个 GStack Browser 窗口打开让你能看到。技能打印一段指令。把那段贴到另一个 agent 的聊天里。它用一次性 setup key 换 session token,创建自己的 tab,开始浏览。你看到两个 agent 在同一个浏览器里工作,各在各自的 tab,互不干扰。如果装了 ngrok,隧道自动起,所以另一个 agent 可以在完全不同的机器上。同机 agent 走零摩擦捷径,直接写凭证。这是不同厂商的 AI agent 第一次能通过共享浏览器真实安全地协作:scoped token、tab 隔离、限速、域名限制和活动归属。

**Multi-AI second opinion.** 在 Codex 上,gstack 通过 `/claude-code` 把外部审查送到 Claude Code。在 Claude Code 上,/codex 把它们送到 OpenAI Codex。其它 harness 暴露两个技能并在支持自动外部审查时用 Codex。每个技能支持代码审查、对抗挑战、带会话连续性的咨询。路由跟随 harness,所以改你的配置模型不改外部审查者。报告标识实际完成每个审查的 provider;不可用的外部覆盖保持可见。

**Safety guardrails on demand.** 说 "be careful",`/careful` 在任何破坏性命令前告警 — rm -rf、DROP TABLE、force-push、git reset --hard。`/freeze` 在调试时把编辑锁在一个目录,Claude 不能意外 "修" 不相关的代码。`/guard` 同时激活。`/investigate` 自动 freeze 到被调查的模块。

**Proactive skill suggestions.** gstack 注意到你处在哪个阶段 — brainstorm、review、debug、test — 建议对的技能。不喜欢?说 "stop suggesting" 它跨会话记住。

## 10-15 parallel sprints

gstack 单 sprint 就强。10 个并行就变了。

[Conductor](https://conductor.build) 并行跑多个 Claude Code 会话 — 每个在隔离的 workspace 里。一个跑 `/office-hours` 做新想法,另一个在 PR 上跑 `/review`,第三个实现 feature,第四个在 staging 上跑 `/qa`,还有 6 个在别的 branch 上。所有都同时。我经常跑 10-15 个并行 sprint — 这就是目前的实用上限。

Sprint 结构就是让并行起作用的地方。没有流程,10 个 agent 就是 10 份混乱的源头。有流程 — think、plan、build、review、test、ship — 每个 agent 知道做什么、何时停。你管理它们的方式就像 CEO 管团队:在关键决策上 check in,让其余的跑。

### Voice input (AquaVoice、Whisper 等)

gstack 技能有语音友好的触发短语。说你想说的 — "run a security check"、"test the website"、"do an engineering review" — 正确的技能就激活。你不需要记斜杠命令名或缩写。

## Uninstall

### Option 1: 跑卸载脚本

如果你本机装了 gstack:

```bash
~/.claude/skills/gstack/bin/gstack-uninstall
```

它处理 skills、符号链接、全局 state(`~/.gstack/`)、项目本地 state、browse daemons 和临时文件。`--keep-state` 保留配置和分析。`--force` 跳过确认。

### Option 2: 手动移除(无本地 repo)

如果你没有克隆仓库(例如你通过 Claude Code 一段粘贴装的,然后删了克隆):

```bash
# 1. 停 browse daemons
pkill -f "gstack.*browse" 2>/dev/null || true

# 2. 移除 SKILL.md 指到 gstack/ 的每 skill 目录
#    (rm -rf,不是 rmdir — 已装目录还含 runtime-asset link)
find ~/.claude/skills -mindepth 1 -maxdepth 1 -type d ! -name gstack 2>/dev/null |
while IFS= read -r dir; do
  link="$dir/SKILL.md"
  [ -L "$link" ] || continue
  target=$(readlink "$link" 2>/dev/null) || continue
  case "$target" in
    gstack/*|*/gstack/*)
      rm -rf "$dir"
      ;;
  esac
done
# gstack 创建的目录带一个 .gstack-owned 标记(Windows 上唯一信号,因为
# 安装是文件拷贝,没符号链接可读)
for marker in ~/.claude/skills/*/.gstack-owned; do
  [ -f "$marker" ] && rm -rf "$(dirname "$marker")"
done
# Alias skills 装成拷贝(没符号链接可检测) — 按名移除
rm -rf ~/.claude/skills/_gstack-command ~/.claude/skills/connect-chrome 2>/dev/null

# 3. 移除 gstack
rm -rf ~/.claude/skills/gstack

# 4. 移除全局 state
rm -rf ~/.gstack

# 5. 移除集成(跳过你从没装过的)
rm -rf "${CODEX_HOME:-$HOME/.codex}/skills/gstack"* 2>/dev/null
rm -rf ~/.factory/skills/gstack* 2>/dev/null
rm -rf ~/.kiro/skills/gstack* 2>/dev/null
rm -rf ~/.openclaw/skills/gstack* 2>/dev/null
rm -rf ~/.cursor/skills/gstack* 2>/dev/null
rm -rf ~/.config/opencode/skills/gstack* 2>/dev/null
rm -rf ~/.gemini/antigravity-cli/skills/gstack* 2>/dev/null

# 6. 移除临时文件
rm -f /tmp/gstack-* 2>/dev/null

# 7. 每项目清理(在每个项目根跑)
rm -rf .gstack .gstack-worktrees .claude/skills/gstack 2>/dev/null
rm -rf .agents/skills/gstack* .factory/skills/gstack* 2>/dev/null
```

手动移除会让 gstack 的 hook 条目留在 `~/.claude/settings.json`(卸载脚本为你移除所有,包括 `_gstack_source` tag 被剥的那些)。编辑那个文件,删掉所有 hook 命令路径指到 `.claude/skills/gstack/` 的:SessionStart auto-update hook、AskUserQuestion PreToolUse/PostToolUse hooks 与 Stop hooks(session timeline,加上 verify-gate 如果你 opt-in)。留着,一旦安装目录没了,它们就会在每个匹配事件上报错。

### Clean up CLAUDE.md

卸载脚本不编辑 CLAUDE.md。在每个加了 gstack 的项目里,移除 `## gstack` 与 `## Skill routing` 段落。

### Playwright

`~/Library/Caches/ms-playwright/`(macOS)保留,因为其它工具可能共享。如果你不需要别的东西可以删。

### Aside

gstack 从没装过 Aside,所以从不卸载它。跟任何 app 一样保留或移除。

---

免费,MIT 协议,开源。没有高级版,没有 waitlist。

我开源了我做软件的方式。你可以 fork 并变成你的。

> **We're hiring.** 想用 AI 编码的速度 ship 真实产品并一起加固 gstack?来 YC 工作 — [ycombinator.com/software](https://ycombinator.com/software)极有竞争力的薪资与股权。旧金山,Dogpatch 区。

## GBrain — persistent knowledge for your coding agent

[GBrain](https://github.com/garrytan/gbrain) 是 AI agent 的持久化知识库 — 把它想成 agent 真正在会话之间保留的记忆。GStack 给了一条命令的路径从零到 "它在跑了,我的 agent 能调用它。"

```bash
/setup-gbrain
```

四条路径,选一条:

- **Supabase,已有 URL** — 你的云 agent 已经 provision 了一个 brain;粘贴 Session Pooler URL,现在这台笔记本用同一份数据。
- **Supabase,自动 provision** — 粘贴 Supabase Personal Access Token;技能创建新项目,poll 到 healthy,拉 pooler URL,交给 `gbrain init`。端到端 ~90 秒。
- **PGLite local** — 零账号、零网络、~30 秒。这台 Mac 上的隔离 brain。先试再买;之后用 `/setup-gbrain --switch` 迁到 Supabase。
- **Remote gbrain MCP** — 你的 brain 跑在另一台机器(Tailscale、ngrok、内网 LAN)或队友服务器;粘贴 MCP URL 和 bearer token。可选搭配本地 PGLite 做 split-engine mode 下的符号感知代码搜索。跨机器记忆的最佳选择,不用起本地 DB。

init 后,技能会主动把 gbrain 注册成 Claude Code 的 MCP server(`claude mcp add gbrain -- gbrain serve`),让 `gbrain search`、`gbrain put` 等成为一等类型化工具 — 而不是 bash shell 调用。

**保持 brain 最新。** 在任何 repo 跑 `/sync-gbrain` 把它的代码重新索引到 gbrain(默认增量,`--full` 全量 reindex,`--dry-run` 预览)。技能通过 `gbrain sources add` 把 cwd 注册成联邦源,跑 `gbrain sync --strategy code`,把 `## GBrain Search Guidance` 块写到项目 CLAUDE.md,让 agent 优先用 `gbrain search`/`code-def`/`code-refs` 而非 Grep。能力检查失败时该块自动移除 — 不会有指向未装工具的过期 guidance。

**Per-remote trust policy.** 你机器上每个 repo 拿三种 tier 中的一种:

- `read-write` — agent 既能搜 brain 又能从该 repo 写回新页
- `read-only` — agent 能搜但不写(最适合多 client 顾问:搜共享 brain,在 client B 的 repo 时别被 client A 的工作污染)
- `deny` — 完全不交互 gbrain

技能对每个 repo 问一次。决策在同一 remote 的 worktree 和 branch 之间是粘性的。

**GStack memory sync(不同功能,同一私有 repo 基建)。** 可选把 gstack state(learnings、CEO plans、设计文档、retros、开发者 profile)推到私有 git repo,让记忆跨机器跟随你走,带一次性隐私提示(全 allowlist / 仅 artifact / 关)与纵深防御秘密扫描,在 AWS key、token、PEM block、JWT 离开你机器前阻断它们。

```bash
gstack-artifacts-init
```

**在 Conductor 里跑 gstack?** Conductor 显式地从每个 workspace 的进程 env 中剥 `ANTHROPIC_API_KEY` 与 `OPENAI_API_KEY`,所以付费 eval 和 gbrain embeddings 不会开箱即用。改为在 Conductor 的 workspace env 配置里设 `GSTACK_ANTHROPIC_API_KEY` 与 `GSTACK_OPENAI_API_KEY` — gstack 的 TS 入口在运行时把它们提升为规范名。完整细节与新入口 import 的贡献者清单:[Conductor + GSTACK_* env vars](USING_GBRAIN_WITH_GSTACK.md#conductor--gstack_-env-vars)。

**全套 — 每个场景、每个 flag、每个 bin helper、每个故障排除步骤:** [USING_GBRAIN_WITH_GSTACK.md](USING_GBRAIN_WITH_GSTACK.md)

其他参考:[docs/gbrain-sync.md](docs/gbrain-sync.md)(sync 专属指南) • [docs/gbrain-sync-errors.md](docs/gbrain-sync-errors.md)(错误索引)

## Docs

| Doc | What it covers |
|-----|---------------|
| [Skill Deep Dives](docs/skills.md) | 每个技能的哲学、例子和工作流(含 Greptile 集成) |
| [Diagrams & Document Formats](docs/howto-diagrams-and-formats.md) | PDF 里的 Mermaid/excalidraw 栅栏、图片缩放与安全默认、`--to html\|docx`、`/diagram` 三元组 |
| [Builder Ethos](ETHOS.md) | Builder 哲学:Boil the Ocean、Search Before Building、三层知识 |
| [Using GBrain with GStack](USING_GBRAIN_WITH_GSTACK.md) | `/setup-gbrain` 的每条路径、flag、bin helper 与故障排除步骤 |
| [GBrain Sync](docs/gbrain-sync.md) | 跨机器记忆 setup、隐私模式、故障排除 |
| [Architecture](ARCHITECTURE.md) | 设计决策与系统内部 |
| [Browser](BROWSER.md) | gstack 怎么优先驱动 Aside(契约、cookbook、渲染、调研)、fallback 引擎何时接管,以及 fallback 的完整 `$B` 命令参考 |
| [Contributing](CONTRIBUTING.md) | Dev setup、测试、contributor mode 与 dev mode |
| [Memorable recall bridge](docs/memorable-workflow-memory.md) | 通过 gstack 的可选第三方 workflow 记忆:两次同意、gstack 移交什么且能证明什么、移除、故障排除 |
| [Troubleshooting](docs/troubleshooting.md) | 每条 `not run` / `unavailable` 消息、它的意思与修法 |
| [Changelog](CHANGELOG.md) | 每个版本的新东西 |

## Privacy & Telemetry

除非 opt-in,gstack 不发任何关于你怎么用它的东西。三种网络流量存在,每种都有自己的开关。

**Telemetry(opt-in,默认关)。** 首次运行时 gstack 问一次。你可以说不,`gstack-config set telemetry off` 随时关。

- **每次技能运行发什么:** 技能名、事件类型、outcome(success/error/abort)、duration、gstack 版本、OS 与 CPU 架构、是否用了 browser、运行是 live 还是 test、session id、live session marker 数量(下)、错误 class、failed step,以及经过 gstack 秘密 redact 的错误消息(裁到 200 字符)。`community` 加一个稳定 device ID;`anonymous` 不发 ID。Prompt-injection 检测加攻击页面的 domain、payload hash、置信度、检测 layer 与 verdict。
- **绝不发什么:** 代码、文件路径、repo 名、branch 名、prompts 或你写的任何其它内容。
- **留在你机器上的:** opt-in 后,每次运行也记到 `~/.gstack/analytics/skill-usage.jsonl`(在你的 state root 下;见 [docs/state-root.md](docs/state-root.md)),带 repo 名与 branch,所以 `gstack-analytics` 能显示你自己的面板。`gstack-telemetry-sync` 在上传前删除这两个字段(`anonymous` 还要删 device ID)。Telemetry 关闭时 gstack 根本不记使用 analytics;那里不会写新东西。
- **去哪儿:** [Supabase](https://supabase.com)(Firebase 的开源替代)。schema 在 [`supabase/migrations/`](supabase/migrations/),所以你可以验证收了什么。repo 里的 publishable key 是公开 key;行级安全拒绝对直接访问,edge function 强制 schema 检查、事件 allowlist 与字段长度限制。

**Update check(默认开,无论 telemetry)。** 每个交互式技能启动跑 `gstack-update-check`。缓存过期时(60 分钟前 up-to-date 结果,升级 pending 时 12 小时,失败后 10 分钟)它向 GitHub 要最新版本:对 `github.com/garrytan/gstack` 跑 `git ls-remote` 与对 `raw.githubusercontent.com` 跑 VERSION 文件的 GET。GitHub 看到你的 IP;不会发任何关于你或你的代码。Telemetry 开着时,同样的检查也把你的 gstack 版本与 OS post 到 gstack 的 Supabase。用 `gstack-config set update_check false` 关掉。

**Outside reviews(默认开;第一次打印通知)。** Plan reviews、`/autoplan`、`/review` 和 `/ship` 对抗 pass、`/ship` 的设计 check、`/design-review`、`/design-consultation`、`/office-hours`、`/document-release` 与 `/spec` 都可以请第二个模型做独立审查:Claude Code 下的 Codex,其它 harness 下的 Claude Code。这把审查 prompt 和相关代码、diff 或 plan 发到那个 provider,用你自己的 Codex 或 Claude Code 登录。机器上第一次发时说明是哪个 provider 哪个账号。用 `gstack-config set codex_reviews disabled` 关掉;然后发东西的二进制拒绝(`not_run`、`consent_missing`),什么都不起,不可读的 config 算 disabled。`/codex` 和 `/claude-code` 在你点名要它们时仍然跑。

**Local state that is not telemetry.** 不管 telemetry 怎么设,gstack 保留几个本地文件。它们是功能 state,不是使用日志,且永不发送:

- **Session markers:** `~/.gstack/sessions/<pid>`,每个 live agent 会话一个空文件,在技能启动时被 touch,120 分钟后删除。计数让技能注意到一台机器上有多个会话;telemetry 开时,该数字就是上面的会话数。
- **Project timeline:** `~/.gstack/projects/<repo>/timeline.jsonl` 记技能启动与结束、branch 名,给 `/context-restore` 和 `/retro`。只在 `artifacts_sync_mode full` 时离开机器([docs/gbrain-sync.md](docs/gbrain-sync.md))。

**Every off-machine send is receipted.** 任何 gstack 发起的网络发送,含 telemetry,在发送前都向 `~/.gstack/security/egress.jsonl` 写一条 hash 链式、防篡改的收据;敏感 sink 如果收据写不出来直接拒发。用 `gstack-egress list` 审计,`gstack-egress verify` 验证 chain(篡改时退出 3),`gstack-egress grants` 看长期同意设置。账本记尝试发送,所以事故可审;它是审计轨迹,不是网络防火墙。

**Optional third-party bridges are off by default and receipted too.** 现存的 [Memorable recall bridge](docs/memorable-workflow-memory.md) 只在你跑 `gstack-memorable enable` 后才把 prompt 交给本地装的 vendor 二进制;每次移交先写收据,同意出现在 `gstack-egress grants`,vendor 之后发什么是它自己说的。

## Troubleshooting

**Not sure what's wrong?** 跑 doctor:`~/.claude/skills/gstack/bin/gstack-doctor`(在其它 host 上,在 gstack checkout 里 `./setup --status` 以 doctor 的绝对路径结束)。不开技能不花一分钱,它打印一行/检查(install、state root、Bun、hooks、Codex 与它的缓存模型 probe、artifacts sync、browse bundle、其它编译过的二进制、/cso 原生 helper、Claude Code、最大的 session journal 和最近的 /autoplan guard code),每个 `ok`、`warn`、`not configured` 或 `fail` 加上修复命令。仅在 `fail` 时退出非零。`--live` 还跑付费 Codex 模型检查(一次短调用)。把它的输出贴进 bug report。

**消息说 `not run`、`unavailable` 或点了某个修复?** 在 [docs/troubleshooting.md](docs/troubleshooting.md) 查:gstack 印的每条 gate 消息、它的意思、留下了什么以及修复它的命令。

**Skill 没出现?** 从 gstack checkout 跑 `./setup --status`。它打印每个安装(host、scope、version、skills directory、source checkout),对于 `stale`、`missing` 或 `unregistered` 行,点出修复它的精确命令。常用修法是从那行的 source 对那个 host 重跑 setup,例如 `cd ~/.claude/skills/gstack && ./setup`(Claude)或 `cd ~/gstack && ./setup --host codex`。项目安装活在项目的 `.claude/skills/gstack` 或 `.agents/skills/gstack`;在项目里跑它的 `setup`。

**Cursor 加载时卡住,或 agent 的 gstack 技能在升级后不见了?** 每个 checkout 现在只渲染从它装出去的 agent 的技能(#1694)。`./setup --status` 列它们;`./setup --host <name>` 加回一个。被剪枝的副本在 `~/.gstack/backups/host-renders/<time>-<id>/`,每条路径在它的 `prune.log` 里([troubleshooting](docs/troubleshooting.md#host-renders-pruned))。

**`/browse`(或 `/qa`、`/design-review`)说 `NEEDS_ASIDE` 或 `ASIDE_NOT_RUNNING`?** 那是 probe 告诉你它要用 fallback browser。想要 Aside?打开 app 并登录 — `aside --version` 应该打印版本,`aside repl 'console.log("ok")'` 应该打印 `ok` — 然后重跑。gstack 从不为你装它。Aside 开着但故意想用 fallback?`GSTACK_SKIP_ASIDE=1` 让每个技能、renderer 和 `./setup` 把 Aside 当作不在。Aside 不在时 probe 打印 `NEEDS_ASIDE: <OS>`,技能信任那行来定 macOS-only 下载提示;`GSTACK_PLATFORM` 覆盖它指的 OS,用于测试和奇怪 host(在你的 shell 里设 — gstack 从不从项目 `.env` 读)。

**`/browse` 在 fallback browser 上失败?** `cd ~/.claude/skills/gstack && bun install && bun run build`

**`/make-pdf` 或 `/diagram` 渲染不出?** 同样的两条路径:Aside 开时它们通过 Aside 打印(`bun run ~/.claude/skills/gstack/bin/gstack-render.ts some.html --screenshot /tmp/out.png` 直接测,首行 `ENGINE=aside` 或 `ENGINE=browse` 指明实际渲染的浏览器);没有就用 bundled browser,所以 `bun run build` 是修复。

**过期安装?** 跑 `/gstack-upgrade` — 或在 `~/.gstack/config.yaml` 里设 `auto_upgrade: true`。在 Codex 和其它非 Claude host 上,从终端升级:在 `./setup --status` 的 `source` 列里找 checkout,然后 `cd <source> && git pull && ./setup --host <host>`,开新会话。

**在 fallback browser 上往特定字段打字?** `browse type --selector '<css>' <text>` 往那个元素打;裸的 `browse type <text>` 往焦点所在打。

**State 位置不对,或某个设不上的设置?** `~/.claude/skills/gstack/bin/gstack-paths --explain` 显示 gstack 用哪个目录做 state 以及为什么。见 [docs/state-root.md](docs/state-root.md)。

**想要短命令?** `cd ~/.claude/skills/gstack && ./setup --no-prefix` — 从 `/gstack-qa` 切换到 `/qa`。你的选择跨升级记住。

**不用某些技能?** `gstack-config set disabled_skills make-pdf,pair-agent` 停止每个 host 注册它们,所以它们的 description 停止加载进每个会话。Claude Code 立即更新;其它 host 在下次 `./setup --host <name>` 或 `/gstack-upgrade`。它们的文件保持装着,所以调用被 disabled 技能的技能仍然能用(会有警告)。打错时拒绝并给最接近的技能名,`gstack-upgrade` 不能被 disabled,`gstack-config set disabled_skills ""` 重新启用全部。

**想要命名空间命令?** `cd ~/.claude/skills/gstack && ./setup --prefix` — 从 `/qa` 切换到 `/gstack-qa`。如果你并行用其它技能包很有用。

**Codex 说 "Skipped loading skill(s) due to invalid SKILL.md"?** 你的 Codex 技能 description 是过期的。`${CODEX_HOME:-~/.codex}/skills/gstack` 是运行时目录,不是 checkout:`./setup --status` 显示 Codex 行的 source checkout。修法: `cd <that source> && git pull && ./setup --host codex` — 对于 repo-local 安装,在项目内跑。

**Windows users:** gstack 在 Windows 11 上通过 Git Bash 或 WSL 工作。Aside 仅 macOS,所以 Windows(以及 Linux)上 browser 技能、`/make-pdf` 和 `/diagram` 总是用 gstack 自带浏览器。除 Bun 外还需 Node.js — Bun 在 Windows 上有 Playwright pipe transport 的已知 bug([bun#4253](https://github.com/oven-sh/bun/issues/4253))。browse server 自动回退到 Node.js。确保 `bun` 和 `node` 都在 PATH 上。原生 `/cso` 额外需要 PowerShell(优先 PowerShell 7 `pwsh`;Windows PowerShell 5.1 作为回退)和带 C++ 桌面开发负载的 Visual Studio 2022 Build Tools;缺失时 setup 让该技能显式 unavailable。/cso 是可选的:如果它的原生 helper 构建或发布失败,setup 仍然完成,说哪一步失败带 log 路径和重试命令,并保留更早的 helper(若它有)(`GSTACK_STRICT_BUILD=1` 让失败 fatal,CI 也这么做)。

**Known issue: Windows Smart App Control**([#2595](https://github.com/garrytan/gstack/issues/2595))。gstack 编译过的二进制(`browse`、`find-browse`、`design`、`pdf`、`gstack-global-discover`)在本机构建且未签名,所以开了 Smart App Control 的 Windows 11 拒绝启动它们;Git Bash 显示 `Permission denied`。setup 检测这点,指出被阻断的二进制和需要它们的技能,`gstack-doctor` 把它们报为 `blocked`。目前的 workaround 是在 WSL 里跑 gstack,或关掉 Smart App Control。细节:[troubleshooting](docs/troubleshooting.md#windows-smart-app-control)。sidebar 终端在 Windows 上用 Consolas,所以文本不再分散渲染。

从 PowerShell,`.\setup.ps1`(同 `./setup` 的参数)检查 Git for Windows、Bun 与 Node.js 在 PATH 上,对缺失的打印 `winget` 命令,否则在 Git Bash 里跑 `./setup`。

在 Windows 没开发者模式(MSYS2 / Git Bash)时,`setup` 回退到文件拷贝而不是符号链接,因为 `ln -snf` 产生 `git pull` 不刷新的 frozen copy。**每次 `git pull` 后重跑 `cd ~/.claude/skills/gstack && ./setup`** 让你的技能文件匹配 repo。`setup` 打印一行提醒。Unix 和 WSL 保持符号链接不需要重跑。

**Chromium 安装失败或在 `./setup` 时挂?** bundled browser 是尽力而为:setup 记原因,完成注册每个技能,打印哪些技能受影响(`/qa`、`/qa-only`、`/design-review`、`/browse`、make-pdf、`/diagram`、`/pair-agent`)。Aside 开时,浏览器技能继续在 Aside 里跑,只有 fallback 引擎缺失;`/pair-agent` 总需要 bundled browser。修原因并重跑 `./setup`。开关:`GSTACK_PLAYWRIGHT_INSTALL_TIMEOUT=<seconds>` 在慢链接上提高下载上限(默认 600);`GSTACK_SKIP_PLAYWRIGHT=1` 完全跳过 Chromium 安装(CI、无浏览器 box);`GSTACK_CHROMIUM_NO_SANDBOX=1` 在 Chromium 装了但因宿主阻断 unprivileged user namespace 而不能启动(Ubuntu 24.04+ AppArmor default,#2157)时修。

**Setup 结束 "Not registered (a skill you own already uses the name; left untouched)"?** gstack 只删除或 link 覆盖它能证明自己创建的 skill 条目:一个 symlink 到 gstack,一个带 `.gstack-owned` 标记的目录(`./setup` 写到每个它创建的目录),或一个 byte-identical 于 gstack 的或带生成 `<!-- AUTO-GENERATED from ... -->` 横幅的 SKILL.md。你自己写的 `qa/` 或 `ship/` 被 `./setup`、`gstack-relink` 和两种 prefix-mode 翻转留着不动,linker 在最后总结里点出它。改个名或移走你的,或切 mode(`./setup --prefix` / `--no-prefix`)让名字不再撞。如果你从一个生成的 gstack SKILL.md 起手自己的 skill 然后改了它,那个文件在 gstack 自己的被 link 进之前移到 `~/.gstack/backups/skills/<timestamp>/<skill>/SKILL.md`,绝不删。

**Claude 说看不到技能?** 确保项目 `CLAUDE.md` 有 gstack 段。加上:

```
## gstack
Use /browse from gstack for all web browsing. Never use mcp__claude-in-chrome__* tools.
Available skills: /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review,
/design-consultation, /design-shotgun, /design-html, /review, /deslop-shared-libs, /test-audit, /ship, /eval-plan, /land-and-deploy,
/canary, /benchmark, /browse, /open-gstack-browser, /qa, /qa-only, /design-review, /scrape,
/setup-browser-cookies, /setup-deploy, /setup-gbrain, /sync-gbrain, /retro, /investigate,
/document-release, /document-generate, /codex, /cso, /autoplan, /pair-agent, /careful, /freeze,
/guard, /unfreeze, /gstack-upgrade, /learn.
```

## License

MIT。永远免费。去造东西吧。
