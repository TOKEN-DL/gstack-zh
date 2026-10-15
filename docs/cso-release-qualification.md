# 构建和限定 CSO 运行时

`bun run build:cso` 编译可信启动器、helper 和独立的 watchdog。Bun 必须支持所有四个 `--no-compile-autoload-*` 开关。macOS 和 Linux 还需要 C 编译器(`cc`、Clang 或 GCC)。Linux 静态链接启动器;macOS 使用加固运行时对其进行签名。`bun run build` 包括此步骤,并且仍然是严格的发布/开发者构建。Setup 功能探测四个 Bun 开关和主机本机工具链,然后再进行构建。如果两者都不可用,它会构建 gstack 的其余部分,删除任何陈旧的 CSO helpers,并报告 `/cso` 不可用;该 skill 然后报告 `not assessed`,永远不会替代仓库工具。安装先决条件后重新运行 setup。当探测成功时,setup 保留严格的失败行为并检查工件的新鲜度。生成的 `bin/gstack-cso-launcher` 是 setup 构建的直接命令;编译的核心不是入口点。核心的直接调用标记捕获意外的滥用;它不会验证进程对可信的同用户主机。原生启动器是剥离 Bun、Node 和 loader 注入变量的预运行时边界。CSO 不作为 npm package bin 公开:其原生启动器、编译核心、watchdog 和隐藏的 `.gstack-cso-generation` 清单必须为当前主机一起构建和安装。Windows 还需要来自同一发布的空 `.gstack-cso-generation.lock` 文件。Windows 使用 Visual Studio 2022 Build Tools(使用 C++ 进行桌面开发)构建原生启动器,通过 Git Bash 从已安装的 Developer PowerShell 脚本中发现;仅使用 Bun 编译核心。专用的 Windows CI 作业分别检查启动、参数转发和运行时注入,而非静态审计和私有状态支持。POSIX watchdog 执行在那里仍然不可用。

尚不存在合格的运行时或扫描器映像,因此包含的运行时和扫描器执行不可用,并且 setup 不会预加载:扫描器目录在其构建输入等待审查时为空,而运行时目录的配置文件在仍正在构建的私有评估器可以限定它们之前保持不合格。

每次成功的新安装或升级都要求可信启动器预加载当前主机平台的合格运行时和扫描器映像。这是唯一的自动映像获取路径。它首先验证提交的目录,固定本地 Unix 套接字 Docker endpoint,拒绝远程 TCP/SSH 上下文,并使用新的空 Docker 配置,以便拉取是匿名的。仅请求完全限定的 `registry/repository@sha256:...` 目录条目。缺少 Docker 或公共注册表访问是可见的、非致命的先决条件:静态审计仍然可用,以后的 `./setup` 会重试。预加载阶段为 endpoint 准入保留 30 秒,并为每个声明的映像提供单独的 30 秒检查/下载窗口。当注册表速度需要更长的每映像窗口时,设置 `GSTACK_CSO_IMAGE_PULL_TIMEOUT_SECONDS=120`(接受范围:5-300 秒)。帮助器从目录大小派生总截止日期,并将整个阶段上限为一小时。每映像超时使该摘要不可用并继续下一个条目;达到硬总截止日期停止所有剩余工作。当注册表先决条件停止进一步下载时,setup 仍会在本地检查以后的目录条目,并报告每个已存在的摘要。后来的 setup 从确切的本地摘要恢复。应用程序、验证器和扫描器执行仍使用 `--pull=never` 而无出口。`gstack-cso doctor --repo PATH` 永远不会下载;它报告合格的配置文件不可用,直到该确切摘要和平台已存在于本地守护进程中。

普通的免费套件在没有要求 Docker 的情况下行使合约。免费通道的 `cso-docker-integration` 作业设置 `GSTACK_CSO_DOCKER_TESTS=1` 并运行 `test/cso-docker-integration.test.ts`;不可用的 Docker 或缺少包含先决条件会使该作业失败。节点生命周期和 stack-cold 文件需要分阶段的堆栈映像,因此它们通过 `test:cso:docker` 在 CSO 运行时工作流中运行。在使用工作的本地守护进程的情况下本地运行所有三个:

