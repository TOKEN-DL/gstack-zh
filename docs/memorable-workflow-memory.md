# 使用 Memorable(可选,第三方)的工作流记忆

第三次你要求 Claude Code 做相同形状的工作时,它会再次从零开始。它重新读取相同的文件、重新运行相同的搜索,并得出它上个月已经编写的修复。**Memorable** 是一个第三方 CLI,记录任务是如何完成的,并在下一次你要求类似的事情时将其交回。

gstack 不安装它、打包它或依赖它。gstack 添加的是**桥梁**:Memorable 的 `UserPromptSubmit` hook 通过 gstack 自己的 hook manager 注册,包装在 gstack 为每个其他外部 sink 提供的保证中。它在你打开之前是关闭的,Claude Code 是它唯一能工作的主机,并且 Windows 上尚不可用。

## 你得到什么

- 在每次提示之前,hook 询问 Memorable 过去的会话是否已经解决了与此接近的问题,如果是,则将该过程注入为清晰标记的参考数据。
- 一个显式的 gstack 端同意密钥 `memorable_recall`,默认情况下关闭,由 `gstack-egress grants` 列出其撤销命令。
- 每次交给供应商二进制的提示都有一个收据,在交接之前(`gstack-egress list --sink memorable-recall`)。没有收据,没有交接。
- 一个 HIGH 级别的秘密预扫描:携带实时形状凭据的提示永远不会被移交。
- 信任信封和 8 KiB 上限,无论返回什么,供应商都无法阻止提示或代表 gstack 发言。
- 供应商在允许列表的环境中运行(没有来自你的会话的 API 密钥)在其自己的进程组内,并且整个组在 hook 完成、超时或被 Claude Code 中途中止时被杀死。供应商故意分离到其自己的会话(`setsid`)的进程在该组之外,并且在此保证之外;这是供应商自己行为中可见的选择,而 gstack 无法阻止。
- 在稳定安装时注册,在每个 `./setup` 上修复,在 `./setup --no-team` 中幸存,由 `gstack-uninstall` 删除,以及一个即使在 Claude Code 重写 `settings.json` 之后也能工作的关闭开关。

## 这不是什么

- 不是确定性重放。它是被回忆的指导,模型可能会忽略。
- 与 Aside 或浏览器自动化无关。
- 不是 gstack 功能,gstack 的保证超出了流程边界。`memorable` 二进制文件在 gstack 将其交给它之后所做的所有内容都属于另一个供应商的闭源 npm 包。

## 两个同意,彼此不隐含

| 同意 | 设置者 | 它控制什么 |
|---|---|---|
| `memorable_recall` (gstack) | `gstack-memorable enable` / `disable` | gstack 的 hook 是否将提示交给供应商二进制 |
| Memorable 自己的同意 | `memorable enable` / `disable` / `forget`,由你运行 | Memorable 是否存储过程,并根据其文档将其会话跟踪发送到其提取 API |

gstack 永远不会运行供应商的同意命令,也永远不会读取其状态。`gstack-memorable enable` 打印它们,以便你自己运行或检查它们。关闭桥梁会关闭 gstack 的交接;它不会更改 Memorable 允许对其已有内容执行的操作。

## gstack 交出什么,以及它能证明什么

| 命令 | gstack 交给供应商二进制的内容 |
|---|---|
| `gstack-memorable status`、`enable`、`disable` | 无。它们检查二进制是否存在;它们永远不会执行它。 |
| hook,在每次提示上(gate on) | Claude Code 的 `UserPromptSubmit` JSON:`session_id`、`cwd`、`transcript_path`、`prompt`。二进制以你的权限运行,因此它可以读取你可以访问的任何内容,包括该路径命名的脚本。 |

收据确切地证明这些字节(计数和 sha256),并命名 gstack 实际运行的接收者:`local:<path to the memorable executable>`。它不会也不能证明供应商的网络活动。Memorable 的文档声明,当本地词汇匹配未命中时,recall 仅通过其 API 嵌入一条清理过的任务行,并且捕获上传工具名称、允许列表参数字段和 200 字符的任务行。这些是供应商的说法。该桥梁针对 memorable-cli 0.5.18 的 hook 合约进行了测试;其他版本是供应商的兼容性声明。

