# 远程浏览器访问 — 如何与 GStack Browser 配对

> **范围:** `/pair-agent` 共享 gstack 自己的浏览器引擎,这是当 Aside 浏览器不存在时(Linux、Windows、关闭的 Aside 应用程序)运行的**回退**。在打开了 Aside 的 Mac 上,每个 skill 都改为驱动 Aside,没有共享守护进程可以配对 — 驱动 Aside 的每个智能体都打开自己的标签页。参见 [BROWSER.md](../BROWSER.md) 了解何时触发回退。

GStack Browser 服务器可以与任何可以发出 HTTP 请求的 AI 智能体共享。智能体获得对真实 Chromium 浏览器的范围访问:导航页面、读取内容、点击元素、填写表单、截取屏幕截图。每个智能体都有自己的标签页。

本文档是远程智能体的参考。快速入门说明由 `$B pair-agent` 生成,其中包含实际的凭据。

## 架构

```
Your Machine                          Remote Agent
─────────────                         ────────────
GStack Browser Server                 Any AI agent
  ├── Chromium (Playwright)           (OpenClaw, Hermes, Codex, etc.)
  ├── Local listener  127.0.0.1:LOCAL         │
  │    (bootstrap, CLI, sidebar, cookies)      │
  ├── Tunnel listener 127.0.0.1:TUNNEL ◄───────┤
  │    (pair-agent only: /connect and          │
  │     /command — locked allowlist)           │
  ├── ngrok tunnel (forwards tunnel port only) │
  │     https://xxx.ngrok.dev ─────────────────┘
  └── Token Registry
        ├── Root token (local listener only)
        ├── Setup keys (5 min, one-time)
        ├── Session tokens (24h, scoped)
        └── SSE session cookies (30 min, stream-scope)
```

### 双监听器架构(v1.6.0.0)

守护进程绑定两个 HTTP 套接字。**本地监听器** 仅服务于 127.0.0.1 的完整命令表面,并且永远不会被转发。**隧道监听器** 在 `/tunnel/start` 上延迟绑定(并在 `/tunnel/stop` 上拆除),具有锁定的路径允许列表。ngrok 仅转发隧道端口。

意外发现你的 ngrok URL 的调用者无法到达 `/health`、`/cookie-picker`、`/inspector/*` 或 `/welcome` — 这些路径在该 TCP 套接字上不存在。通过隧道发送的根 token 会收到 403。隧道监听器仅接受 `/connect` 和 `/command`(具有范围化的 token + 26 命令浏览器驱动允许列表)。

