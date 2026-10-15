# Slop-scan:要修复的内容,要保留的内容

逐字从 CLAUDE.md 移出(减少 token 加载)。在对任何 slop-scan 发现采取行动之前阅读。

### 要修复的内容(真正的质量改进)

- **文件操作周围的空 catch** — 使用 `safeUnlink()`(忽略 ENOENT,重新抛出 EPERM/EIO)。在清理中吞下的 EPERM 意味着静默数据丢失。
- **进程 kill 周围的空 catch** — 使用 `safeKill()`(忽略 ESRCH,重新抛出 EPERM)。吞下的 EPERM 意味着你认为你杀死了某些东西,但实际上没有。
- **冗余的 `return await`** — 当没有封闭的 try 块时删除。节省一个微任务,发出意图。
- **类型化异常捕获** — `catch (err) { if (!(err instanceof TypeError)) throw err }` 当 try 块进行 URL 解析或 DOM 工作时,确实比 `catch {}` 更好。你知道你期望什么错误,所以说出来。

### 不要修复的内容(linter gaming,不是质量)

- **错误消息的字符串匹配** — `err.message.includes('closed')` 很脆弱。Playwright/Chrome 可以随时更改措辞。如果 fire-and-forget 操作可能因任何原因失败而你不在乎,`catch {}` 是正确的模式。
- **向通过的包装器添加注释以免除** — "alias for active session" 在方法之上仅仅触发 slop-scan 的免除规则是噪音,而不是文档。
- **将扩展 catch-and-log 转换为选择性重新抛出** — Chrome 扩展在未捕获错误上完全崩溃。如果 catch 记录并继续,这对于扩展代码是正确的模式。不要让它抛出。
- **收紧尽力而为的清理路径** — 关闭、紧急清理和断开连接代码应使用 `safeUnlinkQuiet()`(吞下所有错误)。在 EPERM 上抛出的清理路径意味着其余清理不会运行。那更糟。

### `lib/error-handling.ts` 中的实用程序(从 `browse/src/error-handling.ts` 重新导出)

| 函数 | 何时使用 | 行为 |
|----------|----------|----------|
| `safeUnlink(path)` | 正常文件删除 | 忽略 ENOENT,重新抛出其他 |
| `safeUnlinkQuiet(path)` | 关闭/紧急清理 | 吞下所有错误 |
| `safeKill(pid, signal)` | 发送信号 | 忽略 ESRCH,重新抛出其他 |
| `isProcessAlive(pid)` | 布尔进程检查 | 返回 true/false,从不抛出 |

### 分数跟踪

基线(2026-04-09,清理前):100 个发现,432.8 分数,2.38 分数/文件。
清理后:90 个发现,358.1 分数,1.96 分数/文件。

不要追逐这个数字。修复代表实际代码质量问题的模式。接受"草率"模式是正确工程选择的地方的发现。
