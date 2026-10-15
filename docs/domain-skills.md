# 领域技能(Domain Skills)

> **范围:** domain-skills 是 gstack 自己的浏览器引擎(`$B`)的功能,这是当 Aside 浏览器不存在时(Linux、Windows、关闭的 Aside 应用程序)运行的**回退**。驱动 Aside 的 Skills 通过 `/learn` 记录每项目的 learnings。参见 [BROWSER.md](../BROWSER.md)。

智能体为自己编写的每站点注释。跨会话复合:一旦智能体弄清楚了某个非显而易见的网站,它就会保存一项技能,将来在该主机上的会话会将该注释注入到它们的提示上下文中。

这是 gstack 从 [browser-use/browser-harness](https://github.com/browser-use/browser-harness) 借鉴的内容。gstack 复制了每站点注释模式,**而不是**自修改运行时模式。技能是加载到提示中的 markdown 文本;它们不是可执行代码。

## 智能体如何使用它

```bash
# Agent wrote down what it learned about a site after a successful task.
# The host is taken from the active tab automatically (no agent argument).
echo "# LinkedIn Apply Button

The Apply button on /jobs/view pages is inside an iframe with a class
matching 'jobs-apply-button-iframe'. Use \$B frame --url 'apply' first,
then snapshot." | $B domain-skill save

# See what's saved
$B domain-skill list

# Read the body of a specific host's skill
$B domain-skill show linkedin.com

# Edit interactively in $EDITOR
$B domain-skill edit linkedin.com

# Promote an active per-project skill to global (cross-project)
$B domain-skill promote-to-global linkedin.com

# Roll back a recent edit
$B domain-skill rollback linkedin.com

# Delete (tombstone — recoverable via rollback)
$B domain-skill rm linkedin.com
```

## 状态机

```
  ┌──────────────┐  3 successful uses        ┌────────┐  promote-to-global   ┌────────┐
  │ quarantined  │ ─────────────────────▶  │ active │ ──────────────────▶  │ global │
  │ (per-project)│  (no classifier flags)   │(project)│  (manual command)    │        │
  └──────────────┘                          └────────┘                      └────────┘
         ▲                                       │
         │  classifier flag during use           │  rollback (version log)
         └───────────────────────────────────────┘
```

新的保存落地为 **quarantined(隔离)**,不会在提示中自动触发。在该主机上 3 次使用后,如果 L4 ML 分类器未标记技能内容,则该技能会自动提升为项目中的 **active(活动)**。活动技能在该主机名的每个新 sidebar-agent 会话中触发。

要使技能跨项目触发(例如,"我希望我的 LinkedIn 技能在我工作的每个 gstack 项目上都可用"),请显式运行 `$B domain-skill promote-to-global <host>`。这是按设计选择加入的(Codex T4 外部声音评审):全面的跨项目复合会在不相关的工作之间泄漏上下文。

## 存储

技能存在于两个位置:

- **Per-project(每个项目)**:`~/.gstack/projects/<slug>/learnings.jsonl` — 与 `/learn` skill 使用的同一 JSONL 文件。领域技能是 `type:"domain"` 行。
- **Global(全局)**:`~/.gstack/global-domain-skills.jsonl` — 仅 `state:"global"` 行。

两个文件都是仅追加 JSONL。删除的墓碑;空闲的压缩器定期重写文件。宽容的解析器在读取时丢弃部分尾随行,以便写入中途崩溃不会污染后续读取。

## 安全模型

技能是智能体编写的内容,加载到未来的提示上下文中。这使它们成为经典的智能体到智能体提示注入向量。该计划明确通过多层解决了这个问题:

| 层 | 内容 | 位置 |
|-------|------|-------|
| L1-L3 | 数据标记、隐藏元素剥离、ARIA 正则表达式、URL 黑名单 | `content-security.ts`(已编译二进制文件) |
| L4 | TestSavantAI ONNX 分类器 | `security-classifier.ts`(sidebar-agent,未编译) |
| L4b | Claude Haiku 脚本分类器 | `security-classifier.ts`(sidebar-agent) |
| L5 | 金丝雀 token 泄漏检测 | `security.ts` |

L1-L3 检查在**保存时**(在守护进程中)运行。L4 ML 分类器在**加载时**(在 sidebar-agent 中)运行,因此每个将技能加载到其提示中的会话也会重新验证内容。这捕获仅在分类器模型更新后出现的问题。

save 命令从**活动标签的顶层 origin** 派生主机名,而不是从智能体参数派生。这关闭了 Codex 标记的 confused-deputy 错误:恶意页面重定向链可能会欺骗智能体污染不同的域。

## 错误参考

| 错误 | 原因 | 操作 |
|-------|-------|--------|
| `Save blocked: classifier flagged content as potential injection` | 保存时 L4 分数 ≥ 0.85 | 重写技能,删除类似指令的散文;重试。 |
| `Save blocked: <L1-L3 message>` | 保存时 URL 黑名单匹配或 ARIA 注入 | 检查技能主体中的可疑模式。 |
| `Save failed: empty body` | 通过 stdin 或 `--from-file` 无内容 | 将 markdown 通过管道传送到 `$B domain-skill save`,或传递 `--from-file <path>`。 |
| `Cannot save domain-skill: no top-level URL on active tab` | 标签为 `about:blank` 或 `chrome://...` | 首先 `$B goto <target-site>`,然后保存。 |
| `Cannot promote: skill is in state "quarantined"` | 技能尚未自动提升 | 在此项目中使用它,直到 3 次成功运行没有分类器标记。 |
| `Cannot rollback: <host> has fewer than 2 versions` | 仅存在一个版本 | 改用 `$B domain-skill rm` 删除。 |

## 遥测

当启用遥测时(默认 `community` 模式,除非关闭),以下事件写入 `~/.gstack/analytics/browse-telemetry.jsonl`:

- `domain_skill_saved {host, scope, state, bytes}`
- `domain_skill_save_blocked {host, reason}`
- `domain_skill_fired {host, source, version}`
- `domain_skill_state_changed {host, from_state, to_state}`(计划中)

仅主机名 — 没有主体内容,没有智能体文本。完全禁用,使用 `gstack-config set telemetry off` 或 `GSTACK_TELEMETRY_OFF=1`。
