# iOS QA 守护进程的 Tailscale ACL 示例

仅当您传递 `--tailnet` 时,Mac 端守护进程才绑定 Tailscale 接口。默认情况下,守护进程仅本地 USB。本文档逐步介绍将您的 iPhone 安全地暴露给远程代理的步骤,以便它们可以通过 tailnet 运行 iOS QA。

## 威胁模型回顾

- **iOS 应用 StateServer:** 始终仅环回。可通过 CoreDevice IPv6 隧道从 Mac 访问。永远不要直接绑定到 tailnet。
- **Mac 守护进程:** 拥有 tailnet 接口。绑定两个侦听器 — 环回(完整表面,永不转发)和 tailnet(具有能力层级的锁定允许列表)。
- **身份验证:** 通过本地 `tailscaled` 套接字(`/var/run/tailscale.sock` LocalAPI WhoIs)进行 Tailscale 身份验证。允许列表文件位于 `~/.gstack/ios-qa-allowlist.json` 是谁可以做什么的单一真相来源。

## Step 1: 安装并运行 Tailscale

```bash
brew install --cask tailscale
# 登录 + 启动 tailscaled,然后验证:
tailscale status
```

确认守护进程可以读取 LocalAPI 套接字:

```bash
test -S /var/run/tailscale.sock && echo "socket present" || echo "MISSING"
```

如果缺失,守护进程将拒绝打开 tailnet 侦听器(失败关闭)。

## Step 2: 设置守护进程的 ACL

守护进程需要知道哪些 Tailscale 身份被允许以哪种能力层级控制哪些设备。允许列表文件是 JSON:

```json
{
  "version": 1,
  "entries": [
    {
      "identity": "you@example.com",
      "capabilities": ["restore"],
      "expires_at": null,
      "note": "Owner — full access"
    },
    {
      "identity": "ci@example.com",
      "capabilities": ["mutate"],
      "expires_at": "2026-12-31T00:00:00Z",
      "note": "CI runner — can write state but not full restore"
    },
    {
      "identity": "tag:claude-readonly",
      "capabilities": ["observe"],
      "expires_at": null,
      "note": "Agents that should only read"
    }
  ]
}
```

身份通过 WhoIs 进行规范化:

- **用户 OAuth:** `user@example.com`(无 `acct:`,无域重写)。
- **标记节点:** `tag:<tagname>`(小写)。
- **节点密钥:** `node:<nodekey-hex>`(很少见;改用标记)。

能力层级有序: `observe` < `interact` < `mutate` < `restore`。
授予 `restore` 意味着所有较低的层级。

## Step 3: 为远程代理铸造会话令牌

您可以让代理自发现(如果他们的身份在允许列表中),或者您可以在服务器端为他们铸造:

```bash
# 服务器端铸造(仅所有者,在带设备的 Mac 上本地运行):
gstack-ios-qa-mint --remote ci@example.com --capability mutate --ttl 1h

# 自助铸造(通过 tailnet 的代理):
curl -X POST http://<mac-tailnet-ip>:9999/auth/mint \
  -H "Content-Type: application/json" \
  -d '{"capability": "interact"}'
# → {"session_token": "...", "expires_at": "...", "capability": "interact"}
```

## Step 4: 加强 Tailscale ACL(深度防御)

守护进程的允许列表是主要的访问控制。双重保护: 限制 tailnet ACL 以限制谁可以甚至*到达*守护进程端口。

```jsonc
// 在您的 Tailscale 管理控制台中:
{
  "acls": [
    // 允许 CI 运行程序仅在端口 9999 上访问 iOS QA Mac。
    {
      "action": "accept",
      "src": ["ci@example.com"],
      "dst": ["ios-qa-mac:9999"]
    },
    // 标记的 Claude 代理 — 仅观察层级(由守护进程强制,而不是 ACL)。
    {
      "action": "accept",
      "src": ["tag:claude-readonly"],
      "dst": ["ios-qa-mac:9999"]
    },
    // 默认拒绝。
    {
      "action": "drop",
      "src": ["*"],
      "dst": ["ios-qa-mac:9999"]
    }
  ]
}
```

## Step 5: 审计跟踪

通过 tailnet 侦听器的每个经过身份验证的变异请求都会向 `~/.gstack/security/ios-qa-audit.jsonl` 写入一行:

```jsonl
{"ts":"2026-05-18T14:23:00Z","identity":"ci@example.com","device_udid":"00008101-XXXX","endpoint":"/tap","session_id":"abc...","capability":"interact","request_id":"req_001","status":200}
```

拒绝(无令牌、过期令牌、能力不足、身份未在允许列表中、达到速率限制)会写入 `~/.gstack/security/attempts.jsonl`。

## 速率限制

- `/auth/mint`: 每个身份 60 秒内 10 个铸造。第 11 个返回 429。
- 每个 tailnet 请求正文: 1MB 硬上限(以上为 413)。
- 截图响应: 10MB 硬上限(以上为带清理错误消息的 500)。

## 令牌生命周期

- 守护进程铸造的会话令牌: 默认 1 小时 TTL,通过 `--tailnet-session-ttl` 最长 24 小时。
- 可通过 `POST /session/heartbeat` 刷新(以 `ttl_seconds` 扩展,上限为原始最大值)。
- 引导令牌(iOS 应用启动和守护进程轮换之间): ~5 秒生命周期 — 守护进程在第一次抓取时立即轮换。

## 失败模式

| 症状 | 原因 | 操作 |
|---|---|---|
| 守护进程拒绝打开 tailnet 侦听器 | `/var/run/tailscale.sock` 缺失或权限被拒绝 | 安装 Tailscale;验证 `tailscale status` 作为运行守护进程的用户工作 |
| `403 identity_not_allowed` | 身份未在允许列表中 | 所有者铸造: `gstack-ios-qa-mint --remote <identity>` |
| `403 capability_insufficient` | 令牌层级低于端点要求 | 所有者使用更高 `--capability` 层级铸造 |
| `429 rate_limited` | 一个身份每分钟超过 10 次铸造 | 等待 60 秒;调查代理为何如此频繁地重新铸造 |
| `409 schema_mismatch` 在 `/state/restore` 上 | 来自旧应用构建的快照 | 丢弃快照;从当前应用构建重新捕获 |