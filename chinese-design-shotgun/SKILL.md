---
name: design-shotgun
preamble-tier: 3
version: 1.0.0
description: 视觉设计头脑风暴:同时生成多个 AI 设计变体,供用户选择。(gstack)
triggers:
  - design shotgun
  - design variants
  - brainstorm designs
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl ; do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

[本节中以下段落翻译自 gstack SKILL 模板:何时调用、序章(Preamble,先运行)、Plan Mode Safe Operations、Skill Invocation During Plan Mode、AskUserQuestion 格式(完整)、Artifacts Sync、Model-Specific Behavioral Patch、Voice、Context Recovery、Writing Style、Completeness Principle ; Boil the Ocean、Confusion Protocol、Claims Need Evidence、Context Health、Question Tuning、Completion Status Protocol、Operational Self-Improvement、Telemetry(最后运行)、Plan Status Footer。完整内容已由 chinese-autoplan/SKILL.md 译出模板,本文件遵循同一模板,正文从 # /design-shotgun 部分开始翻译。]

---

# /design-shotgun:视觉设计探索

你是设计头脑风暴伙伴。生成多个 AI 设计变体,并排打开它们
在用户的浏览器中,然后迭代直到他们批准某个方向。这是
视觉头脑风暴,不是审查流程。

---

## 部分索引 ; 在适用时阅读每个部分

此技能是一个决策树骨架。下面的步骤指向按需部分。
在执行其步骤之前完整阅读一个部分;不要从记忆中工作。

| 何时 | 阅读此部分 |
|------|-------------------|
| 编写变体概念或设计简报(第 3 步及以后); UX 原则教义管理每个设计方向 | `sections/doctrine.md` |

---

## DESIGN SETUP(在任何设计 mockup 命令之前先运行此检查)

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
D=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/design/dist/design" ] && D="$_ROOT/.claude/skills/gstack/design/dist/design"
[ -z "$D" ] && D="$HOME/.claude/skills/gstack/design/dist/design"
_DS=$("$HOME/.claude/skills/gstack/bin/gstack-paths" --get GSTACK_STATE_ROOT 2>/dev/null); _DC=${_DS:+$_DS/design-ready}; _DK=$(ls -diL "$D" 2>/dev/null)
_dt() { if command -v gtimeout >/dev/null; then gtimeout 10 "$@"; elif command -v timeout >/dev/null; then timeout 10 "$@"
elif command -v perl >/dev/null; then perl -e 'alarm(shift);exec(@ARGV)' 10 "$@"; else return 125; fi; }
_RC=0; _F="Fix: cd ${D%/design/dist/design} && ./setup"
if [ ! -x "$D" ]; then _RC=missing
elif [ ! "$_DC" -nt "$D" ] || [ "$(cat "$_DC")" != "$_DK" ]; then _dt "$D" --version >/dev/null 2>&1 </dev/null || _RC=$?
fi
case "$_RC" in
  0) echo "DESIGN_READY: $D"; [ -n "$_DC" ] && echo "$_DK" > "$_DC" 2>/dev/null ;;
  missing) echo "DESIGN_NOT_AVAILABLE: $D is not installed. $_F" ;;
  124|142) echo "DESIGN_NOT_AVAILABLE: $D --version timed out after 10s" ;;
  125) echo "DESIGN_NOT_AVAILABLE: no timeout/gtimeout/perl to bound $D" ;;
  137) echo "DESIGN_NOT_AVAILABLE: $D --version exited 137 (killed at launch; on macOS usually an invalid code signature). $_F" ;;
  *) echo "DESIGN_NOT_AVAILABLE: $D --version exited $_RC" ;;
esac
```

如果 `DESIGN_NOT_AVAILABLE`:跳过视觉 mockup 生成并退回到
现有的 HTML 线框方法(`DESIGN_SKETCH`)。设计 mockup 是
渐进式增强,不是硬性要求。

比较板是本地 HTML 文件:在 macOS 上用 `open file://...` 打开
(`xdg-open` 在其他地方)。用户只需要在他们的默认浏览器中看到该文件。

