# 如何使用 GStack iOS 测试 iOS 应用

这是随 gstack 提供的 iOS QA 功能的端到端演练:将规范 Swift 模板安装到你的应用中,通过 USB 连接真实 iPhone,并从任何 agent(本地 Claude Code,或通过 Tailscale 的任何支持 HTTP 的 agent)驱动它。没有模拟器,没有 XCTest harness,没有 WebDriverAgent。

下面所有内容已端到端验证在运行 iOS 26.5 的真实 iPhone 17 Pro Max 上。同样的流程适用于任何运行 iOS 16+ 的设备。

## 你需要什么

- 安装了 Xcode 16.0+ 的 macOS(`xcrun devicectl --version` 必须成功)。Xcode 16 附带了 CoreDevice 隧道 `devicectl` 用于通过 USB 访问设备。
- 运行 iOS/iPadOS 16 或更高版本的真实 iPhone 或 iPad(iPad Stage Manager 和多窗口布局尚未经过设备验证)。已解锁,与你的 Mac 配对,在 Settings → Privacy & Security 中启用 **Developer Mode**。
- Apple 开发者团队 — 免费个人团队适用于实时设备调试部署。你需要团队 ID(例如 `623FYQ2M88`),而不是证书 ID。在 Xcode → Settings → Accounts → 你的 Apple ID → 团队列表中找到它。该设置在首次部署时通过 `-allowProvisioningUpdates -allowProvisioningDeviceRegistration` 为你的设备签署应用。
- 已安装 gstack(`./setup` 完成;`gstack-ios-qa-regen` 和 `gstack-ios-qa-daemon` 必须在 PATH 上)。
- PATH 上的 Bun 运行时(`bun --version`)。Mac 端守护进程是 bun 进程。

对于可选的远程 agent(Tailscale)模式,你还需要在 Mac 上安装 Tailscale,其中 `/var/run/tailscale.sock` 可读。

## 一口气的架构

```
┌─────────────────┐   tailnet (opt)    ┌──────────────────────┐   USB CoreDevice    ┌─────────────────────┐
│ Remote agent    │ ─────────────────▶ │ gstack-ios-qa-daemon │ ──────────────────▶ │ iOS app StateServer │
│ (Claude, GPT,   │  bearer + session  │  (Mac, bun/TS)       │  IPv6 ULA tunnel    │  (loopback only)    │
│  OpenClaw, ...) │                    │                      │                     │                     │
└─────────────────┘                    └──────────────────────┘                     └─────────────────────┘
```

- iOS 应用嵌入一个 `StateServer`(`DebugBridge` SPM 库,仅 `#if DEBUG`)监听 `::1` + `127.0.0.1` 端口 9999。Bearer token 保护。一次性的启动 token 存在于应用 `tmp/` 中的 0600 文件中(从不 `os_log`);守护进程将其复制出来并在约 5 秒内轮换,这会删除文件,因此稍后的复制是一个无效凭据。如果应用无法写入文件,它会记录 `NOT READY`,`/healthz` 携带 `boot_token_error`。
- Mac 守护进程通过 `xcrun devicectl` 在连接配对设备时自动打开的 CoreDevice IPv6 隧道代理流量。
- 在 Tailscale 模式下,守护进程公开一个绑定到你的 tailnet IP 的单独监听器,具有按会话 token 强制执行的能力 tier(observe / interact / mutate / restore)。Token 由 Mac 所有者通过 `gstack-ios-qa-mint` 显式铸造;远程调用者永远不会自动允许列表。

iOS `StateServer` 即使在远程模式下也始终是 loopback-only。身份验证发生在 Mac 端,因为 iPhone 无法验证 Tailscale 身份。

## 步骤 1:生成 DebugBridge 包

从应用根目录运行 `/ios-qa`,或直接调用相同的确定性重新生成器:

```bash
gstack-ios-qa-regen \
  --app-source "$PWD/Sources/YourApp" \
  --bridge-dir "$PWD/DebugBridge"
```

该命令将规范模板的显式允许列表复制到本地 `DebugBridge/` Swift 包,生成 `DebugBridgeGenerated/StateAccessor.swift`,并将安装的版本写入 `DebugBridgeGenerated/.gstack-version`。它将生成输出从其自己的模式哈希中排除,因此使用未更改的源重新运行是快速、字节稳定的缓存命中。它还从较旧的平面 harness 布局中删除显式的旧生成文件集,以便陈旧的桥接源无法影子该包。

