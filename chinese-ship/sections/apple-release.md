<!-- AUTO-GENERATED from apple-release.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
# Apple App Store 发布

<!-- Ported from time-attack/gstack (GStack 2) APPLE-RELEASE.md, refined
     across 21 live App Store releases. Portions copyright (c) 2026
     Sina Matian, time-attack/gstack, MIT. Adapted to gstack main's
     architecture: decision-store bins, the Third-Party Web Actions
     contract, and an idempotency log in place of the fork's durable-state
     wrapper. All credential examples are placeholders, never live formats. -->

适用于 ship 目标是 Apple 平台应用的情况:仓库包含 `.xcodeproj` 或 `.xcworkspace`,或带有 app 产品的 Swift 包。在 Step 1 的分支闸门和任何预检之前阅读此文档。该适配器将 ship 的判断扩展到 App Store 旅程的端到端,并且每次上传或提交都被视为持久的外部效应(下面的幂等性合约);但商店分发是其自身的发布路径,不是仓库着陆:ship 的分支/PR 仪式(特性分支闸门、提交审查 PR 流水线、合并队列)仅在用户要求仓库变更时适用。向 App Store 或 TestFlight 发布的用户从他们所在的任何分支通过此适配器继续;在基础分支上的干净工作树是归档和上传的正常有效状态。永远不要因为分支拓扑而中止 App Store 发布。

**文档预检:** 在归档/上传/提交之前,阅读 `~/.claude/skills/gstack/ship/sections/documentation.md` 并以 `read-only` 模式对所选发布源执行其审计闸门,包括在基础分支上。本次审计不授权源编辑、变更状态的 Git 操作或变更仓库着陆仪式。解决阻塞或获得明确的命名文档风险异常,然后再进行分发;在最终商店发布报告中包含 `updated`/`current`/`blocked` 和任何异常。此指针不会改变下面的任何 App Store 权限。

一个工具运行整个发布:机器级 fastlane,包括 `produce`(app 记录和 bundle ID)、`cert` 和 `sigh`(签名)、`gym`(归档和签名导出)、`pilot`(TestFlight)、`deliver`(元数据、截图、提交审核)、`frameit`(设备边框)。缺失时安装它(`brew install fastlane`)并附上一行公告,而非一次性提问;发布授权涵盖机器工具安装。永远不要安装额外的 App Store CLI 工具,并且在运行期间永远不要向用户提及 API 密钥、`.p8` 文件、会话或任何凭证格式;唯一的例外是关闭报告中的一行常驻凭证披露(请参阅 Storefront 完成),该披露命名了发布创建的持久化密钥以便稍后可以稍后撤销它;他们支付了 99 美元,想要发布,并且发布本身不会向用户的项目添加任何新的依赖项。

仅 Mac 才需要构建腿。归档、签名和二进制上传搭载 Xcode 的仅限 macOS 的工具链;Apple 不在其他地方提供它,也没有工具绕过它。在非 macOS 主机上,如实说明,然后通过 macOS CI 运行器完全路由这些腿(执行相同 `gym` 和 `deliver`/`pilot` 命令的 GitHub Actions `macos` 运行器,使用铸造的上传密钥作为 CI 秘密提供;密钥身份验证正是 CI 所需要的);登录、密钥铸造、元数据、截图、定价和提交判断仍然是留在用户机器上的纯 API 工作。永远不要声称整个发布在 Mac 之外是不可能的,也永远不要假装构建腿在那里是可能的。

## 一次授权时刻

为两次常规互动提供计划。真正的阻塞可能需要安全或命名文档风险决策;为该决策停止,而不是将发布授权视为豁免。

首先,事先:确认用户持有付费的 Apple Developer Program 会员资格(每年 99 美元,App Store 和 TestFlight 都需要),并授权发布。

定价属于同一次呼气,每个应用仅有一次:在授权问题内询问免费还是付费(如果是付费则询问价格),而不是向匿名单独的中断。在检查决策存储(`bin/gstack-decision-search --scope repo --query "pricing"`)之后;持续存储答案(`~/.claude/skills/gstack/bin/gstack-decision-log`,作用域 `repo`)以便以后的发布不再询问,并且付费答案如实命名一次性的付费应用银行/税务协议,因为在签署之前没有任何东西可以销售。价格是代理永远不会默默默认的发布决策:免费发布无法取消发布。