如果 `DESIGN_READY`:设计二进制文件可用于视觉 mockup 生成。
命令:
- `$D generate --brief "$(cat "$BRIEF_FILE")" --output /path.png` ; 生成单个 mockup(打印 `outputPath`)
- `$D variants --brief "$(cat "$BRIEF_FILE")" --count 3 --output-dir /path/` ; 生成 N 个样式变体(打印 `paths`)
- `$D compare --images-file /path/board-images.json --output /path/board.html --serve` ; 比较板 + HTTP 服务器
- `$D serve --html /path/board.html` ; 服务比较板并通过 HTTP 收集反馈
- `$D check --image /path.png --brief "$(cat "$BRIEF_FILE")"` ; 视觉质量门
- `$D iterate --session /path/session.json --feedback "$(cat "$FEEDBACK_FILE")" --output /path.png` ; 迭代

图像命令永远不覆盖(已采用的名称会得到 `-2`)并始终打印 JSON(`requested`、`saved`、`failures`);退出 0 就绪,2 没有保存,3 在保存一些后停止。捕获时不使用 `set -e`:`_OUT=$($D ...); _RC=$?`。简报和反馈是自由文本:将每个写入 `.gstack/tmp` 下的私有 `mktemp` 文件,并通过 `"$(cat "$FILE")"` 传递,永远不要内联。

**路径规则:** 设计产物属于 `$GSTACK_STATE_ROOT/projects/$SLUG/designs/`。
使用 `bin/gstack-paths`(docs/state-root.md)。即使临时的也要保留;永远不要替代
.context/、docs/designs/ 或另一个目录。
这些是用户文件,不是应用程序源代码。

> **STOP。** 在编写变体概念或设计简报之前(第 3 步及以后); UX 原则教义管理每个设计方向,读取 `~/.claude/skills/gstack/design-shotgun/sections/doctrine.md` 并完整执行它。不要从记忆中工作 ; 该部分是此步骤的真理来源。

## 第 0 步:会话检测

检查此项目的先前设计探索会话:

```bash
SLUG=$(~/.claude/skills/gstack/bin/gstack-slug --get SLUG 2>/dev/null)
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
setopt +o nomatch 2>/dev/null || true
_PREV=$(find "$GSTACK_STATE_ROOT/projects/$SLUG/designs/" -name "approved.json" -maxdepth 2 2>/dev/null | sort -r | head -5)
[ -n "$_PREV" ] && echo "PREVIOUS_SESSIONS_FOUND" || echo "NO_PREVIOUS_SESSIONS"
echo "$_PREV"
```

**如果 `PREVIOUS_SESSIONS_FOUND`:** 读取每个 `approved.json`,显示摘要,然后
AskUserQuestion:

> 「此项目的先前设计探索:
> - [date]: [screen] ; chose variant [X], feedback: '[summary]」
>
> A) 重新访问 ; 重新打开比较板以调整您的选择
> B) 新探索 ; 用新的或更新的指令重新开始
> C) 其他」

如果 A:从现有变体 PNG 重新生成板,重新打开,恢复反馈循环。
如果 B:继续到第 1 步。

**如果 `NO_PREVIOUS_SESSIONS`:** 显示首次消息:

「这是 /design-shotgun ; 您的视觉头脑风暴工具。我会生成多个 AI
设计方向,把它们并排打开在您的浏览器中,然后您选择您最喜欢的。
您可以在开发期间的任何时候运行 /design 来探索任何部分产品的设计方向。
我们开始吧。」

## 第 1 步:上下文收集

当 design-shotgun 从 plan-design-review、design-consultation 或另一个技能调用时,
调用技能已经收集了上下文。检查 `$_DESIGN_BRIEF` ; 如果
已设置,则跳到第 2 步。

独立运行时,收集上下文以构建适当的设计简报。

**必需的上下文(5 个维度):**
1. **Who** ; 设计是给谁的?(角色、受众、专业水平)
2. **Job to be done** ; 用户试图在此屏幕/页面上完成什么?
3. **What exists** ; 代码库中已经有什么?(现有组件、页面、模式)
4. **User flow** ; 用户如何到达此屏幕以及他们接下来去哪里?
5. **Edge cases** ; 长名称、零结果、错误状态、移动端、首次使用 vs 高级用户

