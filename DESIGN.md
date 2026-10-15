---
# gstack: design-md-format=spec
name: gstack
typography:
  display:
    fontFamily: Satoshi
  body:
    fontFamily: DM Sans
  label:
    fontFamily: DM Sans
  mono:
    fontFamily: JetBrains Mono
    fontFeature: tnum
colors:
  primary-dark-mode: "#F59E0B"
  primary-light-mode: "#D97706"
  primary-text-accent-dark-mode: "#FBBF24"
  primary-text-accent-light-mode: "#B45309"
  zinc-50: "#FAFAFA"
  zinc-400: "#A1A1AA"
  zinc-600: "#52525B"
  zinc-800: "#27272A"
  surface-dark: "#141414"
  base-dark: "#0C0C0C"
  surface-light: "#FFFFFF"
  base-light: "#FAFAF9"
  border-dark: "#262626"
  border-light: "#E7E5E4"
  success: "#22C55E"
  warning: "#F59E0B"
  error: "#EF4444"
  info: "#3B82F6"
spacing:
  2xs: 2px
  xs: 4px
  sm: 8px
  md: 16px
  lg: 24px
  xl: 32px
  2xl: 48px
  3xl: 64px
rounded:
  sm: 4px
  md: 8px
  lg: 12px
  full: 9999px
---

# Design System — gstack

## 总览

- **这是什么:** gstack 的社区网站——一个把 Claude Code 变成虚拟工程团队的 CLI 工具
- **为谁服务:** 正在发现 gstack 的开发者、已有社区成员
- **领域/行业:** 开发者工具(对标:Linear、Raycast、Warp、Zed)
- **项目类型:** 社区仪表板 + 营销站点

- **方向:** 工业/实用主义——功能优先、数据密集、等宽字体作为个性字体
- **装饰程度:** 刻意的——在表面施加细微的噪点/颗粒纹理以增加材质感
- **气质:** 严肃的工具,由真正关心工艺的人打造。温暖,而非冷漠。CLI 血统本身就是品牌。
- **参考站点:** formulae.brew.sh(竞品,但我们的是活的、可交互的)、Linear(暗色 + 克制)、Warp(暖色点缀)

## 颜色

- **取向:** 克制——琥珀色强调色稀有且有意义。仪表板数据获得颜色,外壳保持中性。
- **Primary(暗色模式):** amber-500 #F59E0B——温暖、有活力,读起来像「终端光标」
- **Primary(亮色模式):** amber-600 #D97706——在白底上加深以提升对比度
- **Primary text accent(暗色模式):** amber-400 #FBBF24
- **Primary text accent(亮色模式):** amber-700 #B45309
- **中性色:** 冷调 zinc 灰
  - zinc-50: #FAFAFA(最浅)
  - zinc-400: #A1A1AA
  - zinc-600: #52525B
  - zinc-800: #27272A
  - Surface(暗): #141414
  - Base(暗): #0C0C0C
  - Surface(亮): #FFFFFF
  - Base(亮): #FAFAF9