1. 将 `DebugBridge/` 添加为本地包依赖。仅在 Debug 配置中依赖于 `DebugBridgeUI` 产品;`DebugBridgeCore` 和 `DebugBridgeTouch` 传递性地提供。
2. 将 `DebugBridgeGenerated/StateAccessor.swift` 添加到应用目标。
3. 在你的 `@main` 应用初始化中,在启动服务器之前安装 UIKit resolvers,然后注册生成的访问器。将示例状态/访问器名称替换为生成器找到的类型:

   ```swift
   #if DEBUG
   import DebugBridgeCore
   #if canImport(UIKit)
   import DebugBridgeUI
   #endif
   #endif

   // Inside App.init(), after appState is initialized:
   #if DEBUG
   #if canImport(UIKit)
   DebugBridgeUIWiring.installAll()
   #endif
   DebugBridgeManager.shared.start(
       appState: appState,
       register: AppStateAccessor.register
   )
   #endif
   ```

三个 SwiftPM 目标拆分如下:`DebugBridgeCore` 是跨平台的(因此 CI Mac 主机上的 `swift build` 可以在没有 UIKit 的情况下验证大部分代码),`DebugBridgeUI` 和 `DebugBridgeTouch` 是仅限 iOS 的(它们链接 UIKit)。`DebugBridgeTouch` 是 Objective-C — 它包含 KIF 派生的 UITouch 合成,具有 iOS 18+ `_UIHitTestContext` 修复,使 SwiftUI Button 点击真正触发。

结构化 Release 构建保护是分层的。`Package.swift` 中的 `.when(configuration: .debug)` 子句意味着 SwiftPM 拒绝在任何 Release 构建中链接任何 `DebugBridge*` 目标,因此即使你忘记清理,该桥也无法发布到 TestFlight。除此之外,`DebugBridgeTouch.m` 的私有 API 触摸合成在 `#if TARGET_OS_IOS && DEBUG` 后面编译(在 Debug 配置中仅通过 `Package.swift` 中的 `cSettings` 为该目标定义 `DEBUG`),因此触摸桥的 Release 编译会发出空的翻译单元 — 即使链接器保护被绕过,二进制文件中也没有私有符号。

## 步骤 2:构建 + 安装到设备

从应用的项目目录:

```
xcodebuild \
  -scheme YourAppScheme \
  -configuration Debug \
  -destination 'generic/platform=iOS' \
  -derivedDataPath /tmp/build \
  -allowProvisioningUpdates -allowProvisioningDeviceRegistration \
  CODE_SIGN_STYLE=Automatic \
  DEVELOPMENT_TEAM=YOUR_TEAM_ID \
  build
```

然后安装 + 启动:

```
UDID=$(xcrun devicectl list devices 2>/dev/null | awk 'NR>2 && $0!="" {print $(NF-2); exit}')
xcrun devicectl device install app --device "$UDID" /tmp/build/Build/Products/Debug-iphoneos/YourApp.app
xcrun devicectl device process launch --device "$UDID" --terminate-existing your.bundle.id
```

如果手机被锁定,你将收到 `FBSOpenApplicationServiceErrorDomain error 1 — Locked`。解锁并重试。第一次安装会在手机上显示 Trust 对话框;点击 Trust,然后重新运行。

## 步骤 3:启动 Mac 端守护进程

两种选择。

**选项 A — 让 skill 生成它。** 在 Claude Code 中的任何地方运行 `/ios-qa`;该 skill 按需生成守护进程,引导隧道,轮换启动 token,并通过代理公开设备。本地 USB 使用的最清晰路径。

**选项 B — 自己启动它。** 运行:

```
gstack-ios-qa-daemon
```

守护进程在两个 loopback 监听器都绑定后打印 `READY: port=<n> pid=<pid>`。默认端口是 9099。生成器可以使用约 5 秒超时读取该行以确认就绪;你也可以将 `curl` 指向打印的端口。

无论哪种方式,守护进程都会在 `~/.gstack/ios-qa-daemon.pid` 上获取独占 flock — 从两个 Claude Code 会话运行它是安全的;第二次调用发现运行守护进程的端口并加入。

设置这些环境变量以定位特定设备或包:

```
GSTACK_IOS_TARGET_UDID=248C3A58-B843-5BDB-8F5D-89ADB7D7BF6A
GSTACK_IOS_TARGET_BUNDLE_ID=com.yourorg.yourapp
GSTACK_IOS_DAEMON_PORT=9099       # loopback listener port; default 9099
```

如果 `GSTACK_IOS_TARGET_UDID` 未设置,守护进程会选择最佳的配对、可用的 iPhone 或 iPad,优先选择有线设备。当两个设备(一个 iPhone 和一个 iPad 都在 USB 上)打成平局时,它拒绝猜测:引导错误列出每个设备及其 UDID,并打印准备好的 `export GSTACK_IOS_TARGET_UDID=<udid>` 行。