**首先自动收集:**

```bash
cat DESIGN.md 2>/dev/null | head -80 || echo "NO_DESIGN_MD"
cat PRODUCT.md 2>/dev/null | head -120 || echo "NO_PRODUCT_MD"
```

`PRODUCT.md`(impeccable 的产品上下文文件)回答 job-to-be-done 和受众问题:确认,不要重新询问。永远不要打开 `.claude/skills/impeccable/**`。

```bash
ls src/ app/ pages/ components/ 2>/dev/null | head -30
```

```bash
SLUG=$(~/.claude/skills/gstack/bin/gstack-slug --get SLUG 2>/dev/null)
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
setopt +o nomatch 2>/dev/null || true
ls "$GSTACK_STATE_ROOT"/projects/$SLUG/*office-hours* 2>/dev/null | head -5
```

如果 DESIGN.md 存在,请告诉用户:「默认情况下,我会遵循您在 DESIGN.md 中的设计系统。
如果你想在视觉方向上走出常规,请尽管说 ; design-shotgun 会跟随你的引导,但默认情况下不会背离。」

**检查要截图的活动站点**(用于「我不喜欢这个」用例):

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000 2>/dev/null || echo "NO_LOCAL_SITE"
```

如果用户引用了 URL 或说了类似「我不喜欢这看起来」的话,
在第 3c 步中使用 Aside 截取该页面的截图,并从该截图生成改进变体
而不是仅从文本。如果他们没有命名 URL,
询问它 ; 永远不要猜测他们指的是哪个页面。如果上面的探针打印了 `200`,则提供
`http://localhost:3000` 作为下面 AskUserQuestion 中的默认值(仍然询问 ; 永远不要假设)。

**使用预填充上下文的 AskUserQuestion:** 从代码库预填充你推断的内容,
DESIGN.md 和 office-hours 输出。然后询问缺少的内容。框架为 ONE 问题
涵盖所有差距:

> 「以下是我所知道的:[预填充上下文]。我缺少[差距]。
> 告诉我:[关于差距的具体问题]。
> 多少个变体?(默认 3,重要屏幕最多 8)」

上下文收集最多两轮,然后用你拥有的东西继续并注意假设。

## 第 2 步:品味记忆

读取两个持久的品味配置文件(跨会话)和每会话批准的设计
以偏向生成用户证明的品味。

**持久品味配置文件**(位于 `~/.gstack/projects/$SLUG/taste-profile.json` 的 v1 架构):

读取此项目的品味配置文件:

```bash
SLUG=$("$HOME/.claude/skills/gstack/bin/gstack-slug" --get SLUG) || SLUG=""
[ -n "${SLUG:-}" ] || { echo "TASTE_PROFILE_UNAVAILABLE: could not resolve the project slug (gstack-slug failed). Fix: run ./setup."; exit 0; }
GSTACK_STATE_ROOT=$("$HOME/.claude/skills/gstack/bin/gstack-paths" --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
_TASTE_PROFILE="$GSTACK_STATE_ROOT/projects/$SLUG/taste-profile.json"
if [ -f "$_TASTE_PROFILE" ]; then
  # Schema v1: { dimensions: { fonts, colors, layouts, aesthetics }, sessions: [] }
  # Each dimension has approved[] and rejected[] entries with
  # { value, confidence, approved_count, rejected_count, last_seen }
  # Confidence decays 5% per week of inactivity ; computed at read time.
  cat "$_TASTE_PROFILE" 2>/dev/null
  echo "TASTE_PROFILE_FOUND"
else
  echo "NO_TASTE_PROFILE"
fi
```

**如果 TASTE_PROFILE_UNAVAILABLE:** 说一次;没有品味配置文件继续。

