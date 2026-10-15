# 项目结构

逐字从 CLAUDE.md 移出(减少 token 加载)。

## 目录树

```
gstack/
├── browse/          # /browse skill + gstack 自己的无头浏览器引擎 (Playwright) — Aside 不存在时的回退
│   ├── SKILL.md.tmpl  # /browse: Aside 优先 ({{ASIDE_SETUP}} + cookbook), $B 回退
│   ├── src/         # CLI + server + commands
│   │   ├── commands.ts  # 命令注册表(单一事实来源)
│   │   ├── routes/      # 守护进程 HTTP 路由表(table.ts: 条目形状、auth 类型、denials、dispatch) + 每个区域的处理程序
│   │   └── snapshot.ts  # SNAPSHOT_FLAGS 元数据数组
│   ├── test/        # 集成测试 + fixtures
│   └── dist/        # 编译的二进制文件
├── hosts/           # 类型化主机配置(每个 AI agent 一个)
│   ├── claude.ts    # 主要主机配置
│   ├── claude/hooks/  # Claude Code 生命周期 hooks(AUQ capture + enforcement、spawned-session directive、timeline stop、Memorable recall bridge(opt-in));hook-log.ts 是它们的一个错误日志写入器
│   ├── codex.ts, factory.ts, kiro.ts  # 现有主机
│   ├── opencode.ts, slate.ts, cursor.ts, openclaw.ts  # IDE 主机
│   ├── hermes.ts, gbrain.ts  # Agent runtime 主机
│   └── index.ts     # 注册表:导出所有,派生 Host 类型
├── scripts/         # 构建 + DX 工具
│   ├── gen-skill-docs.ts  # 模板 → SKILL.md 生成器(配置驱动)
│   ├── gen-agents-digest.ts  # 生成预算上限的指令层摘要(agents-digest/)
│   ├── host-config.ts     # HostConfig 接口 + 验证器
│   ├── host-config-export.ts  # setup 脚本的 shell 桥接
│   ├── resolvers/   # 模板解析器模块(preamble, aside = Aside 驱动合约 + research, browse = $B 回退设置 + 命令参考, qa = 表面感知的 QA/exploration, sections = 延迟加载, design, design-checklist = 从 lib/design-catalog.ts 渲染 review/design-checklist.md, review-dashboard / plan-gates / spec-review / review-scope, outside-voice = 外部声音原语 + 一个失败策略, outside-voice-steps = 第二意见 / 对抗 / 计划和文档评审, gbrain 等)
│   ├── lib/shard-engine.ts  # 两个测试通道的共享分片引擎:spawn、group kill、严格 verdict、按分片沙箱、日志、持续时间 seed、flags
│   ├── skill-check.ts     # 健康仪表板
│   ├── test-free-shards.ts  # 分片引擎上的免费通道策略(GSTACK_FREE_JOBS、可选 flaky retry)
│   ├── test-paid-shards.ts  # 分片引擎上的付费通道策略(每个分片一个 Bun 进程)
│   ├── test-strict-output.ts  # lib/shard-engine.ts 的兼容性重新导出
│   ├── eval-flake-rank.ts  # Flake-telemetry dial:根据 eval 运行 + 免费通道分类账的 retry passes 对测试进行排名
│   ├── sandbox-doctor.sh  # 一条命令 cloud-sandbox fixer:使免费套件变绿
│   └── dev-skill.ts       # 监视模式
├── lib/state-root.ts  # State-root owner(resolveStateRoot、readConfigKey);bin/gstack-state-root.sh 是其 sourced bash 双胞胎;docs/state-root.md
├── test/            # 技能验证 + eval 测试
│   ├── helpers/     # skill-parser.ts、session-runner.ts、llm-judge.ts、eval-store.ts、aside-available.ts(Aside self-skip 探针);pty/ = PTY harness(session.ts 拥有 runner 循环,fake-session.ts 是脚本化测试驱动程序),通过 claude-pty-runner.ts barrel 导入
│   ├── fixtures/    # 基本事实 JSON、植入的 bug fixtures、eval baselines、impeccable 引擎捕获(impeccable-*.json、转储的 slop 页面、fake-impeccable.ts shim)
│   ├── aside-driver.test.ts      # 第 1 层:固定 {{ASIDE_SETUP}} 合约句子 + 回退交接
│   ├── aside-render.test.ts      # 第 1 层 pins + 两个引擎上的假可执行文件运行 + 实时 Aside 渲染(没有 Aside 时自跳)
│   ├── gstack-render-cli.test.ts # 第 1 层:bin/gstack-render.ts argv guards + 与假守护进程的输出合约
│   ├── skill-validation.test.ts  # 第 1 层:静态验证(免费,<1s)
│   ├── gen-skill-docs.test.ts    # 第 1 层:生成器质量(免费,<1s)
│   ├── setup-*.test.ts, relink.test.ts, hook-scripts.test.ts  # 第 1 层:setup 链接器所有权、退役技能修剪、浏览器提示、重新构建检查 + Chromium 引导(从 setup 锚定切片)、gstack-relink、PreToolUse hooks(免费)
│   ├── skill-llm-eval.test.ts   # 第 3 层:LLM-as-judge(每次运行 ~$0.15)
│   └── skill-e2e-*.test.ts       # 第 2 层:通过 claude -p 的 E2E(每次运行 ~$3.85,按类别拆分)
├── qa/, qa-only/    # 表面感知浏览器/功能 QA;/qa-only 报告并提出测试,无需产品编辑
├── plan-design-review/  # /plan-design-review skill(仅报告的设计审计)
├── design-review/    # /design-review skill(设计审计 + 修复循环)
├── ship/            # Ship 工作流 skill
├── review/          # PR review skill(checklist.md 是手写的;design-checklist.md 从 lib/design-catalog.ts 生成)
├── deslop-shared-libs/ # 仅建议的审计,用于有价值的共享代码提取
├── test-audit/      # 报告优先扫描低价值测试(测试值栏,审计模式)
├── eval-plan/       # 预注册的付费评估:prereg、arms、$0 试运行、定价试点(bin/gstack-eval-plan)
├── plan-ceo-review/ # /plan-ceo-review skill
├── plan-eng-review/ # /plan-eng-review skill
├── autoplan/        # /autoplan skill(自动评审流水线:CEO → design → DX → eng,eng 始终最后)
├── benchmark/       # /benchmark skill(性能回归检测)
├── canary/          # /canary skill(部署后监控循环)
├── codex/           # /codex skill(通过 OpenAI Codex CLI 进行多 AI 第二意见)
├── land-and-deploy/ # /land-and-deploy skill(merge → deploy → canary verify)
├── office-hours/    # /office-hours skill(YC Office Hours — 启动诊断 + 构建者头脑风暴)
├── investigate/     # /investigate skill(系统化的根本原因调试)
├── spec/            # /spec skill(五阶段规范 → GitHub issue,可选代理 spawn,/ship 自动关闭)
├── retro/           # 回顾 skill(包括 /retro global 跨项目模式)
├── careful/         # /careful skill;bin/check-careful.sh(PreToolUse 破坏性命令 hook) + bin/hook-extract.sh(共享 hook helpers:payload extraction、deny JSON、gstack_hook_state_root)
├── freeze/          # /freeze skill;bin/check-freeze.sh(PreToolUse 编辑边界 hook;源 careful/bin/hook-extract.sh,失败关闭)
├── guard/, unfreeze/  # /guard(careful + freeze 一体化),/unfreeze
├── gstack-upgrade/  # /gstack-upgrade skill + migrations/(在升级期间的 ./setup 之后运行)
├── bin/             # CLI 实用程序(gstack-render.ts = 通过 Aside 或引擎渲染本地 HTML 文件,gstack-design-detect.ts = 通过用户安装的 impeccable 引擎进行探针/扫描;gstack-design-md.ts = 打开 DESIGN.md check/convert/tokens/mark;gstack-repo-mode,gstack-slug,gstack-config,gstack-wtree,gstack-evidence,gstack-issue-guard,gstack-relink,gstack-memorable 等)
├── document-release/ # /document-release skill(每次发布前的预验证审计;独立文档更新 + Diataxis 覆盖映射)
├── document-generate/ # /document-generate skill(Diataxis 文档生成器:tutorial/how-to/reference/explanation)
├── cso/             # /cso skill(OWASP Top 10 + STRIDE 安全审计)
├── design-consultation/ # /design-consultation skill(从头开始设计系统)
├── design-shotgun/  # /design-shotgun skill(视觉设计探索)
├── open-gstack-browser/  # /open-gstack-browser skill(启动 GStack Browser)
├── connect-chrome/  # symlink → open-gstack-browser(向后兼容)
├── setup-browser-cookies/, pair-agent/, skillify/  # 回退引擎技能(cookie 导入、共享浏览器隧道、编码 /scrape)
├── scrape/          # 浏览器数据提取(使用 design-review/, canary/, benchmark/);Aside 优先,$B 回退
├── make-pdf/        # /make-pdf skill + 编译的 `pdf` 二进制文件(嵌入 lib/aside-render.ts);test/ = 单元测试(cli-exit-codes、setup-smoke、render) + 在任何解析的引擎上的 e2e/*-gate.test.ts
├── diagram/         # /diagram skill(mermaid → SVG/PNG/.excalidraw 通过 bin/gstack-render.ts + lib/diagram-render)
├── design/          # 设计二进制 CLI(GPT Image API)
│   ├── src/         # CLI + commands(generate、variants、compare、serve 等)
│   ├── test/        # 集成测试
│   └── dist/        # 编译的二进制文件
├── agents-digest/   # 提交的 2KB 指令层规则摘要(gstack-AGENTS.md),用于读取规则的主机
├── extension/       # Chrome 扩展(侧面板 + 活动提要 + CSS 检查器)
├── lib/             # 共享库(aside-render.ts = 本地 HTML 渲染,Aside 优先,引擎回退;design-catalog.ts = 每个设计技能从中渲染的类型化设计反模式目录;design-detect-contract.ts = 检测器 sentinel 词汇;design-md.ts = 打开 DESIGN.md reader/writer;dom-dump-script.ts + 生成的 dom-dump.js = 用于检测器的渲染 DOM 转储;review-evidence.ts = 评论开始收据绑定、计算新鲜度和共享代码快照资格;frontend-scope.ts;claude-bin.ts、error-handling.ts、worktree.ts、egress-receipt.ts、context-bill.ts、redact-engine.ts、tracker-guard.ts、version-source.ts、code-intelligence/)
│   └── diagram-render/  # Vendored mermaid + excalidraw 运行时,内置到渲染器加载的一个离线 bundle
├── patches/         # bun `patchedDependencies` 补丁(playwright-core windowsHide)
├── docs/designs/    # 设计文档(包括 IMPECCABLE_INTEROP.md = 设计检测器 / 目录 / 打开 DESIGN.md 记录,以及 fork-port-residual-2026-09/ 评估证据)
├── setup-deploy/    # /setup-deploy skill(一次性部署配置)
├── .github/         # CI 工作流 + 共享复合操作(.github/actions/) + Docker 镜像(claude CLI 已固定)
│   ├── workflows/   # evals.yml(Ubicloud 上的 E2E)、quality-gate.yml(秘密扫描)、dependency-review.yml、osv-scanner.yml、skill-docs.yml、actionlint.yml,以及 free-tests、Windows、periodic 和 marathon evals、CSO release、平台资格和较小的 gates
│   └── docker/      # Dockerfile.ci(预烘焙工具链 + Playwright/Chromium)
├── contrib/         # 仅贡献者工具(永远不会为用户安装)
│   └── add-host/    # /gstack-contrib-add-host skill
├── setup            # 一次性设置:构建 browse、design + make-pdf 二进制文件(在缺少任何文件或 browse/make-pdf/design/lib 源更改时重新构建)+ 尽力而为的 Chromium 引导 + 链接技能 + 修剪退役技能渲染(都是所有权 gated)
├── SKILL.md         # 从 SKILL.md.tmpl 生成(不要直接编辑)
├── SKILL.md.tmpl    # 模板:编辑此文件,运行 gen:skill-docs
├── ETHOS.md         # 构建者哲学(煮沸海洋、构建前搜索)
├── NOTICE.md        # 第三方通知:从 impeccable 和 DESIGN.md 规范派生的材料(都是 Apache-2.0)
├── licenses/        # 上述通知的逐字许可文本(Apache-2.0.txt)
└── package.json     # browse 的构建脚本
```