```bash
bun run build:cso
GSTACK_CSO_DOCKER_TESTS=1 bun run test:cso:docker
```

当分阶段映像工作流提供 `GSTACK_CSO_TEST_IMAGE`、平台、堆栈和确切的工具版本时,相同的命令还运行特定于堆栈的限定旅程。第二个原生限定作业等待每个相同架构的分阶段作业,因此 Rails 接收其应用程序摘要和分阶段的 PostgreSQL 摘要;缺少匹配的工件会使 gate 失败。Node 旅程从空的公共存档缓存开始,通过注册表限制的代理获取完整性固定的依赖项,在离线安装并执行,重现生产者可见的 fixture 缺陷,并验证绑定到其记录的关闭和准备证明的补丁。当其进程外见证验证外部断言时,帮助器仅发出 `runtime_tested` `RepairBundle`。规范的项目测试在目标控制的进程中运行,因此它们的完成仍为 `self_reported`:目标代码可以伪造 reporter 输出或终止 runner。记录的命令、计数、退出和输出哈希是诊断证据;它们不建立目标独立的测试完成。`tested` 状态仍保留给未来的目标独立见证,今天未发出。在运行时测试的 bundle、重放和当前源重新检查旅程通过之前,保留的修复 gate 仍挂起,并且目录提升被禁止。候选旅程还断言没有运行拥有的容器、准备调用目录、存档暂存目录或验证工作目录保留。未跳过的分阶段映像旅程不是通过的冷启动或私有保留修复结果。Bun 和 Python 具有冷采集、离线安装、引导、控制和项目测试旅程。Rails 针对相同的冷公共 gem 输入运行这些检查两次:一次使用两个一次性 SQLite 连接,一次使用新分阶段的 PostgreSQL sidecar 中的两个数据库。Rails fixture 仅在离线执行阶段构建源平台 `sqlite3` 和 `pg` 本机 gem。

准备输出通过 `docker cp` 离开其容器,无法读取 tmpfs mount。因此采集 `/archives` 目录和离线 `/work` 目录是由有界 tmpfs 支持的 Docker 本机卷,为每个容器使用私有模式创建,运行时 uid/gid、`nosuid` 和 `nodev`(`/archives` 也是 `noexec`;`/work` 必须保持可执行以供 virtualenv 和 binstub 启动器使用)。Runner 检查每个容器,如果 Docker 未保留这些选项,则拒绝启动它。npm 的用户和全局配置指向两个不同的空文件,因为 npm 拒绝在两个角色中加载一个文件。Rails 应用程序和测试容器在 `run-app` 复制源后在 `sleep` 上保持;`db:prepare` 仅在该复制完成后启动。对 `/opt/cso/preparation` 或映像配方的更改需要新的分阶段运行,然后才能进行原生限定。

每个受监督的命令都有 300 秒的上限(`lib/cso/process.ts` 中的 `COMMAND_TIMEOUT_MS`)。仅采集和离线准备阶段的依赖项获取和安装命令获得 `PREPARATION_COMMAND_TIMEOUT_MS`,即 900 秒。仅列出 `ruby` 平台的 Rails lockfile 离线编译 nokogiri、sqlite3 和 pg,在应用程序角色的 0.85 CPU 共享下大约需要 385 秒。两个上限都被削减到旅程总截止日期之前剩余的时间,因此较长的上限永远不会延长旅程。Helper 探针、导出、应用程序启动、`db:prepare` 和项目测试保持 300 秒上限。`test/cso-docker-mounts.test.ts` 中的免费测试固定哪些调用可以请求较长的上限。CPU 共享保持不变,因此 `ISOLATION_POLICY_HASH` 和绑定到它的配置文件不受影响。

当映像内 helper 拒绝导出准备好的树时,准备失败,错误代码为 `PREPARED_EXPORT_REJECTED`,而不是通用的 `TOOL_FAILED`。该消息携带 helper 的退出状态及其固定原因,例如 `prepared tree contains a hard-linked file`,或文件系统 errno 和 syscall。它从不包括目标路径。没有重试。将复发视为缺陷的证据,而不是要重新运行的 flake。