Apple 登录发生在同一时刻:通过主机的交互式命令路径运行 `fastlane spaceauth -u <apple-id>`(在 Claude Code 中,用户键入 `! fastlane spaceauth -u <email>`,以便他们的密码和一个两因素码直接进入会话内的 Apple;仅当主机没有交互式路径时,单独的终端窗口作为回退)。将打印的会话令牌保留在会话之外;`~/.fastlane/spaceship/` 中的缓存 cookie 是 fastlane 实际使用的凭证;永远不要存储、回显或记录密码或令牌,并且当会话过期时重新运行相同的单一命令。

在第一次登录之后,立即从会话铸造永久的上传密钥(归档和上传的步骤 4),当该密钥已经位于 `~/.gstack/apple/api-key.json` 并且不需要新的应用记录时,完全跳过登录:重复发布授权并继续零登录。

其次,仅当预检发现图标或截图缺失时:下面的商店资产问题。

阻塞清除后,工具安装、上传、店面和提交由授权涵盖,无需询问。身份验证菜单、工具选择问题、计划确认和逐步叙述请求是合约违规。

无会员资格:停止 App Store 路径。提供通过第三方 Web 操作合约(此技能中的早期)走过 developer.apple.com 的注册(用户自己完成购买;活动可能需要一两天),并如实命名免费账户上限:仅在用户自己的设备上安装个人团队,7 天后过期,无 TestFlight,无 App Store。

## 发布预检

在归档之前解决并验证。防止打印的更改边界授权;将其他所有内容报告为阻塞发现。

- 签名:应用目标上的开发团队;当不存在时,`cert` 和 `sigh` 铸造分发证书和 App Store 配置文件。
- 版本控制:用户应该看到的营销版本和严格大于已为该版本上传的任何版本的内部版本号。
- 依赖项: `xcodebuild -resolvePackageDependencies` 成功;如果存在 `Podfile` 或 `Cartfile`,则其安装步骤已运行且锁文件是最新的。
- App Store 验证阻塞器:完整的应用图标集,包括 1024pt 营销图标、启动屏幕、应用触及的每个隐私门控 API 的使用描述字符串、必需的隐私清单、导出合规答案(`ITSAppUsesNonExemptEncryption`),以及合理的部署目标。

## 商店资产

仅当预检发现图标或截图缺失时,询问一次,旅程中的第二次也是最后一次允许的问题,然后在没有进一步提示的情况下采取行动。每个应用仅有一次:在询问之前,检查决策存储(`bin/gstack-decision-search --scope repo --query "store assets"`);已确定的选择(包括 "defer screenshots" 或 "TestFlight only")静默应用,从不重新询问。在用户回答后,持久化它(`~/.claude/skills/gstack/bin/gstack-decision-log`,作用域为 `repo`),以便未来的运行不再询问;用户通过说出来来更改它,而不是通过被重新提示。提供:

- **应用图标**: SnapAI(`npx snapai`,应用图标代理技能)使用用户自己的图像生成密钥生成单个 1024×1024;Xcode 15+ 从该单个图像派生每个大小。
- **营销截图,免费且本地,无 API 密钥**:应用商店截图卡组编辑器技能,搭建脚手架,用模拟器捕获和优势标题预填其卡组 JSON,并导出一个覆盖每个所需 iPhone 尺寸的捆绑包(导出可无头自动)。营销级不要求图像后端;在此技能已安装时永远不要声称截图需要 API 密钥。
- **纯边框,免费且本地**:在模拟器中捕获构建的应用,并使用 fastlane `frameit` 为其加边框;当不想要设计好的卡组时的最小选项。
- **AI 增强的营销截图**:aso-appstore-screenshots 代理技能(优势标题、突围面板、精确的 App Store 尺寸),唯一需要用户自己的图像生成密钥的选项;当安装时,遵循其工作流而不是重新实现它。
- **用户提供的文件**:始终是有效答案;验证尺寸并继续。

最终,在创建商店资产时根据应用商店截图卡组编辑器技能的安装情况进行:当该技能已安装时,其免费无密钥选项必须出现在列表中;省略它等于声称截图需要 API 密钥,这是合约违规。已经存在的资产完全跳过此问题。在退出时声明当时所有生成的内容。

