---
name: careful
version: 0.1.0
description: 针对破坏性命令的安全护栏。(gstack)
triggers:
  - be careful
  - warn before destructive
  - safety mode
allowed-tools:
  - Bash
  - Read
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: 'bash -c "exec bash \"$HOME/.claude/skills/gstack/careful/bin/check-careful.sh\""'
          statusMessage: "Checking for destructive commands..."
    - matcher: "PowerShell"
      hooks:
        - type: command
          command: 'bash -c "exec bash \"$HOME/.claude/skills/gstack/careful/bin/check-careful.sh\""'
          statusMessage: "Checking for destructive commands..."
---
<!-- AUTO-GENERATED from SKILL.md.tmpl ; do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## 何时调用该技能

在执行 rm -rf、DROP TABLE、force-push、git reset --hard、kubectl delete 及类似破坏性操作之前发出警告。
用户可以逐个覆盖每个警告。在接触生产、调试实时系统或在共享环境中工作时使用。
在用户说「be careful」、「safety mode」、「prod mode」或「careful mode」时使用。

# /careful ; 破坏性命令护栏

安全模式现已**启用**。每个 Bash 和 PowerShell 命令在运行前都会被检查
是否存在破坏性模式。如果检测到破坏性命令,你会收到警告
并可以选择继续或取消。

```bash
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
mkdir -p "$GSTACK_STATE_ROOT"/analytics
echo '{"skill":"careful","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> "$GSTACK_STATE_ROOT"/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 受保护的内容

| 模式 | 示例 | 风险 |
|---------|---------|------|
| `rm -rf` / `rm -r` / `rm --recursive` | `rm -rf /var/data` | 递归删除 |
| `DROP TABLE` / `DROP DATABASE` | `DROP TABLE users;` | 数据丢失 |
| `TRUNCATE` | `TRUNCATE orders;` | 数据丢失 |
| `git push --force` / `-f` | `git push -f origin main` | 历史重写 |
| `git reset --hard` | `git reset --hard HEAD~3` | 未提交工作丢失 |
| `git checkout .` / `git restore .` | `git checkout .` | 未提交工作丢失 |
| `kubectl delete` | `kubectl delete pod` | 影响生产 |
| `docker rm -f` / `docker system prune` | `docker system prune -a` | 容器/镜像丢失 |

## PowerShell 和 cmd(Windows)

该钩子还检查 PowerShell 工具(Claude Code 的主要 Windows shell)
和任何从 Bash 启动的 `pwsh`/`powershell`/`cmd`。匹配不区分大小写,涵盖命令位置的别名,接受参数前缀(`-r`、`-fo`)并剥离 cmd `^` 转义。所有上面行也都适用。

| 模式 | 示例 |
|---------|---------|
| `Remove-Item`/`rm`/`ri`/`del`/`erase`/`rd`/`rm` + `-Recurse` 或 `-Force` | `gci \| ri -r -fo` |
| cmd `rd /s`、`rmdir /s`、`del /s`、`erase /s` | `cmd /c rd /s /q C:\proj` |
| `Format-Volume`、`Clear-Disk`、`Clear-Content`、`[IO.Directory]::Delete`、`[IO.File]::Delete` | `Clear-Disk -Number 1` |
| `-EncodedCommand`、`iex`、`Start-Process` 一个 shell、`& $cmd`(无法检查) | `irm $url \| iex` |

**PowerShell 上的尽力而为。** 字符串匹配无法看到运行时构造的命令。
要硬停止,请添加 Claude Code 权限拒绝规则,它们会解析
PowerShell 及其别名: `"deny": ["PowerShell(Remove-Item *)"]` 在
`.claude/settings.json` 中。该钩子通过 `bash` 运行,所以 Windows 需要 Git Bash。

## 安全例外

以下模式无需警告即可放行:
- `rm -rf node_modules` / `.next` / `dist` / `__pycache__` / `.cache` / `build` / `.turbo` / `coverage`

## 工作原理

钩子从工具输入 JSON 中读取 `tool_name` 和命令,对照上面的模式进行检查,如果找到匹配则返回 `hookSpecificOutput` 载荷,带有 `permissionDecision: "ask"` 和警告原因(该决策必须位于 `hookSpecificOutput` 下 ; Claude Code 忽略顶层 `permissionDecision`)。你始终可以覆盖该警告并继续。

## HIGH 等级(硬拒绝)

两种灾难性形态会被**拒绝**(denied),而不是询问(asked):`rm -r`/`-R` 恰好是 `/`、`~` 或 `$HOME`,以及强制推送到 repo 的**默认分支**。仅限简单命令(无 `;`、`&&`、`||`、`|`、换行) ; 复合形态落到 MEDIUM 询问;`--force-with-lease` 从不是 HIGH。尽力而为的咨询性硬停止,不是策略边界:逃生口是结束选择加入的会话范围 /careful 会话。

## 项目模式(仅添加)

在 `~/.gstack/careful-patterns.txt`(全局)或
`~/.gstack/projects/<slug>/careful-patterns.txt`(每项目)中按 POSIX ERE 规则(每行一个,允许 `#` 注释)添加警告规则。
在内置家族之后查阅,所以配置只能添加规则,永远不能抑制基线警告。无效的正则表达式行会被跳过。

要停用,请结束对话或开启新对话。钩子是会话范围的。