Rails 离线安装运行 `bundle install --jobs 1`。Ruby 的 `File.umask` getter 将进程 umask 瞬间设置为 0,RubyGems 为其提取的每个文件调用它。使用两个 Bundler 安装线程,在该瞬间由一个线程创建的目录出来为 0777,并且导出以 `prepared tree contains a publicly writable directory` 拒绝该树。该拒绝是正确的,因此安装在一个线程上运行,而不是放宽 gate。在应用程序角色的 0.85 CPU 共享下,第二个线程没有节省可测量的时间。

对于由 pip 解析的哈希 Python 需求,代理记录请求的索引 URL、允许的联系的主机和已验证的 wheel 哈希。Pip 的内部 `files.pythonhosted.org` 响应 URL 对该 TLS 边界不透明,因此 `resolvedUrl` 仍为 `null`;`registryResponseSha256` 标识获取的存档字节,而不是已验证的响应 URL。

映像发布从提交的 [`build-inputs.json`](../lib/cso/images/build-inputs.json) 开始。其审查的修订固定每个堆栈的不同原生清单和 SBOM 生成器。每个配置文件记录检查的源标签和不可变索引以进行出处,然后 `baseImages` 由 `linux/amd64` 和 `linux/arm64` 键控;Python 此外记录原生 uv 清单。如果源标签不再解析为已审查的索引、该索引中缺少平台清单、其映像配置命名另一个架构,或可执行文件报告不同的运行时或包管理器版本,则 CI 失败。Bun 不声明 Node 版本,因为其容器的 `node` 路径是 Bun 支持的回退,而不是 Node 版本。因此具有 Node 引擎约束的 Bun 项目会收到显式先决条件,而不是伪造的兼容性结果。

使用所需的维护者审查和仅限于受保护 `main` 的部署来配置仓库的 `cso-runtime-release` GitHub 环境。[`cso-runtime-images.yml`](../.github/workflows/cso-runtime-images.yml) 工作流在 pull request 上运行仅构建的原生矩阵。其手动发布作业仅接受 `main`。它们在原生 amd64 和 arm64 runner 上构建,发布到 `cso-staging` 命名空间,签署出处和 SBOM 证明,并验证工作流标识和源提交。然后它将确切的摘要拉入本地守护进程,以进行禁止隐式拉取的测试。每映像烟雾测试对具有通过控制和故意失败控制的环回服务以及包含套件一起执行嵌入式验证器。在所有分阶段行完成后,原生限定作业运行上述匹配的冷启动旅程。Node 行另外运行完整的生命周期旅程。每个原生作业命名为 `qualify-native <runtimeId>` 并使用 Bun 的 JUnit reporter 运行 Docker 套件(`--reporter=junit --reporter-outfile=native-tests.junit.xml`)。[`cso-native-evidence.ts`](../scripts/cso-native-evidence.ts) 将每个原生检查键映射到证明该堆栈的命名测试,并且仅当报告没有失败的测试用例并且每个命名测试出现一次、通过并至少进行一次断言时才记录键为 true。环境跳过的测试(例如没有 `GSTACK_CSO_TEST_IMAGE` 的分阶段运行时烟雾测试)报告为已跳过,并将其检查保留为 false;任何错误的原生检查都会使作业失败。待处理的私有检查是该堆栈从 `cso-runtime-promotion.ts` 中减去其原生键所需的发布 gate:`secretCanaryPassed` 和 `watchdogCleanupPassed` 用于 PostgreSQL,加上 `heldOutRepairPassed` 和 `accuracyGatesPassed` 用于应用程序堆栈。该作业在 `qualification-evidence.json` 旁边上传 JUnit 报告,该报告记录其摘要。注册表发布是分阶段的,而不是运行时限定。