有关完整端点表,请参见 [ARCHITECTURE.md](../ARCHITECTURE.md#dual-listener-tunnel-architecture-v1600)。

## 连接流程

1. **用户运行** `$B pair-agent`(或在 Claude Code 中 `/pair-agent`)
2. **服务器创建** 一个一次性设置密钥(5 分钟后过期)
3. **用户复制** 指令块到其他智能体的聊天中
4. **远程智能体运行** `POST /connect` 并提供设置密钥
5. **服务器返回** 一个范围化的会话 token(24 小时默认)
6. **远程智能体创建** 通过 `POST /command` 并提供 `newtab` 创建自己的标签页
7. **远程智能体浏览** 使用 `POST /command` 并提供其会话 token + tabId

## API 参考

### 身份验证

所有命令端点都需要 Bearer token:

```
Authorization: Bearer gsk_sess_...
```

`/connect` 是未经身份验证的(限速) — 这是远程智能体将设置密钥交换为范围化会话 token 的方式。`/health` 在本地监听器上是未经身份验证的(仅存活/状态 — 永远不是 token),但在隧道监听器上**不**存在(404)。扩展 token bootstrap 是本地监听器上的 `POST /extension-token`,由固定的 `chrome-extension://` Origin 控制;它也不在隧道表面上。

SSE 端点(`/activity/stream`、`/inspector/events`)接受 Bearer token 或 HttpOnly `gstack_sse` cookie(通过 `POST /sse-session` 铸造,30 分钟 TTL,仅流范围 — 不能用于 `/command`)。自 v1.6.0.0 起,不再接受 `?token=<ROOT>` 查询字符串身份验证。

### 端点

#### POST /connect
将设置密钥交换为会话 token。无需身份验证。限速为 300/分钟(泛洪防御 — 设置密钥是 24 个随机字节,不可暴力破解)。

```json
Request:  {"setup_key": "gsk_setup_..."}
Response: {"token": "gsk_sess_...", "expires": "ISO8601", "scopes": ["read","write","admin","meta"], "agent": "agent-name"}
```

#### POST /command
发送浏览器命令。需要 Bearer 身份验证。

```json
Request:  {"command": "goto", "args": ["https://example.com"], "tabId": 1}
Response: (plain text result of the command)
```

#### GET /health
服务器状态。无需身份验证。返回状态、标签页、模式、正常运行时间。
永远不携带 token — 扩展 token bootstrap 是 `POST /extension-token`(仅本地监听器,验证固定的 `chrome-extension://` Origin 和 loopback Host;否则 403)。无法通过隧道访问(404)。

### 命令

#### 导航
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `goto` | `["URL"]` | 导航到 URL |
| `back` | `[]` | 返回 |
| `forward` | `[]` | 前进 |
| `reload` | `[]` | 重新加载页面 |

#### 阅读内容
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `snapshot` | `["-i"]` | 带有 @ref 标签的交互式快照(最有用) |
| `text` | `[]` | 完整页面文本 |
| `html` | `["selector?"]` | 元素或完整页面的 HTML |
| `links` | `[]` | 页面上的所有链接 |
| `screenshot` | `["/tmp/s.png"]` | 截取屏幕截图 |
| `url` | `[]` | 当前 URL |

#### 交互
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `click` | `["@e3"]` | 点击元素(使用快照中的 @ref) |
| `fill` | `["@e5", "text"]` | 填写表单字段 |
| `select` | `["@e7", "option"]` | 选择下拉值 |
| `type` | `["text"]` | 输入文本(键盘) |
| `press` | `["Enter"]` | 按键 |
| `scroll` | `["down"]` | 滚动页面 |

#### 标签页
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `newtab` | `["URL?"]` | 创建新标签页(写入前需要) |
| `tabs` | `[]` | 列出所有标签页 |
| `closetab` | `["id?"]` | 关闭标签页 |

## Snapshot → @ref 模式

这是最强大的浏览模式。而不是编写 CSS 选择器:

1. 运行 `snapshot -i` 获取带有标记元素的交互式快照
2. 快照返回类似以下文本:
   ```
   [Page Title]
   @e1 [link] "Home"
   @e2 [button] "Sign In"
   @e3 [input] "Search..."
   ```
3. 在命令中直接使用 `@e` 引用:`click @e2`,`fill @e3 "search query"`

这就是快照系统的工作方式,它比猜测 CSS 选择器可靠得多。始终先 `snapshot -i`,然后使用引用。

## 范围

| 范围 | 它允许的内容 |
|-------|---------------|
| `read` | snapshot、text、html、links、screenshot、url、tabs、console 等 |
| `write` | goto、click、fill、scroll、newtab、closetab 等 |
| `admin` | eval、js、cookies、storage、cookie-import、useragent 等 |
| `meta` | tab、diff、frame、responsive、watch |
| `control` | stop、restart、disconnect、state、handoff — 浏览器范围的破坏性操作 |

配对的智能体默认获得 `read+write+admin+meta`;配对仪式是信任边界。`--restrict` 缩小列表(它永远不能授予 `control`)。`--control` 添加 control 范围(`--admin` 是传统别名)。通过隧道,`js`/`cookies`/`storage` 命令无论如何都被命令允许列表阻止;`eval` 起作用。当智能体将读取不受信任的 Web 内容时,使用 `--restrict "read,write"` 配对 — 范围限制了 prompt 注入的爆炸半径。

要收紧已经配对的智能体,请使用**相同的 `--client` 名称** 和较窄的 `--restrict`/`--domain` 重新配对:减少的重新配对会撤销先前的会话并立即释放其标签页(智能体必须使用新密钥重新连接),因此旧的广泛访问永远不会徘徊。扩大或刷新保持工作会话没有中断。没有 `--client` 的重新配对会铸造一个新的智能体。`root` 是保留的客户端名称。

## 标签页隔离

每个智能体拥有它创建的标签页。规则:
- **读取:** 任何智能体都可以读取任何标签页(snapshot、text、screenshot)
- **写入:** 只有标签页所有者可以写入(click、fill、goto 等)
- **未拥有的标签页:** 预先存在的标签页仅对 root 写入可用
- **第一步:** 在尝试交互之前始终 `newtab`

## 错误代码

| 代码 | 含义 | 该怎么办 |
|------|---------|------------|
| 401 | Token 无效、过期或被撤销 | 要求用户再次运行 /pair-agent |
| 403 | 命令不在范围内、标签页不是你的,或者不在隧道允许列表上 | 使用 newtab;用户可以在没有 `--restrict` 或使用 `--control` 的情况下重新配对 |
| 429 | 速率限制超过(>10 req/s) | 等待 Retry-After 标头 |

## 安全模型

- **物理端口分离。** 本地监听器和隧道监听器是单独的 TCP 套接字。ngrok 仅转发隧道端口。隧道调用者根本无法到达引导端点(404,错误的端口)。
- **隧道命令允许列表。** 隧道上的 `/command` 仅接受 26 个浏览器驱动命令(goto、click、fill、snapshot、text、newtab、tabs、back、forward、reload、closetab 等)。服务器管理命令(tunnel、pair、token、useragent、js)在隧道上被拒绝。
- **根 token 被隧道阻止。** 通过隧道监听器携带根 token 的请求返回 403 并带有配对提示。只有范围化的会话 token 可以通过隧道工作。
- **设置密钥** 在 5 分钟后过期,并且只能使用一次。
- **会话 token** 在 24 小时后过期(可配置)。
- 根 token 永远不会出现在指令块或连接字符串中。
- **Control 范围**(stop/restart/disconnect)默认被拒绝,并且永远不会通过范围列表出现。Admin 在配对时被授予;`js`/`cookies`/`storage` 无论如何都被命令允许列表阻止通过隧道。对于不太受信任的智能体,请使用 `--restrict`。
- Token 可以立即撤销:`$B tunnel revoke agent-name` 删除会话以及任何挂起的设置密钥,并根据实时智能体列表进行验证。`$B tunnel agents` 显示谁已配对(包括挂起的设置密钥)。`$B stop` 清除所有内容 — token 永远不会在守护进程重启后幸存。
- **SSE 身份验证** 使用 30 分钟的 HttpOnly SameSite=Strict cookie,仅流范围(永远不能用于 `/command`)。
- **`/welcome` 上的路径遍历保护** — `GSTACK_SLUG` 必须匹配 `^[a-z0-9_-]+$` 或回退到内置模板。
- **goto、download 和 scrape 路径上的 SSRF 防护** — 根据 localhost/私有范围黑名单验证 URL 目标。
- **隧道表面拒绝日志记录。** 隧道监听器上的每个拒绝(`path_not_on_tunnel`、`root_token_on_tunnel`、`missing_scoped_token`、`disallowed_command:*`)都会附加到 `~/.gstack/security/attempts.jsonl`,并带有时间戳、源 IP、路径、方法。速率上限为 60 次写入/分钟。
- **隧道启动时的出口收据(v1.63+)**。每个隧道会话打开都会将哈希链收据(sink `browse-tunnel`)写入 `~/.gstack/security/egress.jsonl`,**在** ngrok 转发任何内容**之前**。失败关闭:如果无法写入收据,则隧道拒绝启动。使用 `bin/gstack-egress list` / `bin/gstack-egress verify` 进行审计。
- 所有智能体活动都使用归因(clientId)记录。

**已知的非目标(跟踪为 #1136):** 在 Windows 上,cookie-import-browser 路径使用 `--remote-debugging-port=<random>` 启动 Chrome。对于 App-Bound Encryption v20,同一用户的本地进程可以连接到该端口并提取解密的 v20 cookie — 相对于直接读取 SQLite DB 的提升路径。修复方向是 `--remote-debugging-pipe` 而不是 TCP。

## 同机快捷方式

如果两个智能体在同一台机器上,请跳过复制粘贴:

```bash
$B pair-agent --local openclaw    # writes to ~/.openclaw/skills/gstack/browse-remote.json
$B pair-agent --local codex       # writes to ~/.codex/skills/gstack/browse-remote.json
$B pair-agent --local cursor      # writes to ~/.cursor/skills/gstack/browse-remote.json
```

不需要隧道。直接使用 localhost。

## ngrok 隧道设置

对于不同机器上的远程智能体:

1. 在 [ngrok.com](https://ngrok.com) 注册(免费套餐可用)
2. 从仪表板复制你的身份验证 token
3. 保存:`echo 'NGROK_AUTHTOKEN=your_token' > ~/.gstack/ngrok.env`
4. 可选地声明稳定域:`echo 'NGROK_DOMAIN=your-name.ngrok-free.dev' >> ~/.gstack/ngrok.env`
5. 使用隧道启动:`BROWSE_TUNNEL=1 $B restart`
6. 运行 `$B pair-agent` — 它将自动使用隧道 URL