## 归档和上传

1. 使用 `gym` 归档和导出签名的 Release 构建(它驱动预检中铸造的 xcodebuild 和签名)。具有自定义归档要求的项目可以直接使用 `xcodebuild archive`;无论如何,输出都是 App Store 签名的 `.ipa`。
2. 上传是外部效应:在持久化效应合约下运行 `pilot`(TestFlight)或 `deliver`(App Store):在执行之前,将键 `appstore.upload.<bundle-id>.<build>` 追加到 `$GSTACK_STATE_ROOT/projects/$SLUG/apple-effects.log`(在读取或追加它的同一命令中解析状态根和 slug);如果键已经由于先前的运行(崩溃、重试)而存在,则视为可能已完成,请勿重新运行。在模糊性上永远不要重新上传;首先检查 App Store Connect 的构建。
3. 缓存的会话、铸造的访问密钥和每个密钥文件都是环境或文件级别的机密:永远不要 argv、永远不要回显、永远不要提交。
4. 永远不要要求应用特定密码;会话改为铸造上传密钥。

   根据 fastlane 的文档身份验证,Apple 的二进制上传工具(iTMSTransporter,`deliver`/`pilot` 为 `.ipa` 调出它)不接受 Web 会话:它仅采用 App Store Connect API 密钥或应用特定密码,并且 Apple 的错误 `-22938`("使用应用特定密码登录")正是 Transporter 所说的。

   那不是闸门也不是问题,因为 Web 会话本身会静默创建密钥:通过 fastlane 的捆绑 spaceship(`Spaceship::Tunes.login(<apple-id>)` 重用缓存的 cookie,然后是原始客户端请求),`POST https://appstoreconnect.apple.com/iris/v1/apiKeys` 带有 JSON:API 正文,作用域为正在发布的应用,而不是所有应用: `{data:{type:"apiKeys",attributes:{nickname:"gstack-upload",allAppsVisible:false,roles:["APP_MANAGER"],keyType:"PUBLIC_API"},relationships:{apps:{data:[{type:"apps",id:"<asc-app-id>"}]}}}}`,其中 `<asc-app-id>` 是 App Store Connect 应用 id(从 `produce` 的输出,或 `GET https://appstoreconnect.apple.com/iris/v1/apps?filter[bundleId]=<bundle-id>`)。

   `allAppsVisible:false` 与显式 `apps` 关系是故意最小权限;`allAppsVisible:true` APP_MANAGER 密钥对团队中的每个应用具有常设权限,如果机器稍后受到威胁,则不必要的爆炸半径。`apps` 关系是必需的,不是可选的:没有应用关联的密钥看不到任何东西,上传因权限错误而失败,因此将其作用域限定为目标应用,而不是仅翻转标志位。

   仅在应用记录存在之后才铸造它(因此当应用为新应用时首先运行 `produce`)。

   然后 `GET .../iris/v1/apiKeys/<id>?fields[apiKeys]=privateKey`,`privateKey` 属性是完整 PEM 文件的 base64:精确解码一次并立即写入 `~/.appstoreconnect/private_keys/AuthKey_<id>.p8`(0600),它只能在创建时下载。颁发者 ID 是来自 `GET https://appstoreconnect.apple.com/olympus/v1/session` 的 `provider.publicProviderId`。

   将密钥 id、颁发者 id 和密钥内容作为 fastlane api-key JSON 记录到 `~/.gstack/apple/api-key.json`(0600)并从那时起使用 `api_key_path` 运行 `deliver`/`pilot`。

   该密钥永不过期,因此同一应用的每个后续发布都跳过登录;发布不同的应用将该应用重新关联到该密钥(`PATCH .../iris/v1/apiKeys/<id>` 将其添加到 `apps` 关系)或铸造新的应用作用域密钥,因为该密钥故意不是所有应用的。会话仅在 `produce`(Apple 的公共 API 无法创建应用记录)、该重新关联以及如果密钥被撤销则重新铸造时才需要。

   在密钥铸造未经尝试时声明用户必须自己生成任何凭证是合约违规。

   在触摸凭证之前对错误进行分类:错误仅为身份验证失败当且仅当它这么说(401/403、会话无效或过期、"登录"、"应用特定密码"在 Apple 自己的话中)。`Spaceship::UnexpectedResponse`、缺失/无效属性、验证或预检查错误是元数据问题;修复负载(例如,Apple 在 `app_rating_config.json` 中扩展的年龄分级属性,如 `lootBox`、`ageAssurance`、`parentalControls`、`messagingAndChat`),然后从 CLI 重试。将元数据错误视为凭证问题是合约违规。