GitHub Container Registry 创建新的私有包。在分阶段摘要可以进入原生或私有限定之前,包管理员必须在 GitHub 的包设置中将其包公开。这是一个显式的引导步骤,因为 GitHub 将可见性更改记录为不可逆,并且不为其提供受支持的工作流 API。发布作业使用其范围化的工作流 token 从 GitHub 的包 REST 端点要求 `visibility: public`,然后将确切的平台摘要拉入 `docker --config` 指向一个新目录,该目录的 `config.json` 仅包含空的 `auths` 对象。GitHub token、Docker 凭据、上下文、证书路径和凭据帮助器在该 Docker 客户端中不存在。私有包、元数据不匹配、失败的匿名拉取或不同的返回摘要会停止分阶段、限定和提升。运行时和扫描器提升在新的托管 runner 上重复此匿名拉取检查,然后再打开目录 PR。

因此,第一次受保护的主分阶段运行按设计在公共包检查处停止:其推送将十个 `cso-staging` 包创建为私有。包管理员将这些十个包公开,然后再次调度分阶段工作流。以后的运行将新摘要发布到同一公共包中。

一起审查保留的分阶段工件以及 [`qualification.json`](../lib/cso/images/qualification.json) 中的所有检查。冷启动 fixture、保留的修复评估、精度/召回阈值、崩溃清理和所有秘密金丝雀检查必须通过,然后审查的目录更改才能使摘要可执行。成功的受保护主限定运行为每个堆栈和架构打包一个 `qualified-runtime.json` 语句,位于 `cso-qualified-runtime-statements` 工件下。所有十个语句绑定相同的源提交和工作流运行。受保护的 [`cso-runtime-promote.yml`](../.github/workflows/cso-runtime-promote.yml) 工作流通过 Actions API 验证该运行,根据提交的构建配置文件检查每个语句,并发出经验证的 `runtime-catalog.candidate.json`。受保护的作业立即根据其工作流标识、受保护的主源 ref、源提交和确切的文件摘要验证候选证明。然后它将 `previousRevision` 与源目录进行比较,将这些相同的字节提交到新分支,并打开正常的审查 PR。使用工作流的 `GITHUB_TOKEN` 进行的推送和拉取请求不会启动工作流,因此在推送分支之后,作业使用 `gh workflow run free-tests.yml --ref <branch>` 调度 [`free-tests.yml`](../.github/workflows/free-tests.yml)(其 `actions: write` 授权)。该运行在 PR 提出的分支 head 上报告所需的 `free-tests` 检查。打开 PR 还需要存储库设置 **Allow GitHub Actions to create and approve pull requests**(设置 → 操作 → 常规 → 工作流权限)。没有它,`gh pr create` 在推送和调度之后失败,并且维护者从推送的分支打开 PR。同时进行的目录提升会使比较和交换失败,或产生合并冲突,而不是默默替换较新的矩阵。目录存储保留的运行时矩阵的单独可重新计算的摘要以及完整外部限定语句的摘要。即使保留了语句摘要字段不变,帮助器也会拒绝生成后的映像或限定变更。回滚保留先前的兼容目录修订。

公共限定入口是 [`cso-runtime-qualification.yml`](../.github/workflows/cso-runtime-qualification.yml)。将受保护的环境变量 `CSO_QUALIFICATION_ACTOR` 设置为私有评估器使用的 GitHub 服务帐户。在其保留的断言和准确性运行通过后,该帐户发送 `repository_dispatch` 事件,类型为 `cso-runtime-qualified`。有效负载具有键 `statements` 和可选的 `evaluationRef`;不接受其他键。`statements` 包含十个对象,具有 `schemaVersion`、`helperAbi`、`state`、`buildRevision`、`runtimeId`、`stack`、`platform`、不可变的 `image`、确切的 `versions`、`sourceCommit`,以及 `cso-runtime-promotion.ts` 接受的真实值检查。所有十个命名一个 `sourceCommit`。`evaluationRef` 是不透明的 `sha256:<64 hex>` 引用,允许审批者将调度与私有评估器的裁决匹配。没有 token 权限的未受保护的 `dispatch-summary` 作业验证有效负载形状,并将 `evaluationRef`、源提交以及每个 `runtimeId` 和 `image` 对写入运行的作业摘要。审阅者在那里阅读它,然后再批准受保护的 `qualify` 作业。`evaluationRef` 从不进入规范化语句、上传的证据或目录。

