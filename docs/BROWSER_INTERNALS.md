# 浏览器 / 侧边栏 / 服务器内部

> **范围:** 这是 gstack 自己的浏览器引擎的内部参考 — `browse` 守护进程、GStack Browser、侧边栏扩展。该引擎是当 Aside 浏览器未安装或未运行时的自动**回退**(Linux、Windows、关闭的 Aside 应用程序);Aside 是每个 skill 首先驱动的内容。Aside 合约位于 `scripts/resolvers/aside.ts` 和 [BROWSER.md](../BROWSER.md)。

逐字从 CLAUDE.md 移出(减少 token 加载)。这些是 `browse/src/server.ts`、Chrome 扩展、侧边栏 PTY、SSE 端点、CDP 会话和侧边栏安全堆栈的承重不变量。此处的每条规则另外由其段落中命名的 CI tripwire 测试固定。

**侧边栏架构:** 在修改 `sidepanel.js`、`background.js`、`content.js`、`terminal-agent.ts` 或侧边栏相关的服务器端点之前,请阅读 `docs/designs/SIDEBAR_MESSAGE_FLOW.md`。侧边栏有一个主要表面 — **Terminal** 窗格(交互式 `claude` PTY) — Activity / Refs / Inspector 作为页脚 `debug` 切换后面的调试叠加层。PTY 证明之后聊天队列路径被拆除;`sidebar-agent.ts` 和 `/sidebar-command` / `/sidebar-chat` / `/sidebar-agent/event` 端点消失了。该文档涵盖了 WS 身份验证流程、双 token 模型和威胁模型边界 — 此处的静默失败通常追溯到不理解跨组件流程。

**Embedder terminal-agent 所有权** (v1.42.1.0+,基于身份的 kill v1.44.0.0+)。`browse/src/server.ts` 中的 `buildFetchHandler` 接受 `ServerConfig.ownsTerminalAgent?: boolean`(默认为 `true`)。当为 `true` 时,工厂关闭会获取 `acquireAgentStateLock(stateDir)` 并检查守护进程所有权。它仅在记录的智能体缺席、已经死亡或已通过 `stopAgentByRecord` 确认停止时才删除 `terminal-port`、`terminal-internal-token` 和匹配的 `terminal-agent-pid` 记录。不确定的标识或退出、不可用的锁或后继拥有的守护进程状态会使这些文件保持不变。预先启动自己的 PTY 服务器的 Embedders(例如 gbrowser phoenix overlay)必须传递 `false`,以便其发现文件在 gstack 拆卸周期中幸存。该标志是 `ServerConfig` 中的第三个调用者拥有的拆卸 gate(以及 `xvfb?` 和 `proxyBridge?`);极性是反转的(显式 bool 与存在性)并在字段的 JSDoc 中记录。CLI `start()` 始终显式传递 `true` — `browse/test/server-embedder-terminal-port.test.ts` 中的静态 grep 测试在重构删除它时使 CI 失败。Pre-v1.44 使用 `pkill -f terminal-agent\.ts`(正则表达式匹配)将杀死同一主机上的同级 gstack 会话;新的 `browse/test/terminal-agent-pid-identity.test.ts` 静态 grep tripwire 在任何源文件重新引入 `pkill ... terminal-agent` 或 `spawnSync('pkill', ...)` 时使 CI 失败。

**WebSocket 身份验证使用 Sec-WebSocket-Protocol,而不是 cookie。** 浏览器无法在 WebSocket 升级上设置 `Authorization`,但它们**可以**通过 `new WebSocket(url, [token])` 设置 `Sec-WebSocket-Protocol`。智能体读取它,根据 `validTokens` 进行验证,并且**必须**在升级响应中回显协议 — 没有回显,Chromium 会立即关闭连接。`Set-Cookie: gstack_pty=...` 保留为非浏览器调用者的回退(跨端口 `SameSite=Strict` cookie 路径无法从 chrome-extension 源存活)。

**跨窗格 PTY 注入。** 工具栏的 Cleanup 按钮和 Inspector 的"Send to Code"操作都将文本通过 `window.gstackInjectToTerminal(text)`(由 `sidepanel-terminal.js` 公开)管道传输到实时 claude PTY。没有 `/sidebar-command` POST — 实时 REPL 现在是侧边栏中唯一的执行表面。