- **语义色:** success #22C55E,warning #F59E0B,error #EF4444,info #3B82F6
- **暗色模式:** 默认。近黑底色(#0C0C0C),表面卡片在 #141414,边框在 #262626。
- **亮色模式:** 暖石底色(#FAFAF9),白色表面卡片,石色边框(#E7E5E4)。琥珀色强调色切换为 amber-600 以获得对比度。

## 排版

- **Display/Hero:** Satoshi(Black 900 / Bold 700)——几何形但带温度,独特的字形(小写 `a` 与 `g`)。不是 Inter,不是 Geist。从 Fontshare CDN 加载。
- **Body:** DM Sans(Regular 400 / Medium 500 / Semibold 600)——干净、易读,比几何展示体稍显友好。从 Google Fonts 加载。
- **UI/Labels:** DM Sans(与 body 相同)
- **Data/Tables:** JetBrains Mono(Regular 400 / Medium 500)——个性字体。支持 tabular-nums。等宽字体应显著,而非藏在代码块里。从 Google Fonts 加载。
- **Code:** JetBrains Mono
- **加载:** Google Fonts 加载 DM Sans + JetBrains Mono,Fontshare 加载 Satoshi。使用 `display=swap`。
- **字号:**
  - Hero: 72px / clamp(40px, 6vw, 72px)
  - H1: 48px
  - H2: 32px
  - H3: 24px
  - H4: 18px
  - Body: 16px
  - Small: 14px
  - Caption: 13px
  - Micro: 12px
  - Nano: 11px(JetBrains Mono labels)

## 布局

- **取向:** 仪表板用网格纪律,落地页用编辑型 hero
- **网格:** lg+ 12 列,移动端 1 列
- **内容最大宽度:** 1200px (6xl)
- **圆角:** sm:4px,md:8px,lg:12px,full:9999px
  - 卡片/面板:lg (12px)
  - 按钮/输入:md (8px)
  - 徽标/胶囊:full (9999px)
  - 技能条:sm (4px)

### 间距

- **基础单位:** 4px
- **密度:** 舒适——不局促(不像 Bloomberg Terminal),不空旷(不像营销站点)
- **刻度:** 2xs(2px) xs(4px) sm(8px) md(16px) lg(24px) xl(32px) 2xl(48px) 3xl(64px)

## 动效

- **取向:** 极简且功能化——只保留有助于理解的过渡。仪表板的实时信息流本身就是动效。
- **缓动:** enter(ease-out / cubic-bezier(0.16,1,0.3,1)) exit(ease-in) move(ease-in-out)
- **时长:** micro(50-100ms) short(150ms) medium(250ms) long(400ms)
- **动效元素:** 实时信息流圆点脉冲(2s 无限循环)、技能条填充(600ms ease-out)、hover 状态(150ms)

## 颗粒纹理

在整个页面上施加细微的噪点叠加,以增加材质感:

- 暗色模式:不透明度 0.03
- 亮色模式:不透明度 0.02
- 在 `body::after` 上使用 SVG feTurbulence 滤镜作为 CSS `background-image`
- `pointer-events: none`,`position: fixed`,`z-index: 9999`

## 决策日志(Decisions Log)

| 日期 | 决策 | 理由 |
|------|------|------|
| 2026-03-21 | 初始设计系统 | 由 /design-consultation 创建。工业美学、暖色琥珀强调色,Satoshi + DM Sans + JetBrains Mono。 |
| 2026-03-21 | 亮色模式 amber-600 | amber-500 在白底上太亮/发白,amber-700 又太棕/赭石。amber-600 是甜蜜点。 |
| 2026-03-21 | 颗粒纹理 | 给扁平的暗色表面增加材质感。避免「通用 SaaS 模板」的同质化。 |
| 2026-09-08 | 开放 DESIGN.md 格式 | 用 `gstack-design-md convert` 转换:token 放在 frontmatter,保留规范章节,Motion 与 Decisions Log 保留为附加项。DM Sans 仍是 body 字体:它属于「被过度用作展示体」列表,但在 Operate 表面上的 body/UI 用法是角色范围内规则允许的例外。 |
| 2026-09-08 | 实时信息流脉冲是唯一被编排的动效瞬间 | 实时信息流圆点上的 2s 脉冲是唯一被编排的动效瞬间,作为脉冲圆点目录规则的有意例外。技能条填充与 150ms hover 过渡是功能性状态转换,不是被编排的瞬间。 |
| 2026-09-08 | 数据标签用 JetBrains Mono 11px | 数据标签在 Nano (11px) 用 JetBrains Mono,低于 12px 的 UI 文本下限;UI 标签保持 DM Sans(`typography.label` token)。如此记录,让检测器的「UI 文本过小」与「等宽字体服装」行被读为已决定,而非待决。 |