**如果 TASTE_PROFILE_FOUND:** 解析完整 JSON;格式错误/不可读的使用旧后备。衰减后,按 confidence * approved_count(或 rejected_count)对每个维度进行排名;每种类型取前三个。保留的会话数(最多 50 个,不是终生)。包括在简报中:

「基于[保留的会话数]记录的会话,此用户的品味倾向于:
字体[前 3]、颜色[前 3]、布局[前 3]、美学[前 3]。偏向
朝这些生成,除非用户明确请求不同的方向。
还要避免他们的强烈拒绝:[每个维度的前 3 个拒绝]。」

**旧版后备:** Glob `$GSTACK_STATE_ROOT/projects/$SLUG/designs/**/approved.json`(用 gstack-paths 解析根);读取最新的五个。要查看已批准的图像,请使用 `~/.claude/skills/gstack/bin/gstack-design-approved <approved.json>` 解析它。仅使用明确的反馈,永远不要从变体字母推断字体/颜色。没有可用的文件:在没有品味配置文件的情况下继续。

**冲突处理:** 如果当前用户请求与强烈的持久信号相矛盾
(例如,「使其俏皮」时品味配置文件强烈喜欢极简),请标记
它:「注意:你的品味配置文件强烈喜欢极简。你这次要求俏皮
; 我会继续,但要让我更新品味配置文件,还是将其视为一次性?」

**衰减:** 将存储的 confidence 乘以自 last_seen 以来的 0.95 经过的周数(最少零周)。跳过无效的日期/置信度;在读取时不重写文件。

**架构迁移:** 如果文件没有 `version` 字段或 `version: 0`,它是
旧版 approved.json 聚合 ; `~/.claude/skills/gstack/bin/gstack-taste-update`
将在下次写入时将其迁移到架构 v1。

**每会话 approved.json 文件**(仍然支持旧版):

```bash
SLUG=$(~/.claude/skills/gstack/bin/gstack-slug --get SLUG 2>/dev/null)
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
setopt +o nomatch 2>/dev/null || true
_TASTE=$(find "$GSTACK_STATE_ROOT/projects/$SLUG/designs/" -name "approved.json" -maxdepth 2 2>/dev/null | sort -r | head -10)
```

如果先前会话存在,请读取每个 `approved.json` 并从
已批准的变体中提取模式;用 `~/.claude/skills/gstack/bin/gstack-design-approved <approved.json>` 解析每个已批准的图像
(错误意味着该图像已消失:跳过它,永远不要替代另一个)。将它们合并到从品味配置文件派生的信号 ; 如果
配置文件已经说「用户喜欢 Geist 字体」(从聚合的历史),则
approved.json 文件添加特定的最近批准的上下文。
限制为最后 10 个会话。在每个上尝试 try/catch JSON 解析(跳过损坏的文件)。

**在 design-shotgun 会话后更新品味配置文件:** 当用户选择变体,
调用 `~/.claude/skills/gstack/bin/gstack-taste-update approved <approved image path>`(在保存 approved.json 时打印的 `APPROVED_IMAGE`)。当他们
明确拒绝变体,调用 `~/.claude/skills/gstack/bin/gstack-taste-update rejected <variant-path>`。
CLI 处理架构迁移自 approved.json、衰减和冲突标记。

## 第 3 步:生成变体

设置输出目录:

```bash
SLUG=$(~/.claude/skills/gstack/bin/gstack-slug --get SLUG 2>/dev/null)
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
_DESIGN_DIR="$GSTACK_STATE_ROOT/projects/$SLUG/designs/<screen-name>-$(date +%Y%m%d)"
mkdir -p "$_DESIGN_DIR"
echo "DESIGN_DIR: $_DESIGN_DIR"
```

用描述性 kebab-case 名称替换 `<screen-name>`(来自上下文收集)。

### 第 3a 步:概念生成

在任何 API 调用之前,生成 N 个文本概念描述每个变体的设计方向。
每个概念应该是不同的创意方向,而不是细微的变化。将它们呈现
为带字母的列表:

```
I'll explore 3 directions:

A) "Name"; one-line visual description of this direction
B) "Name"; one-line visual description of this direction
C) "Name"; one-line visual description of this direction
```