该二进制的环境是允许列表,而不是你的会话:`PATH`、`HOME`、用户和 shell 名称、区域设置(`LANG`、`LC_*`)、临时目录、标准的代理和 TLS 变量(`HTTP_PROXY`、`HTTPS_PROXY`、`NO_PROXY`、`SSL_CERT_FILE`、`SSL_CERT_DIR`、`NODE_EXTRA_CA_CERES`)、`XDG_*` 目录,以及每个 `MEMORABLE*` 变量。没有 `ANTHROPIC_API_KEY`、没有 `GSTACK_*`、没有 `CLAUDE_*` 到达它。每当修订引擎在其 stderr 中找到 HIGH 或 MEDIUM 级别的形状(凭据、电子邮件、电话号码)时,其 stderr 就会被排除在 `hook-errors.log` 之外,因此在错误时回显其输入的供应商无法将你的提示复制到日志中。收据的 `payload_class` 是 token `claude-user-prompt-json->local-vendor-cli`:提示 JSON,交给本地供应商可执行文件;网络目的地对 gstack 来说是未知的(Memorable 声明:其嵌入 API,在本地回忆未命中时)。

当 gate 关闭或设置 `MEMORABLE=0` 时,hook 静默跳过交接(没有拒绝,所以没有记录)。

它在以下情况下拒绝交接,在 `~/.gstack/hook-errors.log` 中有一行速率限制:

- 供应商二进制缺失;
- 提示携带 HIGH 级别的凭据形状(在原始字节和解码的字符串值上检查,因此 JSON 转义的键不会滑过),或大于 1 MiB;
- 仓库的每个远程信任策略是 `deny` 或 `read-only`(根据会话的工作目录判断,因此涉及其他仓库的会话不在覆盖范围内),或者无法查找该策略(git 无法读取仓库,存储不可读):查找失败关闭;
- 无法写入收据,或者 hook 的 4.5 秒预算无法负担下一步(非常大的提示的秘密扫描,或供应商 spawn)。

结果缺失的收据意味着主机杀死了 hook 或时钟用完了。将其读取为未知,永远不要成功。结果为 `output-written` 意味着 gstack 在其 stdout 上为 Claude Code 注入了封装的上下文;Claude 是否使用它是 hook 无法知道的事情。

## gstack 测试的内容,以及什么是 Memorable 的声明

gstack 在 `test/gstack-memorable.test.ts`、`test/memorable-user-prompt-hook.test.ts` 和 `test/gstack-settings-hook-schema-aware.test.ts` 中针对假供应商测试桥梁:上述每个拒绝、收据前于交接的顺序、信封和上限、环境允许列表、进程组 kill、Claude Code 剥离标记后的基于身份的删除、扫描排除、卸载分支和退出代码。超出流程边界的所有内容、Memorable 存储的内容、在哪里、它发送的内容以及 `memorable disable` 和 `memorable forget` 删除的内容,都是 Memorable 的声明,而不是我们的。

## 打开它(三步,中间一步是你的)

```bash
npm i -g memorable-cli          # the CLI, from npm, not from gstack
memorable login && memorable enable   # the vendor's account and the vendor's consent; only you can do these
gstack-memorable enable         # gstack's gate + the hook, at the stable install
```

当供应商二进制缺失、当 Memorable 已注册其自己的 hook(见下文)、当稳定安装不携带此桥梁(首先运行 `./setup`)或当无法读取 `settings.json` 时,`enable` 会拒绝并且不更改任何内容。Claude Code 在当前版本上的几秒钟内选取新 hook;如果没有触发,请重新启动会话。验证:

```bash
gstack-memorable status
```

## 如果 Memorable 已注册 hook

`memorable install-hooks` 和 `memorable start` 在 Memorable 自己的名称下注册相同的 `UserPromptSubmit` hook,在 gstack 的表之外(`memorable enable` 不会;只有这两个会)。在这种情况下,`enable` 拒绝而不是添加第二个条目:两个条目在每次提示上运行 hook 两次,注入两次并针对你的允许捕获两次。

要将其交给 gstack,请从 `~/.claude/settings.json` 中删除该条目,然后再次运行 `enable`。Memorable 没有删除其自己 hook 的命令。

## 关闭它

```bash
gstack-memorable disable
```