**`/health` 不得泄露任何 token — 它也不再泄露** (v1.63+)。有头模式的 `AUTH_TOKEN` 的历史泄露已修复:`GET /health` 在每种模式下都是仅存活/状态。Token bootstrap 是 `POST /extension-token`,它根据固定的扩展标识(`extension/manifest.json` 中的 `key` 字段固定扩展 ID — `browse/src/server.ts` 中的 `GSTACK_EXTENSION_ID`,可通过 `bun browse/scripts/extension-id.ts` 重现派生)以及 loopback Host 验证调用者的 Origin。PTY 身份验证仍仅通过 `POST /pty-session` 进行。不要向 `/health` 添加任何 token。

**传输层安全性** (v1.6.0.0+)。当 `pair-agent` 启动 ngrok 隧道时,守护进程绑定两个 HTTP 监听器:本地监听器(127.0.0.1,完整命令表面,永不转发)和隧道监听器(锁定允许列表:`/connect`、`/command` 带范围化 token + 26 命令浏览器驱动允许列表、`/sidebar-chat`)。ngrok 仅转发隧道端口。通过隧道的根 token 返回 403。SSE 端点使用通过 `POST /sse-session` 铸造的 30 分钟 HttpOnly `gstack_sse` cookie(对 `/command` 永远无效)。隧道表面的拒绝转到 `~/.gstack/security/attempts.jsonl` 通过 `tunnel-denial-log.ts`。在编辑 `server.ts`、`sse-session-cookie.ts` 或 `tunnel-denial-log.ts` 之前,请阅读 [ARCHITECTURE.md](../ARCHITECTURE.md#dual-listener-tunnel-architecture-v1600) — 模块边界(没有从 `token-registry.ts` 到 `sse-session-cookie.ts` 的导入)对于范围隔离是承重的。

**服务器出口的 Unicode 清理** (v1.38.0.0+)。每个发送页面内容派生的字符串的服务器出口**必须**通过 `JSON.stringify(payload, sanitizeReplacer)`(用于对象负载)或 `sanitizeLoneSurrogates(body)`(用于文本正文)。否则来自 CDP 页面内容的孤立 UTF-16 代理项的一半作为 `\uD800` 样式转义到达 Anthropic API 并触发 400。今天在四个出口点连接:`handleCommandInternal`(HTTP + 批处理通过围绕 `handleCommandInternalImpl` 的清理包装器)和两个 SSE 生产者(`/activity/stream`、`/inspector/events`)。字符串化后的正则表达式是无操作 — `JSON.stringify` 已经在正则表达式可以匹配之前转义了代理项,因此替换器必须在编码管道内运行。在 `server.ts` 中添加新的 SSE/WebSocket 写入器或 HTTP 响应之前,请阅读 [ARCHITECTURE.md](../ARCHITECTURE.md#unicode-sanitization-at-server-egress-v13800)。`browse/test/server-sanitize-surrogates.test.ts` 用不变量测试固定连接,因此绕过会使 CI 失败。

**SSE 端点帮助器** (v1.51.0.0+)。`server.ts` 中的新 SSE 端点**必须**通过 `browse/src/sse-helpers.ts` 中的 `createSseEndpoint(req, config)` 进行路由。帮助器拥有清理合约(abort + enqueue-throw + heartbeat-throw,都是幂等的),并在每个 `JSON.stringify` 上烘焙 `sanitizeLoneSurrogates`,因此新订阅者不会意外地回归任一不变量。当 TCP 连接在没有触发 `req.signal.abort` 的情况下死亡(Chromium MV3 service-worker suspend、中间代理半关闭)时,内联 `ReadableStream` 连接泄漏订阅者。`/activity/stream`、`/inspector/events` 和 `/memory`(SSE-eligible)都通过它进行路由。`browse/test/sse-helpers.test.ts` 固定清理合约。

**CDP 会话生命周期** (v1.51.0.0+)。`browse/src/cdp-bridge.ts` 之外的直接 `page.context().newCDPSession(page)` 调用通过 `browse/test/cdp-session-cleanup.test.ts` 中的静态 grep tripwire 使 CI 失败。对一次性 CDP 工作使用 `withCdpSession(page, async (s) => {...})`(try/finally detach),对与页面生命周期相关的缓存会话使用 `getOrCreateCdpSession(page, cache)`(通过 `Map<page, session>` 进行 close-detach)。迁移了三个站点:cdp-bridge 框架事件、write-commands archive capture、cdp-inspector。帮助器防止每个会话的泄漏类,其中成功路径 detach 发生但错误路径 detach 被遗漏。

**设置符号链接硬化** (v1.38.0.0+)。`setup` 中的每个链接站点**必须**通过 `IS_WINDOWS` 检测附近的 `_link_or_copy SRC DST` 帮助器进行路由。在没有 Developer Mode 的 Windows 上,普通的 `ln -snf` 会产生在 `git pull` 上不刷新的冻结文件副本 — 每个主机适配器的静默陈旧性。帮助器在 Unix 上保留 `ln -snf`,在 Windows 上切换到 `cp -R` / `cp -f`。`test/setup-windows-fallback.test.ts` 强制执行静态不变量:帮助器正文之外的单个原始 `ln` 调用会使 CI 失败。Windows 用户从 `_print_windows_copy_note_once` 获得一行注释,提醒他们在每次 `git pull` 后重新运行 `./setup`。

**侧边栏安全堆栈** (针对 prompt 注入的分层防御):

| 层 | 模块 | 存在于 |
|-------|--------|----------|
| L1-L3 | `content-security.ts` | 服务器 + 读取路径 — 数据标记、隐藏元素剥离、ARIA 正则表达式、URL 黑名单、信封包装 |
| L4 | `security-classifier.ts`(TestSavantAI ONNX) | **仅安全 sidecar 子进程**(`security-sidecar-entry.ts`,由 `security-sidecar-client.ts` 从 server.ts 驱动) |
| Canary(金丝雀) | `security.ts`(generate/inject/detect) | 纯实用程序 — 今天没有生产注入器(注入它们的聊天提示构建器已被拆除) |
| Combiner(组合器) | `security.ts`(combineVerdict + THRESHOLDS) | 纯,已测试;为 LayerSignal 输入保留 transcript/deberta 投票处理,而任何实时层不再产生 |

历史说明:一个 L4b Haiku 脚本分类器和一个可选的 DeBERTa 集合(`GSTACK_SECURITY_ENSEMBLE=deberta`)一直存在到调用它们的聊天路径智能体被拆除;两者都作为死代码被删除(零生产调用者)。不要将它们重新记录为实时。

**关键约束:** `security-classifier.ts` 无法从已编译的 browse 二进制文件导入。`@huggingface/transformers` v4 需要 `onnxruntime-node`,它无法从 Bun compile 的临时提取目录中 `dlopen` — 因此采用 sidecar 子进程。只有 `security.ts`(纯字符串操作 — 金丝雀实用程序、verdict 组合器、状态)对于 `server.ts` 是安全的。参见 `~/.gstack/projects/garrytan-gstack/ceo-plans/2026-04-19-prompt-injection-guard.md` §"Pre-Impl Gate 1 Outcome" 获取原始架构决策。

**阈值** (在 `security.ts` 中):`BLOCK: 0.85`、`WARN: 0.75`、`LOG_ONLY: 0.40`、`SOLO_CONTENT_BLOCK: 0.92`(无标签的内容分类器无法区分"注入"和"针对用户的网络钓鱼",因此它们的单条门槛更高)。实时 L4 路径在 server.ts 的 sidecar-scan 处理中应用这些;金丝雀泄漏始终 BLOCK(确定性)。

**环境旋钮:**
- `GSTACK_SECURITY_OFF=1` — 紧急终止开关。即使已预热,分类器也保持关闭;L1-L3 过滤器继续运行。
- 分类器模型缓存:`~/.gstack/models/testsavant-small/`(112MB,仅首次运行)
- 攻击日志:`~/.gstack/security/attempts.jsonl` — 由 `tunnel-denial-log.ts` 写入(隧道表面拒绝;在 10MB 时轮换,5 代)

历史说明 (#2557):跨进程会话状态(`~/.gstack/security/session-state.json`)、`getStatus()`、`/health` `security` 字段和 sidepanel SEC 盾牌都已删除 — 当 sidebar-agent.ts 被拆除时,状态文件失去了其唯一的写入器,因此盾牌报告永久的 'inactive' 或来自剩余磁盘状态的过时假绿色 'protected'。实时防御(L1-L3 过滤器、注入扫描路径上的 L4 sidecar)通过它们自己的调用点报告,而不是通过 /health。`browse/test/extension-token.test.ts` 在真实的 /health body 上固定删除,而 `browse/test/pty-inject-scan.test.ts` 在行为上固定实时 L4 连接。不要将这些重新记录为实时。
