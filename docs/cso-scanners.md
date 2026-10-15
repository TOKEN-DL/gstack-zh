# 运行 CSO 扫描器

`gstack-cso scan RUN SCANNER [REQUEST.json]` 在存在匹配的合格扫描器目录配置文件时,通过可信 helper 收集候选证据。已识别的 ID 是 `gitleaks`、`osv`、`semgrep`、`zizmor`、`trivy` 和 `schemathesis`。没有合格的配置文件,该命令记录特定的先决条件;静态评估和使用 `gstack-cso import-sarif RUN REPORT.sarif` 导入现有 CodeQL 或其他 SARIF 仍然可用。返回的不可变工件可以稍后使用 `gstack-cso scanner-outcome RUN ARTIFACT_ID` 读取;调用者永远不需要直接打开私有状态文件。

源控制的目录 [`scanner-images/catalog.json`](../lib/cso/scanner-images/catalog.json) 在每个发布 gate 产生真实证据之前为空。没有审查的配置文件,该命令记录特定的 `not_assessed` 先决条件。在主机上安装扫描器不会启用执行。每个配置文件绑定 Linux 平台、不可变的映像摘要、固定的入口点、绝对的扫描器可执行文件、版本探测哈希、必需的适配器能力、helper ABI 和确切的隔离策略哈希。Helper 在启动时验证该目录,并且没有可执行回退。Setup 使用匿名公共注册表访问通过可信安装 helper 预加载当前平台的合格扫描器摘要。失败的预加载或有限的每映像/setup 截止日期是非致命的先决条件,永远不会导致审计拉取。默认每映像为 30 秒;`GSTACK_CSO_IMAGE_PULL_TIMEOUT_SECONDS=120` 选择更长的允许值(接受范围:5-300 秒),而完整的预加载仍上限为一小时。Doctor 是只读的,并且报告每个扫描器不可用,除非其确切的目录摘要已本地存在。

所有扫描器进程都使用 Docker 包含 watchdog、共享准入限制、只读源、有界输出和隔离的网络命名空间。静态扫描器没有网络访问,包括当审计省略 `--offline` 时。Semgrep 规则和 OSV/Trivy 数据库必须烘焙到已审查的映像中。它们的内容哈希、范围和新鲜度伴随结果。数据库刷新是单独审查的映像采集/发布操作;扫描永远不会下载缺少的资产或切换到不受限制的注册表网络。

`REQUEST.json` 可以选择确切的扫描器 `profile`。Schemathesis 还需要综合模式和 `api` 对象;`gstack-cso schema` 描述其字段。API harness 提供有界的 OpenAPI 3.0/3.1 JSON 文档、1-20 个唯一路径操作 ID、合格的应用程序运行时、绝对的启动命令、数字 loopback 端口、合法的 HTTP 控制以及定义安全边界的源文件。允许内部模式引用。服务器覆盖、远程引用/示例、hooks、回调和 webhook 被拒绝。应用程序必须在扫描器启动之前引导并通过控制。应用程序和扫描器仅共享允许的 loopback 命名空间。API 失败是候选,永远不是重现的漏洞或经过认证的修复。需要依赖准备的源报告该先决条件,直到 API runner 可用已准备的离线闭包。

每次执行在扫描之前探测扫描器版本,并在收集之前和之后验证保留的源。未知输出、失败的诊断、超时、更改的源或不完整的清理不能成为空的干净结果。当适配器可以将其与失败的覆盖区分开来时,Helper 会保留有效的部分候选。扫描器输出被就地修订:每个定位的敏感范围都成为 `<REDACTED-...>` 标记,因此类似于电话号码或 IP 地址的建议 URL 或版本字符串不会丢弃整个报告。无法修订的输出(包括跨 stdout 和 stderr 拆分的范围)被扣留。