守护进程在其生命周期内保持健康的轮换隧道。当 CoreDevice 路由断开(`503 device_disconnected` / `504 upstream_timeout`)时,它重新解析同一 UDID 的隧道地址,并使用它已持有的 bearer 探测正在运行的应用,因此应用保留其状态。仅当应用拒绝该 bearer(401,应用被重新启动)、应用未运行(一次正常启动)或现在选择了不同的设备时,它才会再次引导;旧的 bearer 永远不会发送到另一台设备。响应丢失的点击或其他变更永远不会重放。

如果新启动的守护进程到达已运行的目标,其一次性启动 token 已被早期守护进程消耗,则它会验证包所有者,强制重新启动该目标一次,等待新 token,再次验证所有权,并正常轮换。

## 步骤 4:驱动设备

守护进程运行后,你在 `http://127.0.0.1:9099`(或 `[::1]:9099`)处具有 HTTP 表面。Skill 流程为你执行此操作,但原始端点是:

| 端点 | 它做什么 | 身份验证 |
|---|---|---|
| `GET /healthz` | 版本探测。 | 无(loopback) |
| `POST /auth/rotate` | 仅限守护进程;将启动 token 轮换为仅内存值。 | 启动 token |
| `POST /session/acquire` | 获取每设备会话锁。返回 `{session_id, ttl_seconds}`。 | bearer |
| `POST /session/release` | 释放锁。 | bearer + session |
| `GET /screenshot` | 捕获活动窗口的 PNG。返回 `{png_base64: "..."}`。 | bearer |
| `GET /elements` | 可访问性树快照。 | bearer |
| `GET /state/snapshot` | 将每个 `// @Snapshotable` 字段转储为 JSON。 | bearer |
| `POST /state/restore` | 验证完整快照,然后在 MainActor 上恢复它。 | bearer + session, mutate tier |
| `POST /tap` `{x,y}` | 在窗口坐标合成真实的 UITouch。SwiftUI Buttons 触发。 | bearer + session, interact tier |
| `POST /swipe` `{from_x,from_y,to_x,to_y}` | 滚动最近的封闭 UIScrollView。 | bearer + session, interact tier |
| `POST /type` `{text}` | 在当前第一响应者上设置文本。 | bearer + session, interact tier |

变更请求需要 `Authorization: Bearer <token>` 头和 `X-Session-Id` 头。读取端点(`/screenshot`、`/elements`、`GET /state/*`)仅需要 bearer。

状态快照通过属性上方的独立生成器标记注释按字段选择加入。它故意不是属性包装器,因此它可以使用 Observation 干净地编译:

```swift
@Observable
final class AppState {
    // @Snapshotable
    var username: String = ""

    var authToken: String = "" // never exported
}
```

未标记的字段永远不会出现在快照中,这默认情况下将 token、PII 和身份验证状态排除在记录的 fixture 之外。标记字段必须是文件范围可观察类上的可写实例 `var`,具有显式类型和 internal 或 public setter。受支持的快照类型是 JSON-native 标量(`String`、`Bool`、有符号/无符号整数宽度、`Float`、`Double`、`CGFloat`)、数组、String 键控字典和这些类型的 Optional 组合。快照键在可观察类之间必须唯一。生成器在无效声明、自定义值、隐式解包 Optional、嵌套可观察类或重复键时报告并停止,而不是发出损坏或有损的 Swift。Restore 使用两个阶段:每个模型首先验证完整输入,然后才在 MainActor 上应用赋值。

## 步骤 5:使远程 agent 工作(可选)

要让另一台机器上的 agent 驱动设备,请使用 `--tailnet` 运行守护进程:

```
gstack-ios-qa-daemon --tailnet
```

守护进程首先探测 `/var/run/tailscale.sock`;如果套接字缺失或不可读,它会拒绝打开 tailnet 监听器(loopback 仍然运行)。远程模式永远不会半启动。

然后为应该能够连接的身份铸造一个会话 token:

```
gstack-ios-qa-mint grant --remote 'alice@example.com' --capability interact
gstack-ios-qa-mint grant --remote 'tag:ci' --capability mutate --ttl 86400 --note 'nightly'
gstack-ios-qa-mint list
```

能力 tier 是嵌套的:`observe`(仅读取端点) ⊂ `interact`(点击、滑动、输入) ⊂ `mutate`(`POST /state/*`) ⊂ `restore`(`POST /state/restore`)。选择完成工作所需的最小 tier。允许列表文件位于 `~/.gstack/ios-qa-allowlist.json`(模式 0600)— 守护进程在每个 `/auth/mint` 请求上读取它,因此更改立即生效,无需重新启动。