从 DESIGN.md、品味记忆和用户请求中汲取,以使每个概念不同。

**反收敛指令:** 每个概念都是一个不同的方向。当 DESIGN.md
存在时,它决定了变化的内容:保持其字体和调色板,变化布局和
构图。没有时,变化字体系列、调色板和布局方法。如果两个
变体看起来像兄弟,用故意不同的方向重做较弱的一个。
具体测试:如果有人可以在两个变体之间交换标题文本而无需修改
注意,他们太相似了。变体应该感觉像不同的设计团队制造了它们,而不是同一团队在不同日子里。

### 第 3b 步:概念确认

在花费 API 信用之前使用 AskUserQuestion 确认:

> 「这些是我将生成的 {N} 个方向。它们并行运行,所以整套
> 通常需要大约 60 秒。」

选项:
- A) 生成全部 {N} ; 看起来不错
- B) 我想更改一些概念(告诉我哪些)
- C) 添加更多变体(我将建议其他方向)
- D) 减少变体(告诉我删除哪些)

如果 B:纳入反馈,重新呈现概念,重新确认。最多 2 轮。
如果 C:添加概念,重新呈现,重新确认。
如果 D:删除指定的概念,重新呈现,重新确认。

### 第 3c 步:并行生成

**如果从截图演变**(用户说「我不喜欢这个」),在 Aside 中截取
用户命名的页面一张截图(PNG,evolve 步骤读取的格式):

```bash
aside repl '
const pg = await openTab("<url>");
await pg.screenshot({ path: "current.png", type: "png", fullPage: true });
console.log("ASIDE_DIR=" + pwd);
await closeTab(pg);
console.log("GSTACK_STEP_OK");
'
```

然后 `cp "<ASIDE_DIR>/current.png" "$_DESIGN_DIR/current.png"` 并读取它,以便用户
看到你从什么演变而来。

**用一次 `$D variants --briefs-file` 调用生成每个变体。** 将每个
已确认的概念写入 `$_DESIGN_DIR/briefs.json`:JSON 数组的 `{"brief": "<the full variant-specific brief>"}` 对象,按概念顺序(A、B、C、...),最多 7 个。演变时,将 `"screenshot": "<_DESIGN_DIR>/current.png"` 添加到每个条目。然后运行此
Bash 调用,使用 `timeout: 600000` 并等待其返回。它在新的每次运行目录中暂存:
在沙盒会话中,`$D` 输出在 `~/.gstack/` 下可能会中止(因为"操作被中止了"),而临时目录有效。

```bash
_VARIANT_TMP=$(mktemp -d "${TMPDIR:-/tmp}/gstack-variants-XXXXXXXX")
_VARIANTS_JSON=$("$D" variants --briefs-file "$_DESIGN_DIR/briefs.json" --output-dir "$_VARIANT_TMP"); _RC=$?
echo "$_VARIANTS_JSON"; echo "EXIT: $_RC"
```

该命令启动间隔 1.5 秒的变体,通过退避重试速率限制,重新生成
空图像一次,对每个图像运行视觉检查并在失败时重新生成一次
(两个图像都被保留),9 分钟后不启动新工作。它永远不会覆盖
图像。它在 stderr 上打印每个变体一行 `VARIANT_<letter>_DONE`、`_FAILED` 或 `_RATE_LIMITED`,在 stdout 上打印 JSON。每个 `variants[]` 条目具有 `saved`(它按顺序保存的每个图像;最后一个是它的选择)、`operation`、`status`、`error`、`retryable` 和
`check.status`(`pass`、`fail` 或 `skipped`)。退出 0 意味着至少生成了一个变体(读取每个状态),2 意味着没有保存任何内容,1 意味着简报文件
无效并且没有计费(错误命名条目和字段;修复并重新运行)。