Gate 首先关闭,因此即使在条目消失之前,下一次提示也是关闭的;已经过去的提示完成。然后按身份删除 gstack 的条目(标记或无标记),验证两个结果,并使用非零退出报告任何部分失败。Memorable 自己的同意及其存储的任何内容保持不变:`memorable disable` 停止捕获,`memorable forget` 拒绝一切,并且其自己的文档说明每个擦除的内容。

该条目也随 `gstack-uninstall` 一起删除(在其摘要中命名)并在 `./setup --no-team` 中幸存,该设置仅拆除 team-mode hook。

## 如果你运行 gbrain

Memorable 可以将其过程保留在你自己的 gbrain 数据库中,而不是其本地存储(`memorable init gbrain`,根据其文档)。这会更改供应商存储内容的位置;它不会更改此桥梁的任何内容,该桥梁只会将提示交给本地 `memorable` 二进制文件。gstack 的 `/setup-gbrain` 和 `/sync-gbrain` 与它无关。

## 故障排除

- **hook 永远不会触发。** `gstack-memorable status` 应该显示 gate `on` 和 "registered by gstack"。如果它显示不匹配行,请遵循它。如果一切看起来正确,请重新启动 Claude Code 一次。一个已知的竞争:Claude Code 按自己的计划重写 `settings.json`,在 `enable` 期间落地的重写可以在 gstack 打印 `registered` 后删除条目;hook 管理器是会聚的,而不是独占的,因此再次 `enable`(它报告 `unchanged` 或 `registered`)并检查 `status`。
- **永远不会被回忆。** 如果 `gstack-egress list --sink memorable-recall` 显示具有 `output-written` 或 `injected=no` 结果的收据,则交接正在发生。`injected=no` 意味着供应商未返回任何内容:检查 `memorable status` 和 `memorable doctor` 的登录、同意和存储的过程。
- **结果说 `timeout`。** 供应商花费的时间超过了预算所允许的时间(gstack 自己的工作后大约 4 秒)。这通常是网络。
- **`hook-errors.log` 命名拒绝。** `refused:redaction-high` 意味着提示中存在凭据形状;`trust policy ... deny or read-only` 意味着仓库受到保护;`receipt-unwritable` 意味着 `~/.gstack/security` 不可写。
- **`enable` 表示稳定安装早于此桥梁。** 运行 `./setup`(或 `/gstack-upgrade`),以便在 `~/.claude/skills/gstack` 注册的 hook 是将运行的代码。

## 准确的内部结构

`bin/gstack-memorable` 是前门。它从 `GSTACK_MEMORABLE_BIN`、然后 `MEMORABLE_BIN`、然后 `~/.memorable/bin/memorable`、然后 `PATH` 解析供应商 CLI,当找不到时拒绝并命名这些消息。它永远不会执行二进制。它在 `enable` 和 `disable` 期间持有锁,首先捕获先前的状态,并且在失败时恢复该状态,而不是假定的状态。

`hosts/claude/hooks/memorable-user-prompt-hook` 是注册的命令:在 `memorable-user-prompt-hook.ts` 之上的失败打开 bash shim,它运行上述管道并始终退出 0。

注册通过 `gstack-settings-hook ensure-event` 进行,超时为 5 秒,并且该 hook 在该文件的 `KNOWN_HOOKS` 表中有一行,因此它由其命令标识,而不是由标签标识。后果:

- `gstack-settings-hook list-items --event UserPromptSubmit --owned-by gstack-memorable` 显示它是否存在标签;
- `list-sources` 仅显示标记的条目,因此它可能不会显示。
- `prune-stale --repoint`(由每个 `./setup` 运行)修复陈旧的路径。
- `./setup --no-team` 将其排除在其扫描之外;`gstack-uninstall` 删除它。
- `gstack-settings-hook rollback` 是最后一次突变的整个文件还原,而不是每个 hook 的撤消。

Windows:尚未。没有要包含供应商的进程组,因此 `enable` 拒绝并且 hook 退出 0。在 TODOS.md 中跟踪。

## 致谢

集成及其 hook 合约由 [Advaiyt Sane](https://github.com/AdvaiytSane)(@AdvaiytSane)和 [Nikhil Krishnaswamy](https://github.com/NIkhil-cmd-cmd)(@NIkhil-cmd-cmd)在 Memorable(#2831)中提供。同意、收据、信封和包含层在评审中添加。