然后远程 agent 命中守护进程的 tailnet 监听器的 `POST /auth/mint`。守护进程通过 tailscaled 的 WhoIs 端点规范化调用者的身份,检查允许列表,并返回短期会话 token(1 小时默认,24 小时上限)。每个经过身份验证的变更请求都会记录在 `~/.gstack/security/ios-qa-audit.jsonl` 中;被拒绝的请求会记录在 `~/.gstack/security/attempts.jsonl` 中。

## 步骤 6:发布版本构建

在发布到 TestFlight 或 App Store 之前,请运行 `/ios-clean`。它删除 `DebugBridge` SPM 依赖项,并从你的 `@main` 应用中剥离 `#if DEBUG` 接线。`Package.swift` 中的结构化保护(`condition: .when(configuration: .debug)`)意味着 Release 构建即使你忘记清理也不会链接该桥,但 `/ios-clean` 为你提供了一个整洁的 diff 来审查和发布。

## 常见失败

| 症状 | 什么坏了 |
|---|---|
| `xcodebuild` 失败并显示 `Could not locate device support files for iOS X.Y` | 运行 `xcodebuild -downloadPlatform iOS` 以获取你的 iPhone 的 iOS 版本的设备支持包(约 8GB)。 |
| 安装成功,`process launch` 失败并显示 `Locked` | 手机被锁定。解锁并重试。 |
| 配对设备上的第一次安装失败,没有明确的错误 | 手机需要信任 Mac。在手机上打开 Settings → General → VPN & Device Management 并确认。 |
| `Developer Mode` 切换从 Settings → Privacy 中缺失 | 将设备连接到 Xcode → Window → Devices and Simulators 一次,或尝试针对它的任何 `devicectl device install`。iOS 将在第一次尝试后显示该切换。 |
| `xcrun devicectl device copy from` 返回 ERROR 7000 | 源路径错误 — 启动 token 存在于应用数据容器内的 `tmp/gstack-ios-qa.token`(NSTemporaryDirectory),而不是路径的根目录。 |
| Bootstrap 失败并显示 `multiple_devices` | 连接了多个 iPhone/iPad。运行打印的 `export GSTACK_IOS_TARGET_UDID=<udid>` 并重新启动守护进程。 |
| `boot_token_unavailable ... could not write tmp/gstack-ios-qa.token` | 应用无法写入其启动 token 文件。修复应用的 `tmp/` 目录并重新启动应用。 |
| iOS 26 上的 SwiftUI `DragGesture` 或 `Canvas` 忽略点击和滑动 | 进程内合成的触摸无法到达 iOS 26 上的 SwiftUI 手势识别器,并且 `/swipe` 仅滚动 `UIScrollView`。有关输入路由解决方法,请参阅 `/ios-qa` skill 中的"Known limits"。 |
| `/healthz` 返回 200 但 `/tap` 返回 ok:true 但没有 UI 更改 | 手机已配对,但 StateServer 端口可能在启动之间已更改。重新解析 CoreDevice IPv6(`dscacheutil -q host -a name '<DeviceName>.coredevice.local'`)。 |
| `/auth mint` 中的 `403 identity_not_allowed` | 远程调用者的身份不在 Mac 的允许列表中。在 Mac 上运行 `gstack-ios-qa-mint grant --remote <identity> --capability interact`。 |
| 守护进程不会打开 tailnet 监听器 | Tailscale 未安装,或 `/var/run/tailscale.sock` 不可读。修复 Tailscale,然后重新启动守护进程。Loopback 同时仍然运行。 |
| SwiftUI Button 点击返回 `ok:true` 但操作永远不触发 | 你使用的是 iOS 17 或更早版本,其中 `_UIHitTestContext` 不存在。DebugBridgeTouch 实现回退到普通的 `hitTest:`,它无法解析到 SwiftUI 的手势容器。将设备更新到 iOS 18+,或改用 UIKit 控件。 |

## 这给你带来什么

你可以用任何支持 HTTP 的语言编写 agent 循环。截取屏幕截图,询问模型该怎么做,发送点击。在前后捕获状态快照以为 `/ios-fix` 回归测试记录确定性 fixture。将同事添加到允许列表,他们可以通过 Tailscale 从他们的笔记本电脑驱动你的 iPhone,而无需触摸硬件。通过使用 mutate-tier 能力和 24 小时 TTL 铸造 `tag:ci` 会话 token,将同一守护进程插入 CI。

整个堆栈是你已经拥有的 Mac、你已经拥有的 iPhone、免费的 Apple 开发者帐户和 gstack。没有付费测试服务。没有模拟器漂移。用户看到的就是 agent 驱动的内容。