受保护的作业检查参与者,根据受保护的分阶段工作流和源提交重新验证两个 OCI 证明,并将其观察到的工作流、时间、SBOM 摘要和出处摘要替换为其观察到的值。证明在分阶段作业中签署,在原生限定运行之前,因此仅有效的证明并不表明原生 gate 通过。[`cso-staging-run.ts`](../scripts/cso-staging-run.ts) 因此从每个已验证的 SLSA 出处语句解析分阶段运行:谓词的 `runDetails.metadata.invocationId` 必须等于签名证书的 `runInvocationURI`,并且所有十个映像必须解析为该仓库的同一运行。通过 Actions API(`actions: read`),入口然后要求该运行为 `.github/workflows/cso-runtime-images.yml`,在此仓库的 `main` 上由 `workflow_dispatch` 启动,以结论 `success` 完成,并从语句的 `sourceCommit` 构建。最新的尝试 `qualify-native <runtimeId>` 作业,在作业 API 的每一页上读取,必须正好是每个提交的矩阵行(十个)一个,并且全部 `success`。运行和作业记录保留为 `staging-run-evidence`。目录生成器最后运行作为最终 schema gate,并且入口仅在该 gate 通过后上传语句。有效负载不包含保留的断言、应用程序源、发现或修复 bundle。

限定仅在测量下的输入未更改时有效。[`cso-requalification.ts`](../scripts/cso-requalification.ts) 定义四个重新限定触发器:helper ABI(`CSO_HELPER_ABI`)、隔离策略(`ISOLATION_POLICY_HASH`,涵盖 `cso-isolation-v1` 策略版本和每个限制)、`preparationSha256`(编译或复制到运行时映像中的每个源字节的摘要:映像的验证器和准备 helpers 的导入闭包加上映像构建上下文)和 `build-inputs.json` 修订。分阶段运行的 `reviewed-inputs` 作业将其源提交的这些值记录为 `cso-requalification-triggers` 工件。入口从已验证的分阶段运行下载该工件到证据中,并且提升生成器将其写入 `promotion.requalification`。生成拒绝与提升 checkout 不同的触发器,因此分阶段和提升之间更改的触发器需要新的分阶段运行。免费套件的 [`cso-requalification.test.ts`](../test/cso-requalification.test.ts) 比较提交的目录的记录触发器与当前源,并且在任何不同时失败。因此,对触发器的更改要么在重新限定的目录中登陆,要么在同一拉取请求中撤回目录的运行时。独立地,help 拒绝选择在其他隔离策略下记录的运行时。

法官在一次引导中观察保留的攻击:`VerificationRequest` 的 `security` 字段是一个断言或最多八个具有不同名称的断言的数组,在单个 before/after 对中进行判断。仅当每个断言显示其易受攻击的期望时,before 阶段才是 `intended_failure`,并且仅当每个断言通过其固定的期望时,after 阶段才会通过。单个断言保留其原始哈希。验证器被编译到运行时映像中,因此数组请求需要从包含它的提交分阶段映像。

### 仅评估的 helper 构建

私有评估器在分阶段摘要的任何 gate 被测量之前驱动真实的 helper。它从不编辑提交的目录。相反,`bun scripts/cso-runtime-promotion.ts evaluation-candidate --staged-root <dir> --output <new-file>` 将一个分阶段运行的十个 `staged-image.json` 记录转换为仅评估目录。其修订是 `cso-eval-<runId>-<sha12>`,它携带 `evaluation` 记录而不是 `promotion`,并且每个运行时都是 `evaluation_candidate`,其限定的种类为 `kind: "evaluation"`,没有发布 gate 字段。每个运行时的 `qualifiedAt` 是当前时间,除非 `--created-at <ISO time>` 命名它。为每个平台构建一个单元的评估必须将相同的 `--created-at` 传递给每个构建:评分者要求两个平台的生产者嵌入相同的目录字节。然后

```bash
bun run build:cso -- --evaluation-candidate <catalog.json> --output <new-directory>
```

[desktop write output truncated: original_bytes=27256; if the omitted remainder is needed, continue with read using offset=323 and the same path.]