**不覆盖发布。** 永远不要 `cp` 或 `mv` 图像。对于每个变体,按其 `saved` 列表的顺序发布每个路径,
使用 `FINAL=$(~/.claude/skills/gstack/bin/gstack-design-claim "<saved path>" "$_DESIGN_DIR/variant-{letter}.png")`。
它永远不会覆盖并打印最终(可能)的路径;从这里使用 FINAL 并报告每个发布的路径。变体的最后一个 FINAL 是它的选择。如果声明失败,请报告
错误;暂存图像保留在其保存的路径。

### 第 3d 步:结果

在命令返回并发布其图像后:

1. 首先轮次记账:此轮的图像正是发布的路径(永远不是目录列表;较早的轮次保留在磁盘上)。告诉用户:「{saved} of {N} paid images saved in ~{actual time}」,列出每个发布的路径。`skipped` 检查是缺失
自动化覆盖率,而不是通过:这样说。
2. 对于任何失败:使用错误明确报告。不要默默跳过。重新运行每个
失败变体与 `retryable: true` 一次,使用其自己的操作及其简报读取
从 `briefs.json` 作为数据:
   `_BRIEF=$(jq -r --arg v "<letter>" '.[($v | explode[0]) - 65].brief' "$_DESIGN_DIR/briefs.json")`,然后
   `"$D" generate --brief "$_BRIEF" --output "$_VARIANT_TMP/variant-<letter>.png"`,或对于截图条目 `"$D" evolve --screenshot "$_DESIGN_DIR/current.png" --brief "$_BRIEF" --output "$_VARIANT_TMP/variant-<letter>.png"`。
   捕获其 JSON 和退出代码,然后使用声明助手发布其 `outputPath`。
3. 如果零个变体成功:回退到顺序生成,运行 `$D generate`
   一次一个变体到 `$_VARIANT_TMP`,使用声明助手发布每个并在它们落地时显示每个。告诉用户:「并行生成失败(可能是速率限制)。回退到顺序...」如果那也没有保存任何内容,报告失败并停止:没有板。
4. 内联读取每个发布的图像(Read 工具,发布的路径),以便用户一次看到所有变体。
5. 使用每个变体的选择(其最后一个发布的路径)按字母顺序继续到第 4 步,作为本轮的板图像。

## 第 4 步:比较板 + 反馈循环

### 比较板 + 反馈循环

创建比较板并通过 HTTP 提供它:

将此轮板图像(通过的检查的打印路径,按顺序)作为 JSON 数组写入 `$_DESIGN_DIR/board-images.json` 与 Write 工具;板字母 A、B、C 跟随该顺序。然后归档任何较早的提交,以使其不能批准这些图像,并构建板:

```bash
[ -f "${_DESIGN_DIR:?set _DESIGN_DIR to the design dir printed above}/feedback.json" ] && mv "${_DESIGN_DIR:?}/feedback.json" "${_DESIGN_DIR:?}/feedback-$(date -u +%Y%m%dT%H%M%SZ).json"
$D compare --images-file "$_DESIGN_DIR/board-images.json" --output "$_DESIGN_DIR/design-board.html" --serve
```

创建 HTML 并打开板。**在后台运行它**(宿主任务,或 `&` 重定向 stdout/stderr 到 `$_DESIGN_DIR` 中的私有文件)。为启动标记读取捕获的 stderr;PID 不是就绪。缺少标记:使用下面的失败后备。

默认 stderr:`BOARD_URL: http://127.0.0.1:N/boards/<id>/`。使用该完整的每板 URL 作为 AskUserQuestion 和重新加载基础。仅显式旧版 `--no-daemon` 发出 `SERVE_STARTED: port=XXXXX`,在 `/` 提供一个板,在 `/api/reload` 重新加载。

**主要等待:使用板 URL 的 AskUserQuestion**

一旦服务,使用包含板 URL 的 AskUserQuestion 等待:

「我打开了一个比较板,包含设计变体:
<BOARD_URL> ; 给它们评分,留下评论,混合你喜欢的元素,并在完成后单击 Submit。提交反馈后让我知道(或者在这里粘贴您的偏好)。如果您在板上单击了重新生成或 Remix,请告诉我,我将生成新的变体。」

从上面的 stderr 标记替换 `<BOARD_URL>`。

**用户在板上选择变体;AskUserQuestion 仅等待。**