扫描器和 SARIF 收集持有运行的 mutation lock,并拒绝已完成或中断的报告。每个结果仅在唯一 `scanner-outcomes/` 名称下写入,然后其覆盖记录进入报告。返回的 `artifact` 路径相对于运行目录。模型提交不能替换 `scanner:*` 覆盖或认证导入的候选。免费的扫描器测试使用注入的 runner 和目录来验证调度和失败处理;它们不声称为真实的映像限定。可选的 Docker 限定测试运行生产适配器、映像、watchdog 和隔离策略。它仅在代表性扫描完成且没有覆盖间隙后才发出版本输出哈希。

审查的发布输入 [`build-inputs.json`](../lib/cso/scanner-images/build-inputs.json) 涵盖两个原生平台上的所有六个扫描器。输入要求不可变的 base 和 SBOM 生成器映像摘要、确切的源提交和版本、签名者工作流标识和摘要,以及审查的规范 SLSA/SPDX 语句集摘要。工作流在使用之前以加密方式重新验证每个映像,具有这些存储库、源、签名者和谓词约束。Semgrep 还需要审查的本地规则包。OSV 和 Trivy 需要具有记录的内容哈希、新鲜度和生态系统覆盖的完整离线数据库包。发布工作流不会发明或静默替换它们。上游扫描器映像均未发布 GitHub 工件证明,因此审查的 base 来自 [`garrytan/gstack-cso-scanner-bases`](https://github.com/garrytan/gstack-cso-scanner-bases)。该存储库重新发布每个固定的上游摘要,并添加离线资产,并使用自己的工作流签署出处和 SPDX SBOM。每个标记的版本附加一个 `pending` `build-inputs.json` 候选;审阅者独立重新验证身份和哈希,然后此处的状态更改为 `reviewed`。

[`cso-scanner-images.yml`](../.github/workflows/cso-scanner-images.yml) 允许调度的分支运行只读输入和合约验证。映像发布、原生 amd64/arm64 限定、证明验证、嵌入资产哈希检查、包含、真实适配器执行和目录提案生成仅从受保护的 `main` 通过 `cso-scanner-release` 环境运行。在启用限定或提升之前,使用所需的维护者审阅者和仅限于受保护 `main` 的部署来配置仓库的 `cso-scanner-release` GitHub 环境。否则 GitHub 创建引用的缺失环境,没有保护规则。提升需要来自 `main` 的显式调度和通过该配置环境的批准。它打开一个仅包含提议的源目录的拉取请求,因此代码审查仍然是最终的授权步骤。使用工作流的 `GITHUB_TOKEN` 进行的推送和拉取请求不会启动工作流,因此在推送提升分支之后,作业使用 `gh workflow run free-tests.yml --ref <branch>`(其 `actions: write`)调度 [`free-tests.yml`](../.github/workflows/free-tests.yml);该运行在分支 head 上报告所需的 `free-tests` 检查。打开 PR 还需要存储库设置 **Allow GitHub Actions to create and approve pull requests**(设置 → 操作 → 常规 → 工作流权限);没有它,`gh pr create` 在推送和调度之后失败,并且维护者从推送的分支打开 PR。`previousRevision` 记录先前的兼容目录以进行回滚,并且必须在提升运行时仍等于当前 `main` 目录。

扫描器标志遵循 [`scanners.ts`](../lib/cso/scanners.ts) 中链接的主要文档。离线限定必须使用确切选择的版本进行测试:OSV 的维护者在 2.5.0 中记录了缓存位置回归,这证明了为什么版本字符串本身是不够的。[OSV issue #2983](https://github.com/google/osv-scanner/issues/2983)。Trivy 使用显式内存扫描缓存以及只读烘焙数据库和烘焙的错误配置检查包。Schemathesis 4.26 及更高版本重复模糊测试,直到用完 `--max-time`,因此适配器将该预算设置为扫描器截止日期的四分之三,并且仅当每个选定操作都行使时,将 `max_time` 停止视为完成。[Trivy filesystem reference](https://trivy.dev/docs/v0.68/guide/references/configuration/cli/trivy_filesystem/)。
