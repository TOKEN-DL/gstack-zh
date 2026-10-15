---
name: gstack-openclaw-retro
description: "每周的工程复盘。分析 commit 历史、工作模式以及代码质量指标,具有持久的历史和趋势跟踪。具有团队感知能力,可按人提供贡献、表扬与成长领域。在被要求每周复盘、本周交付了什么、或工程复盘时使用。"
---

# 每周工程复盘

生成一份综合的工程复盘,分析 commit 历史、工作模式和代码质量指标。具备团队感知:识别运行该命令的用户,然后分析每一位贡献者的个人贡献、表扬与成长机会。

## 参数

- 默认:最近 7 天
- `24h`:最近 24 小时
- `14d`:最近 14 天
- `30d`:最近 30 天
- `compare`:将当前窗口与前一个等长窗口比较

## 指示

解析参数以确定时间窗口。默认 7 天。所有时间应报告在用户的**本地时区**。

**午夜对齐的窗口:** 对于以天为单位的参数,在本地午夜计算绝对起始日期。例如,如果今天是 2026-03-18,窗口是 7 天,则起始日期是 2026-03-11。git log 查询使用 `--since="2026-03-11T00:00:00"`。对于以小时为单位的窗口,使用 `--since="N hours ago"`。

---

### Step 1:收集原始数据

先 fetch origin 并识别当前用户:

```bash
git fetch origin main --quiet
git config user.name
git config user.email
```

`git config user.name` 返回的名字就是 **"你"**……也就是阅读这份复盘报告的人。所有其他作者都是队友。

运行以下所有 git 命令(它们是独立的):

```bash
# 所有 commits,带时间戳、主题、hash、作者、改动的文件
git log origin/main --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat

# 每个 commit 的 test vs total LOC 分解,带作者
git log origin/main --since="<window>" --format="COMMIT:%H|%aN" --numstat

# Commit 时间戳,用于 session 检测与每小时分布
git log origin/main --since="<window>" --format="%at|%aN|%ai|%s" | sort -n

# 改得最频繁的文件
git log origin/main --since="<window>" --format="" --name-only | grep -v '^$' | sort | uniq -c | sort -rn

# 从 commit 消息中提取 PR 编号
git log origin/main --since="<window>" --format="%s" | grep -oE '[#!][0-9]+' | sort -t'#' -k1 | uniq

# 每个作者的文件热点
git log origin/main --since="<window>" --format="AUTHOR:%aN" --name-only

# 每个作者的 commit 数
git shortlog origin/main --since="<window>" -sn --no-merges

# 测试文件总数
git ls-files 2>/dev/null | grep -E '(\.test\.|\.spec\.|_test\.|_spec\.)' | wc -l

# 在窗口内改动的测试文件数
git log origin/main --format="" --name-only | grep -E '\.(test|spec)\.' | sort -u | wc -l
```

---

### Step 2:计算指标

计算并在摘要中展示这些指标:

- **到 main 的 commits:** N
- **贡献者:** N
- **已合并的 PRs:** N
- **总插入:** N
- **总删除:** N
- **净增 LOC:** N
- **Test LOC(插入):** N
- **Test LOC 占比:** N%
- **版本区间:** vX.Y.Z → vX.Y.Z
- **活跃天数:** N
- **检测到的 session 数:** N
- **平均每个 session 小时的 LOC:** N

然后**紧接其后**展示每位作者的排行榜:

```
Contributor         Commits   +/-          Top area
You (garry)              32   +2400/-300   browse/
alice                    12   +800/-150    app/services/
bob                       3   +120/-40     tests/
```

按 commit 数降序排序。当前用户始终排在最前面,标记为 "You (name)"。

---

### Step 3:Commit 时间分布

以本地时间展示每小时的直方图:

```
Hour  Commits  ████████████████
 00:    4      ████
 07:    5      █████
 ...
```

指出:
- 峰值时段
- 空白时段
- 双峰模式(早 / 晚)vs 连续
- 深夜编程簇(晚上 10 点之后)

---

### Step 4:工作 Session 检测

使用 45 分钟间隔离散相邻 commits 来检测 sessions。

对 session 进行分类:
- **Deep sessions**(50+ 分钟)
- **Medium sessions**(20–50 分钟)
- **Micro sessions**(<20 分钟,单 commit)

计算:
- 总活跃编程时间
- 平均 session 长度
- 活跃时间内每小时的 LOC

---

### Step 5:Commit 类型分解

按 conventional commit 前缀(feat / fix / refactor / test / chore / docs)分类。以百分比条形展示:

```
feat:     20  (40%)  ████████████████████
fix:      27  (54%)  ███████████████████████████
refactor:  2  ( 4%)  ██
```

如果 fix 比例超过 50%,打上标记……这意味着"先发后修"的模式,可能表示审查上有缺口。

---

### Step 6:热点分析

展示被改动最多的前 10 个文件。指出:
- 改了 5+ 次的文件(变动热点)
- 测试文件 vs 生产文件是否在热点列表里
- VERSION / CHANGELOG 的改动频率

---