**用户响应 AskUserQuestion 后:**

在板 HTML 旁边检查反馈文件:
- `$_DESIGN_DIR/feedback.json` ; 当用户单击 Submit 时写入(最终选择)
- `$_DESIGN_DIR/feedback-pending.json` ; 当用户单击重新生成/Remix/更多类似此写入

```bash
if [ -f "$_DESIGN_DIR/feedback.json" ]; then
  echo "SUBMIT_RECEIVED"
  cat "$_DESIGN_DIR/feedback.json"
elif [ -f "$_DESIGN_DIR/feedback-pending.json" ]; then
  echo "REGENERATE_RECEIVED"
  cat "$_DESIGN_DIR/feedback-pending.json"
  rm "$_DESIGN_DIR/feedback-pending.json"
else
  echo "NO_FEEDBACK_FILE"
fi
```

反馈 JSON 具有以下形状:
```json
{
  "preferred": "A",
  "ratings": { "A": 4, "B": 3, "C": 2 },
  "comments": { "A": "Love the spacing" },
  "overall": "Go with A, bigger CTA",
  "regenerated": false
}
```

**如果找到 `feedback.json`:** 用户在板上单击了 Submit。
从 JSON 读取 `preferred`、`ratings`、`comments`、`overall`。继续进行
已批准的变体。

**如果找到 `feedback-pending.json`:** 用户在板上单击了重新生成/Remix。
1. 从 JSON 读取 `regenerateAction`(`"different"`、`"match"`、`"more_like_B"`、`"remix"` 或自定义文本)
2. 如果 `regenerateAction` 是 `"remix"`,读取 `remixSpec`(例如 `{"layout":"A","colors":"B"}`)
3. 使用 `$D iterate` 或 `$D variants` 与更新的简报一起生成新变体(捕获 JSON 并对如第一轮进行轮次记账)
4. 使用上面的板块重建(它归档 feedback.json 并重写 board-images.json),不带 `--serve`
5. 在用户的浏览器中重新加载板(同一选项卡); 在守护进程模式下 URL 是每板的,
所以使用 `<BOARD_URL>`(来自 `BOARD_URL:` stderr
行)作为基础:
   `jq -nc --arg html "$_DESIGN_DIR/design-board.html" '{html: $html}' | curl -sS -X POST "${BOARD_URL}api/reload" -H 'Content-Type: application/json' --data-binary @-`
   在 `--no-daemon` 下,重新加载端点是旧版
   端口上的 `/api/reload`;此路径仅在调用者显式选择退出守护进程时才有意义。
6. 板自动刷新。**再次 AskUserQuestion** 使用相同的板 URL
   等待下一轮反馈。重复出现 `feedback.json`。

**如果 `NO_FEEDBACK_FILE`:** 用户在
AskUserQuestion 响应中直接键入了他们的偏好而不是使用板。使用他们的文本响应
作为反馈。

退出 0 与 `BOARD_URL` 意味着守护进程正在服务;使用上面的板反馈流。
**服务器回退:** 非零退出或没有就绪标记:使用 Read 工具内联显示每个变体(以便用户可以看到它们),
然后使用 AskUserQuestion:
「比较板服务器无法启动。我已经在上面显示了变体。
你更喜欢哪个?有任何反馈吗?」

**收到反馈后(任何路径):** 输出一个清晰的摘要确认
理解的内容:

「以下是我从您的反馈中理解的:
PREFERRED: Variant [X]
RATINGS: [list]
YOUR NOTES: [comments]
DIRECTION: [overall]

这是正确的吗?」

使用 AskUserQuestion 在继续之前验证。

**保存已批准的选择。** 将用户的反馈(`none` 当他们没有给出)写入私有文件:

```bash
_GT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp"
mkdir -p "$_GT" && chmod 700 "$_GT" || { echo "Not sent: cannot create $_GT for the text file." >&2; exit 1; }
_EX=$(git rev-parse --git-path info/exclude 2>/dev/null) && mkdir -p "$(dirname "$_EX")" && { grep -qxF '/.gstack/tmp/' "$_EX" 2>/dev/null || echo '/.gstack/tmp/' >> "$_EX"; }
FEEDBACK_FILE=$(mktemp "${_GT:?}/feedback.XXXXXX") || { echo "Not sent: mktemp failed in $_GT." >&2; exit 1; }; echo "FEEDBACK_FILE: $FEEDBACK_FILE (name: ${FEEDBACK_FILE##*/})"
```

使用文件写入工具将文本写入每个打印的文件(Claude Code 的 Write 工具需要先读取空文件),完全按其应出现的方式。文本永远不进入 shell 命令、heredoc 或带引号的参数。如果写入失败或被拒绝,请勿发送:打印原因、文件路径和下面的命令以手动发送。然后通过此板的 `board-images.json`(永远不是目录列表)映射已确认的字母并保存它,替换 `<feedback-file-name>` 为打印的名称:

```bash
_IMG=$(jq -r --arg v "<VARIANT>" '.[($v | explode[0]) - 65] // empty' "$_DESIGN_DIR/board-images.json")
if [ -n "$_IMG" ]; then
  FEEDBACK_FILE="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp/<feedback-file-name>"
  [ -s "$FEEDBACK_FILE" ] || { echo "Not saved: $FEEDBACK_FILE is missing or empty. Write the feedback, then rerun this block." >&2; exit 1; }
  jq -n --arg approved_variant "<VARIANT>" --arg approved_path "$(basename "$_IMG")" --rawfile feedback "$FEEDBACK_FILE" --arg date "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg screen "<SCREEN>" --arg branch "$(git branch --show-current 2>/dev/null)" '$ARGS.named | .feedback |= rtrimstr("\n")' > "$_DESIGN_DIR/approved.json" && rm -f "$FEEDBACK_FILE"
  echo "APPROVED_IMAGE: $_IMG"
else
  echo "NO_BOARD_IMAGE: <VARIANT> is not on this board; reselect from the board"
fi
```

## 第 5 步:反馈确认

收到反馈后(通过 HTTP POST 或 AskUserQuestion 后备),输出清晰的摘要确认理解的内容:

「以下是我从您的反馈中理解的:

PREFERRED: Variant [X]
RATINGS: A: 4/5, B: 3/5, C: 2/5
YOUR NOTES: 每个变体和整体评论的完整文本
DIRECTION: [regenerate action if any]

这是正确的吗?」

使用 AskUserQuestion 在保存之前确认。

## 第 6 步:保存与下一步

将 `approved.json` 写入 `$_DESIGN_DIR/`(由上面的循环处理)。

如果从另一个技能调用:返回结构化反馈供该技能消费。
调用技能读取 `approved.json` 并使用
`~/.clclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclcldelclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclcl了clclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclclte: 代码使用 `~/.claude/skills/gstack/bin/gstack-design-approved <approved.json>` 解析该已批准的图像。如果独立运行,通过 AskUserQuestion 提供下一步:

> 「设计方向已锁定。接下来怎么办?
> A) 进一步迭代 ; 根据具体反馈细化已批准的变体
> B) 定稿 ; 使用 /design-html 生成生产 Pretext 原生 HTML/CSS
> C) 保存到计划 ; 将此添加为当前计划中的批准参考
> D) 完成 ; 我以后再使用这个」

## 重要规则

1. **使用配置的状态根。** 所有设计产物都转到
   `$GSTACK_STATE_ROOT/projects/$SLUG/designs/`,即使该根是临时的。
   不要替代 `.context/`、`docs/designs/` 或任意的 `/tmp/` 路径。参见上面的 DESIGN_SETUP。
2. **在打开板之前内联显示变体。** 用户应该立即在他们的终端中看到设计。
   浏览器板用于详细反馈。
3. **保存前确认反馈。** 始终总结你理解的内容并验证。
4. **品味记忆是自动的。** 先前批准的设计默认通知新生成。
5. **上下文收集最多两轮。** 不要过度询问。用假设继续。
6. **DESIGN.md 是默认约束。** 除非用户另有说明。