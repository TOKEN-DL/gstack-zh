# Skill 深度解析

每一个 gstack skill 的详细指南:理念、工作流和示例。

| Skill | 你的专家 | 他们做什么 |
|-------|----------------|--------------|
| [`/office-hours`](#office-hours) | **YC Office Hours(创业导师答疑)** | 从这里开始。六个强制性问题,在你写代码前重新框定你的产品。挑战你的框架、质疑前提、生成可替代的实现方案。设计文档(Design Doc)会输入到所有下游 skill。 |
| [`/spec`](#spec) | **Spec Author(规格作者)** | 用五个阶段把模糊的意图变成精确、可执行的规格。产出 backlog-ready(待办列表就绪)的输出,供下游 skill 选用。可选择在末尾 spawn(派生)智能体。 |
| [`/plan-ceo-review`](#plan-ceo-review) | **CEO / Founder(创始人)** | 重新思考问题。找到隐藏在请求里的 10 星产品。四种模式:扩展(Expansion)、选择性扩展(Selective Expansion)、保持范围(Hold Scope)、缩减(Reduction)。 |
| [`/plan-eng-review`](#plan-eng-review) | **Eng Manager(工程经理)** | 锁定架构、数据流、图示、边界情况和测试。把隐藏假设公开化。 |
| [`/plan-design-review`](#plan-design-review) | **Senior Designer(资深设计师)** | 交互式的 plan-mode 设计评审。每个维度 0-10 评分,说明 10 分长什么样,修复计划。在 plan mode 下工作。 |
| [`/design-consultation`](#design-consultation) | **Design Partner(设计伙伴)** | 从零开始构建完整的设计系统。了解行业格局,提出有创意的风险,生成真实的产品模型。设计处于所有其他阶段的核心。 |
| [`/review`](#review) | **Staff Engineer(资深工程师)** | 找出通过 CI 但在生产环境爆掉的 bug。自动修复明显的问题。标记完整性缺口。咨询性的简化视角会标记过度构建的代码(从不阻塞,从不自动应用)。 |
| [`/investigate`](#investigate) | **Debugger(调试员)** | 系统化的根因调试。铁律:未经调查不修复。追踪数据流,验证假设,3 次修复失败后停止。 |
| [`/design-review`](#design-review) | **Designer Who Codes(会写代码的设计师)** | 实时站点的视觉审计 + 修复循环。80 项审计,然后修复发现的问题。原子提交(atomic commits),修复前/后截图。 |
| [`/design-shotgun](#design-shotgun) | **Design Explorer(设计探索者)** | 生成多个 AI 设计变体,在你的浏览器中打开对比看板,迭代直到你确认方向。品味记忆会偏向你的偏好。 |
| [`/design-html`](#design-html) | **Design Engineer(设计工程师)** | 生成生产级质量的 Pretext-native HTML。可与已批准的模型、CEO 计划、设计评审或从零开始一起工作。文本随尺寸重新流动,高度随内容调整。按设计类型智能路由 API。React/Svelte/Vue 框架检测。预览通过你的 Aside 浏览器渲染。 |
| [`/qa`](#qa) | **QA Lead(质量保证主管)** | 探索浏览器和功能行为(APIs、CLIs、jobs、workers、webhooks),复现缺陷,在修复前证明回归失败,然后修复并重新验证。 |
| [`/qa-only`](#qa-only) | **QA Reporter(质量保证报告员)** | 探索同样的表面并用证据提出回归用例,不动产品代码或测试。 |
| [`/scrape`](#browse) | **Browser Data Extractor(浏览器数据提取器)** | 从网页中提取结构化数据(表格、列表、价格),使用你的 Aside 浏览器的真实登录状态。与 `/browse` 使用相同的驱动契约。在回退浏览器上,编码化的浏览器技能以约 200ms 响应重复意图。 |
| [`/skillify`](#browse) | **Skill Codifier(技能编码器)** | 回退浏览器技能:回溯你的对话,找到最近的 `/scrape` 原型,合成脚本 + 测试 + fixture,运行测试,提交前询问。在 Aside 上,持久化的每站点自动化属于 Aside 自己的技能。 |
| [`/ship`](#ship) | **Release Engineer(发布工程师)** | 同步主分支(main),运行测试,在限定范围内探索变更行为,在最终验证前审计覆盖率和文档,然后推送并打开或更新一个 PR。适当时引导测试框架。 |
| [`/land-and-deploy`](#land-and-deploy) | **Release Engineer(发布工程师)** | 合并 PR,等待 CI 和部署,验证生产健康度。从"已批准"到"在生产中已验证"一条命令搞定。 |
| [`/canary`](#canary) | **SRE(站点可靠性工程师)** | 部署后的监控循环。在你的 Aside 浏览器中观察控制台错误、性能回归和页面失败。 |
| [`/benchmark`](#benchmark) | **Performance Engineer(性能工程师)** | 测量页面加载时间、Core Web Vitals 和资源大小。在每个 PR 上对比修复前/后。随时间跟踪趋势。 |
| [`/cso`](#cso) | **Chief Security Officer(首席安全官)** | 有明确覆盖范围的安全发现支持。静态评估无需目录配置文件即可使用;受控的运行时/扫描器执行需要匹配的合格配置文件。运行时测试的 bundle 单独认证外部断言。项目测试完成仍为 `self_reported`,因为目标代码控制测试流程;`tested` 保留给未来的目标独立完成见证。 |
| [`/document-release`](#document-release) | **Technical Writer(技术作者)** | 每次发布前审计相关文档;独立运行也可以在 PR 存在后更新文档。捕获过时的 README 并报告未解决的缺口。 |
| [`/document-generate`](#document-generate) | **Technical Writer(技术作者)** | 从代码生成 Diataxis 文档(tutorial / how-to / reference / explanation)。 |
| [`/retro`](#retro) | **Eng Manager(工程经理)** | 团队感知的每周回顾。按人拆分、发布连胜、测试健康趋势、成长机会。 |
| [`/browse`](#browse) | **QA Engineer(质量保证工程师)** | 给智能体一双眼睛。首先驱动你的 Aside 浏览器(真实会话、真实点击、真实截图),通过确定性的 `aside repl` 脚本;当 Aside 不在时回退到 gstack 自己的 Chromium(每个命令约 100ms)。 |
| [`/setup-browser-cookies`](#setup-browser-cookies) | **Session Manager(会话管理器)** | 从 Chrome、Chromium、Brave、Edge、仅限 Windows 的 Opera 和 Opera GX,或仅限 macOS 的 Comet、Arc、Dia 复制选定的 cookie 到回退浏览器。选择你的 profile(配置文件)和域名;单独检查登录状态。在 Aside 上不需要,因为它已经有你的会话。 |
| [`/autoplan`](#autoplan) | **Review Pipeline(评审流水线)** | 一条命令,完整评审的计划。自动运行 CEO → design → DX → eng 评审(eng 总是最后,所以发布门控评审最终修订后的计划),并编码决策原则。仅呈现需要你批准的品味决策。 |
| [`/plan-devex-review`](#plan-devex-review) | **DX Reviewer(开发者体验评审员)** | Plan 阶段的 DX 评审。TTHW(time-to-hello-world,到 Hello World 的时间)、神奇时刻、摩擦点、persona 痕迹。三种模式:扩展、打磨、分类。 |
| [`/devex-review`](#devex-review) | **DX Reviewer (live)(实时开发者体验评审员)** | 实时开发者体验审计。走查真实的引导流程,测量 TTHW,捕获文档中的不实之词。 |
| [`/plan-tune`](#plan-tune) | **Question Tuner(问题调优器)** | 按问题自调 AskUserQuestion 灵敏度。标记问题为 never-ask(从不询问)、always-ask(始终询问)或 only-for-one-way(仅单向询问时询问)。 |
| [`/spec`](#spec) | **Spec Author(规格作者)** | 用五个阶段把模糊的意图变成精确、可执行的规格。提交 GitHub issue,在新的 worktree(工作树)中可选 spawn Claude Code 智能体,并让 `/ship` 在合并时关闭源 issue。 |
| [`/learn`](#learn) | **Memory(记忆)** | 管理 gstack 在跨会话中学到的东西。审阅、搜索、修剪并导出项目特定的模式和偏好。 |
| [`/context-save`](#context-save) | **Save State(保存状态)** | 保存工作上下文(git 状态、决策、剩余工作),以便任何未来的会话可以恢复。 |
| [`/context-restore`](#context-restore) | **Restore State(恢复状态)** | 从保存的上下文恢复,甚至跨 Conductor workspace 切换。 |
| [`/health`](#health) | **Code Quality Dashboard(代码质量仪表板)** | 包装类型检查器、linter、测试、死代码检测。计算加权 0-10 分数;随时间跟踪趋势。 |
| [`/deslop-shared-libs`](#deslop-shared-libs) | **Shared Code Reviewer(共享代码评审员)** | 在最近的工作中寻找有价值的共享代码抽取。仅供参考。 |
| [`/test-audit`](#test-audit) | **Test Auditor(测试审计员)** | 扫描现有测试中的低价值、实现耦合或重复测试。仅报告,除非你批准一批。 |
| [`/landing-report`](#landing-report) | **Ship Queue Dashboard(发布队列仪表板)** | 工作区感知的发布队列的只读快照。哪些版本槽位被认领,哪些同级工作区有 WIP(进行中工作)。 |
| [`/benchmark-models`](#benchmark-models) | **Model Benchmark(模型基准)** | 跨模型的并排基准测试 skill(Claude vs GPT vs Gemini)。延迟、token、成本、可选的 LLM 评判质量。 |
| [`/eval-plan`](#eval-plan) | **Eval Planner(评估规划器)** | 预注册付费评估(bars、arms、decoys、停止规则),用 stub 模型进行 $0 试运行,试点三个已定价项,然后从推断表中批准支出。 |
| | | |
| **Multi-AI** | | |
| [`/codex`](#codex) | **Second Opinion(第二意见)** | OpenAI Codex 评审、挑战和咨询。可在 Codex 沙箱外使用。 |
| [`/claude-code`](#claude-code) | **Second Opinion(第二意见)** | Claude Code 评审、挑战和咨询。可在 Claude Code 沙箱外使用;用于 Codex 中的自动外部评审。 |
| [`/pair-agent`](#browse) | **Remote Agent Bridge(远程智能体桥接)** | 将远程 AI 智能体(OpenClaw、Codex、Cursor、Hermes)与 gstack 自己的浏览器配对。范围化的隧道、锁定的允许列表、会话 token。回退浏览器技能;驱动 Aside 的智能体打开自己的标签页。 |
| [`/setup-gbrain`](#setup-gbrain) | **Memory Sync(记忆同步)** | 设置 gbrain 以实现跨机器会话记忆同步。一条命令从零到上线。 |
| [`/sync-gbrain`](#sync-gbrain) | **Keep Brain Current(保持大脑最新)** | 根据此仓库的代码刷新 gbrain;教会智能体何时使用 `gbrain search`/`code-def` 而不是 Grep。幂等;可安全地重新运行。 |
| | | |
| **Safety & Utility** | | |
| [`/careful`](#safety--guardrails) | **Safety Guardrails(安全护栏)** | 在破坏性命令(rm -rf、DROP TABLE、force-push、git reset --hard)前发出警告。可覆盖任何 MEDIUM 警告;根/家目录递归删除和默认分支 force-push 被硬性拒绝。常见的构建清理已列入白名单。 |
| [`/freeze`](#safety--guardrails) | **Edit Lock(编辑锁定)** | 将所有文件编辑限制到单个目录。阻止边界外的 Edit、Write 和 NotebookEdit。调试时防止意外。 |
| [`/guard`](#safety--guardrails) | **Full Safety(全面安全)** | 一条命令结合 /careful + /freeze。生产工作的最大安全性。 |
| [`/unfreeze`](#safety--guardrails) | **Unlock(解锁)** | 移除 /freeze 边界,允许再次在所有地方编辑。 |
| [`/open-gstack-browser`](#open-gstack-browser) | **GStack Browser(GStack 浏览器)** | 有头模式启动 gstack 自己的浏览器,带侧边栏、反机器人隐身、自动模型路由、cookie 导入和 Claude Code 集成。回退引擎的可见面孔;当 Aside 打开时,你在那里观察智能体的标签页。 |
| [`/setup-deploy`](#setup-deploy) | **Deploy Configurator(部署配置器)** | `/land-and-deploy` 的一次性设置。检测你的平台、生产 URL 和部署命令。 |
| [`/gstack-upgrade`](#gstack-upgrade) | **Self-Updater(自我更新器)** | 将 gstack 升级到最新版本。检测全局安装还是 vendored(供应商式)安装,同步两者,显示更改内容。 |
| [`/make-pdf`](#make-pdf) | **PDF Generator(PDF 生成器)** | 将任何 Markdown 文件转换为出版级质量的 PDF。合适的边距、页码、封面、可点击的目录。Mermaid/excalidraw 围栏渲染为矢量图;`--to html\|docx` 用于其他格式。通过你的 Aside 浏览器(macOS 15+)打印,或者当 Aside 不在时通过 gstack 自带的浏览器。 |
| [`/diagram`](#diagram) | **Diagram Maker(图表制作器)** | 英语输入,图表输出:mermaid 源码 + 可编辑的 `.excalidraw`(在 excalidraw.com 打开,手绘风格) + 渲染的 SVG/PNG。完全离线,通过你的 Aside 浏览器(macOS 15+)渲染,或者当 Aside 不在时通过 gstack 自带的浏览器。 |
| [`/ios-qa`](#ios-qa) | **iOS QA Lead(iOS 质量保证主管)** | 通过 USB CoreDevice 隧道 + 嵌入式 StateServer 进行实时设备的 iOS QA。读取 Swift 源代码、生成访问器、驱动真实 iPhone 或 iPad。可选择通过 Tailscale 向远程智能体公开设备。 |
| [`/ios-fix`](#ios-fix) | **iOS Autonomous Fixer(iOS 自动修复器)** | 在真实 iPhone 上完成 find → fix → verify 循环。捕获重现快照,修复源代码,重建,重新部署,验证。 |
| [`/ios-design-review`](#ios-design-review) | **iOS Designer's Eye(iOS 设计师之眼)** | 在真实 iPhone 上进行 10 个维度的 Apple HIG 审计。为每个屏幕评分,说出如何使其成为 10 分。 |
| [`/ios-clean`](#ios-clean) | **iOS Bridge Cleanup(iOS 桥接清理)** | 便捷包装器,用于剥离 DebugBridge SPM + `#if DEBUG` 接线。结构化的 Release-build 保护位于 Package.swift + CI 中;此 skill 用于指导手动移除。 |
| [`/ios-sync`](#ios-sync) | **iOS Bridge Resync(iOS 桥接重新同步)** | 根据最新的上游 gstack 重新生成访问器和 Swift 模板。在添加新的 `@Observable` 类或升级 gstack 时运行。 |

---

## `/office-hours`

这是每个项目应该开始的地方。

在你规划之前,在你评审之前,在你写代码之前,坐下来与一位 YC 风格的伙伴一起思考你实际在构建什么。不是你以为你在构建什么。你*实际*在构建什么。

### 重新框定(The reframe)

这是一个真实项目中发生的事情。用户说:"我想为我的日历做一个每日简报应用。"合理的要求。然后它询问痛点,具体例子,而不是假设。他们描述了一个助手遗漏事项、多个 Google 账户间的日历条目信息陈旧、准备文档是 AI slop(低质 AI 生成内容)、事件地点错误需要很长时间才能追踪。

它回应说:*"我要质疑这个框架,因为我感觉你已经超出了它。你说'多 Google 日历管理的每日简报应用'。但你实际描述的是一个个人首席幕僚 AI。"*

然后它提取了用户没有意识到的五个能力:

1. **观察你的日历** 跨所有账户,检测过时信息、缺失位置、权限缺口
2. **生成真正的准备工作** ,不是后勤摘要,而是*智力工作*——为董事会会议、播客、筹款活动做准备
3. **管理你的 CRM** ——你要见谁,关系是什么,他们想要什么,历史如何
4. **优先排序你的时间** ——标记准备何时需要提前开始,主动预留时间,按重要性对事件排序
5. **用钱换取杠杆** ——主动寻找委派或自动化的方式

那个重新框定改变了整个项目。他们本来要构建一个日历应用。现在他们正在构建比这有价值十倍的东西——因为这个 skill 倾听了他们的痛点,而不仅仅是功能请求。

### 前提挑战

重新框定之后,它提出前提供你验证。不是"这听起来不错?"——而是关于产品的实际可证伪的声明:

1. 日历是锚定的数据源,但价值在于其上的智能层
2. 助手不会被替换,而是被超级强化
3. 最窄的楔子是一个真正有效的每日简报
4. CRM 集成是必须有的,不是锦上添花

你同意、不同意或调整。你接受的每个前提都成为设计文档中的承重点。

### 实现替代方案

然后它生成 2-3 个具体的实现方法,带有诚实的努力估计:

- **方法 A:每日简报优先** ——最窄的楔子,明天就发布,M 努力(人工:约 3 周 / CC:约 2 天)
- **方法 B:CRM 优先** ——首先构建关系图,L 努力(人工:约 6 周 / CC:约 4 天)
- **方法 C:完整愿景** ——一次完成所有事情,XL 努力(人工:约 3 个月 / CC:约 1.5 周)

推荐 A,因为你从真实使用中学习。CRM 数据在第二周自然出现。

### 两种模式

**Startup 模式(创业模式)** ——为构建业务的创始人和企业内部创业者设计。你会得到从 YC 合伙人如何评估产品提炼出的六个强制性问题:需求现实性、现状、迫切具体性、最窄楔子、观察与惊喜、未来适配。这些问题故意让人不舒服。如果你不能指出一个需要你产品的具体的人,那是在写任何代码之前要学习的最重要的事情。

**Builder 模式(构建者模式)** ——为黑客马拉松、副业、开源、学习和寻找乐趣设计。你得到一个热情的协作者,帮助你找到想法最酷的版本。什么会让某人说"哇"?最快达到可以分享的路径是什么?这些问题是生成性的,不是审讯性的。

### 设计文档(Design Doc)

两种模式都以写入 `~/.gstack/projects/` 的设计文档结束——该文档直接输入到 `/plan-ceo-review` 和 `/plan-eng-review`。完整的生命周期现在是:`office-hours → plan → implement → review → QA → ship → retro`。

设计文档批准后,`/office-hours` 会反思它注意到的你是如何思考的——不是泛泛的赞美,而是对你在会话中说的具体事情的具体回调。这些观察也出现在设计文档中,这样你稍后重读时会再次遇到它们。

---

## `/plan-ceo-review`

这是我的 **founder mode(创始人模式)**。

我希望模型在这里展现品味、雄心、用户同理心和长远的时间视野。我不希望它字面理解请求。我希望它首先问一个更重要的问题:

**这个产品实际上是做什么的?**

我将此视为 **Brian Chesky mode(Brian Chesky 模式)**。

重点不是实现显而易见的工单。重点是从用户的角度重新思考问题,找到感觉不可避免的、令人愉快的、甚至有点魔幻的版本。

### 示例

假设我正在构建一个 Craigslist 风格的列表应用,我说:

> "让卖家为他们的商品上传照片。"

一个糟糕的助手会添加一个文件选择器并保存一张图片。

那不是真正的产品。

在 `/plan-ceo-review` 中,我希望模型问"照片上传"是否真的是这个功能。也许真正的功能是帮助某人创建一个实际能卖出去的列表。

如果那是真正的工作,整个计划都会改变。

现在模型应该问:

* 我们能从照片中识别产品吗?
* 我们能推断 SKU 或型号吗?
* 我们能搜索网络并自动起草标题和描述吗?
* 我们能拉取规格、类别和价格比较吗?
* 我们能建议哪张照片作为主图转化率最高吗?
* 我们能在上传的照片丑陋、黑暗、杂乱或低信任度时检测出来吗?
* 我们能让体验感觉高级,而不是像 2007 年的死表单吗?

这就是 `/plan-ceo-review` 为我所做的。

它不只是问,"我如何添加这个功能?"
它问,**"隐藏在这个请求里的 10 星产品是什么?"**

### 四种模式

- **SCOPE EXPANSION(范围扩展)** ——大胆梦想。智能体提出雄心勃勃的版本。每个扩展都呈现为你选择加入的单独决策。热情推荐。
- **SELECTIVE EXPANSION(选择性扩展)** ——保持你当前的范围作为基线,但看看还有什么是可能的。智能体逐个呈现机会,推荐中立——你挑选值得做的。
- **HOLD SCOPE(保持范围)** ——对现有计划的最大严格性。不呈现扩展。
- **SCOPE REDUCTION(范围缩减)** ——找到最小可行版本。削减其他一切。

愿景和决策被持久化到 `~/.gstack/projects/`,所以它们在对话之外仍然存在。卓越的愿景可以被提升到仓库中的 `docs/designs/` 以供团队使用。

---

## `/plan-eng-review`

这是我的 **eng manager mode(工程经理模式)**。

一旦产品方向正确,我想要一种完全不同的智能。我不想更多蔓延的想法。我不想更多"如果...会不会很酷"。我希望模型成为我最好的技术主管。

此模式应做到:

* 架构
* 系统边界
* 数据流
* 状态转换
* 失败模式
* 边界情况
* 信任边界
* 测试覆盖

对我来说是出乎意料的大解锁:**图表(Diagrams)**。

当你强迫 LLM 绘制系统时,它们会变得更完整。序列图、状态图、组件图、数据流图,甚至测试矩阵。图表强迫隐藏的假设公开化。它们让含糊的规划变得更难。

所以 `/plan-eng-review` 是我希望模型构建能够承载产品愿景的技术骨架的地方。

### 示例

拿同样的列表应用示例。

假设 `/plan-ceo-review` 已经做了它的工作。我们决定真正的功能不仅仅是照片上传。它是一个智能的列表流程,可以:

* 上传照片
* 识别产品
* 从网络丰富列表
* 起草强标题和描述
* 建议最佳主图

现在 `/plan-eng-review` 接管。

现在我希望模型回答这样的问题:

* 上传、分类、丰富和草稿生成的架构是什么?
* 哪些步骤同步发生,哪些转到后台作业?
* 应用服务器、对象存储、视觉模型、搜索/丰富 API 和列表数据库之间的边界在哪里?
* 如果上传成功但丰富失败会怎样?
* 如果产品识别是低置信度的怎么办?
* 重试如何工作?
* 我们如何防止重复作业?
* 何时持久化什么,什么可以安全地重新计算?

这是我希望图表的地方——架构图、状态模型、数据流图、测试矩阵。图表强迫隐藏的假设公开化。它们让含糊的规划变得更难。

那就是 `/plan-eng-review`。

不是"让想法变小。"
**让想法可构建。**

调用说明:在 plan mode 中,该 skill 跳过"我应该评审什么?"范围问题,自动评审你的活跃计划,用一行宣布其选择("Scope gate: plan mode — auto-selected B"),以便你可以重定向它。明确命名目标("review PLAN.md"),你的选择在任何模式下都优先。在 plan mode 外部且没有命名目标,它先询问——那个 gate 是个硬停止。

### 评审就绪仪表板(Review Readiness Dashboard)

每次评审(CEO、Eng、Design)都记录其结果。在每次评审结束时,你会看到一个仪表板:

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Required |
```

[desktop write output truncated: original_bytes=57250; if the omitted remainder is needed, continue with read using offset=247 and the same path.]