### Step 7:PR 大小分布

估算 PR 大小并归桶:
- **Small**(<100 LOC)
- **Medium**(100–500 LOC)
- **Large**(500–1500 LOC)
- **XL**(1500+ LOC)

---

### Step 8:Focus Score + 周度旗舰

**Focus score:** 命中单个被改动最多的顶层目录的 commit 占比。占比越高,意味着越深度的专注工作。占比越低,意味着越分散的上下文切换。

**周度旗舰:** 窗口内 LOC 最高的单个 PR。突出显示 PR 编号、改动的 LOC,以及为什么它重要。

---

### Step 9:团队成员分析

对每位贡献者(包括当前用户),计算:

1. **Commits 和 LOC**……总 commits、插入、删除、净 LOC
2. **聚焦的区域**……他们改动最多的目录 / 文件(前 3)
4. **Commit 类型组合**……他们个人的 feat / fix / refactor / test 分解
5. **Session 模式**……他们什么时候写代码(峰值时段),session 数
6. **测试纪律**……他们个人的 test LOC 占比
7. **最大的交付**……他们单个最具影响力的 commit 或 PR

**对于当前用户("You"):** 最深入的分析。包括所有 session 分析、时间模式、focus score。以第一人称框架展开。

**对于每位队友:** 2–3 句话讲他们交付的内容以及他们的工作模式。然后:

- **表扬**(1–2 条具体的):锚定在实际的 commits 上。不是"做得不错"……具体指出好在哪里。
- **成长机会**(1 条具体的):将其框架化为升级,而非批评。锚定在实际数据上。

**如果只有一个人(个人 repo):** 跳过团队细分。

**AI 协作:** 如果 commit 带 `Co-Authored-By` AI trailers,把 "AI-assisted commits" 作为单独的指标来跟踪。

---

### Step 10:周环比趋势(窗口 ≥ 14d 时)

按周拆桶并展示趋势:
- 每周 commits(总与每位作者)
- 每周 LOC
- 每周 test 占比
- 每周 fix 占比
- 每周 session 数

---

### Step 11:连胜追踪

从今天向前回溯,数至少有 1 个 commit 的连续天数。

```bash
# 团队连胜
git log origin/main --format="%ad" --date=format:"%Y-%m-%d" | sort -u

# 个人连胜
git log origin/main --author="<user_name>" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

展示两个:
- "Team shipping streak: 47 consecutive days"
- "Your shipping streak: 32 consecutive days"

---

### Step 12:加载历史与对比

检查 `memory/` 中的过往复盘历史:

如果存在过往复盘,加载最近的一个并计算差值:

```
                    Last        Now         Delta
Test ratio:         22%    →    41%         ↑19pp
Sessions:           10     →    14          ↑4
LOC/hour:           200    →    350         ↑75%
Fix ratio:          54%    →    30%         ↓24% (improving)
```

如果没有过往复盘,标注 "First retro recorded, run again next week to see trends."

---

### Step 13:保存复盘历史

把 JSON 快照保存到 `memory/retro-YYYY-MM-DD.json`,包含指标、作者、版本区间、连胜和推文友好的摘要。

---

### Step 14:写出叙事

**Telegram 格式**(bullet、粗体,最终输出中不要 markdown 表格)。

结构:

**Tweetable summary**(第一行):
> Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm | Streak: 47d

然后是:
- **Summary**……关键指标
- **Trends vs Last Retro**……差值(首次复盘则跳过)
- **Time & Session Patterns**……团队什么时候写代码、session 长度、deep vs micro
- **Shipping Velocity**……commit 类型、PR 大小、fix-chain 检测
- **Code Quality Signals**……test 占比、热点、churn
- **Focus & Highlights**……focus score、周度旗舰
- **Your Week**……当前用户的个人深挖
- **Team Breakdown**……每位队友的分析,带表扬 + 成长(若个人 repo 则跳过)
- **Top 3 Team Wins**……已交付的最高影响的事件
- **3 Things to Improve**……具体的、可操作的、锚定在 commits 上
- **3 Habits for Next Week**……小的、实用的、现实的(<5 分钟可上手)

---

## Compare 模式

当用户说 "compare":
- 对当前窗口运行复盘
- 对前一个等长窗口运行复盘
- 并排展示指标,带箭头表示改善 / 退化
- 用一段简短叙事讲述最大的变化

---

## 重要规则

- **所有时间使用本地时区。** 永远不要设置 `TZ`。
- **为 Telegram 格式。** 使用 bullet 和粗体。在最终输出中避免 markdown 表格。
- **表扬锚定在 commits 上。** 永远不要在不指出好在哪里的情况下说"做得不错"。
- **成长方向锚定在数据上。** 永远不要在没有证据的情况下批评。
- **保存历史。** 每次复盘都保存到 `memory/` 以供趋势跟踪。
- **完成状态:**
  - DONE……已生成复盘,历史已保存
  - DONE_WITH_CONCERNS……已生成但缺少数据(例如,没有过往复盘可用于对比)
  - BLOCKED……不在 git 仓库中,或窗口中没有 commit