5. 在 Apple 发布中,此适配器覆盖第三方 Web 操作合约(此技能中的早期):通用代理浏览器提供从不适用于此处的 App Store 或凭证工作。整个发布是 CLI(fastlane)加上上面的常规交互和阻塞决策;此适配器永远允许的唯一浏览器使用是本文档末尾命名的付费应用协议/银行/税务残余。为该旅程中的任何其他事情打开浏览器,由人驱动或手动,都是合约违规。

   当真实错误确实强制回退时,逐字引用该文本,然后按以下顺序升级:首先根据步骤 4 从会话铸造(或重新铸造)上传密钥,然后使用 `api_key_path` 重试上传;没有磁盘上的密钥的上传身份验证错误意味着跳过了铸造,而非用户欠下凭证。

   其次,如果铸造本身因会话错误而失败,请用户再次登录(与原始授权相同的 `! fastlane spaceauth -u <apple-id>` 时刻),重新铸造,然后重试。

   仅当全新会话仍无法铸造密钥,因为已登录的 Apple ID 不是团队上的 Admin 或功能持有者的权限拒绝,应用特定密码路径才打开,并且其唯一形状是自助服务:用户在任意设备上生成密码,并通过主机的会话内屏蔽提示加入卷密钥库(`fastlane fastlane-credentials add --username <apple-id>`),然后重试上传。

   永远不要提供或推荐浏览器驱动来创建凭据;任何种类的任何代理浏览器,任何密码、密钥或令牌,在任何框架下。
6. App Review 联系方式(姓名、电子邮件、电话)是提交的必需元数据:从已登录的 Apple ID 和 git 配置推断姓名和 email,在授权问题内收集电话号一次,持久化到决策存储,并且永远不要再次询问。联系方式是元数据,而不是要在运行中宣布的阻塞闸门。

## Storefront 完成

`produce` 已在运行期间创建应用记录和 bundle ID;永远不要将应用记录称为手动闸门。通过 App Store Connect 价格计划端点(`POST /v1/appPriceSchedules` 通过会话或铸造的密钥)应用在授权问题中确定的定价:fastlane 的 `price_tier` 选项针对当前 API 已损坏("'prices' is not a relationship on 'apps'"),因此永远不要通过它路由定价或将它的失败称为账户问题。`deliver` 拥有商店列表所需的其他所有内容:描述、关键字、本地化、每个设备尺寸的截图上传、附加上传的构建以及提交审核;当用户要求时,`pilot` 将 TestFlight 组和测试人员管理为中间一轮。提交遵循与键 `appstore.submit.<bundle-id>.<version>` 相同的持久化效应合约;在模糊性上,在重新运行之前检查 App Store Connect。之后从 CLI 监控评审状态。

始终仅有 Web 的操作:付费的 Apple Developer 程序成员资格购买本身(先决条件,而不是发布步骤)以及仅对于付费应用,一次性的付费应用协议与银行和税务;在任何手动检查表之前,通过第三方 Web 操作合约(此技能中的早期)提供代理浏览器驱动。免费应用在任何一个时刻都不需要浏览器。提交后,报告 App Review 通常在一两天内回答并关闭运行;评审结果不是此工作流可以保持打开的闸门。在同一关闭报告中,披露发布创建的持久凭证,一行,每次运行一次: "This created an App Store Connect API key (`gstack-upload`, scoped to this app) that persists for future releases; revoke it anytime at App Store Connect → Users and Access → Integrations, or delete `~/.gstack/apple/api-key.json` locally." 这是此适配器开头中运行中无凭证谈话规则的有意例外:用户否则永远不会被告知其账户上和磁盘上现在存在常驻凭证,因此它永远不会达到他们的撤销检查表。在退出时披露,而不是运行中的问题,以便一次授权时刻合约成立。