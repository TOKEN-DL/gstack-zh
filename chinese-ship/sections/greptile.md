<!-- AUTO-GENERATED from greptile.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## Step 10: 处理 Greptile 审查评论(如果存在 PR)

首先检查 PR。Greptile 审查 PR,因此没有 PR、没有 `gh`、没有 `gh` 登录或非 GitHub remote 时,还没有可分类的内容:

```bash
if ! command -v gh >/dev/null 2>&1; then echo "PR: skip (gh not installed)"
elif ! gh auth status >/dev/null 2>&1; then echo "PR: skip (gh not logged in)"
elif ! _GH_ERR=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>&1 >/dev/null); then echo "PR: skip (not a GitHub remote: $(printf '%s' "$_GH_ERR" | head -1))"
elif _PR_ERR=$(gh pr view --json number -q .number 2>&1 >/dev/null); then echo "PR: exists"
else case "$_PR_ERR" in *"no pull requests found"*) echo "PR: skip (no PR yet)" ;; *) echo "PR: skip (PR lookup failed: $(printf '%s' "$_PR_ERR" | head -1))" ;; esac
fi
```

仅 `PR: exists` 派发。`PR: skip (<reason>)` → 不派发;记录 "Greptile: not run (<reason>); runs on the PR once it exists" 并继续到 Step 11。

**Early PR (Step 6.5):** 当记录包含 `EARLY_PR` 和 `EARLY_PR_OPENED_AT` 时,首先等待 Greptile 对该 PR 的审查,替换两个值:

```bash
~/.claude/skills/gstack/bin/gstack-greptile-early wait <pr-number> --since <opened-at>
```

每次调用在大约 90 秒内返回;中继其进度行。在 `GREPTILE_REVIEW: pending` 上重新运行。`complete` 在下面派发。`timeout` 或 `unavailable` 采用 Unavailable 分类路由,带该行作为原因,从不作为取消评论数。早期头部的评论针对当前差异进行分类(`already_fixed` 当代码已转移)。

通过 Agent 派发子代理,设置 `subagent_type: "general-purpose"` 和 `run_in_background: false`,使用 Step 7 的共享前台派发规则。
它获取并分类所有 Greptile 评论,包括升级层级;父级处理决策并排队已批准的修复。

**子代理提示词:**

> You are classifying Greptile review comments for a /ship workflow. Read `~/.claude/skills/gstack/review/greptile-triage.md` and follow the fetch, filter, classify, and **escalation detection** steps. Do NOT fix code, do NOT reply to comments, do NOT commit; report only.
>
> For each comment, assign: `classification` (`valid_actionable`, `already_fixed`, `false_positive`, `suppressed`), `escalation_tier` (1 or 2), the file:line or [top-level] tag, body summary, and permalink URL.
>
> Return one JSON object on the LAST LINE:
> `{"status":"complete|no_pr|unavailable","total":N,"comments":[{"classification":"...","escalation_tier":N,"ref":"file:line","summary":"...","permalink":"url"},...],"reason":"..."}`
> Use `complete` only after a successful fetch, including zero comments; `no_pr` only after confirming no PR exists; `unavailable` for `gh`/API errors or incomplete classification. The latter two return zero total and an empty array. State the failure reason for `unavailable`; otherwise use an empty reason.

**父级处理:**

将最后一行解析为 JSON。需要显式状态、与评论数匹配的负整数总数,以及上面的状态/原因不变。未知或缺失的状态不可用,绝不是空的成功审查。

对于 `no_pr`,记录 "Greptile: no PR exists";对于零评论的 `complete`,记录 "Greptile: fetched, zero comments"。两者都继续到 Step 11。

**不可用分类:** 返回的 `unavailable`、失败的派发、无效的结果或约 10 分钟后缺少完成采用此路由。在继续之前停止正在运行的子级并确认其已停止。打印 `Greptile triage did not complete; review the PR comments manually`。
在 Step 19 的审查结果中包含 `Greptile triage: UNAVAILABLE (dispatch failed)` 和实际原因;Step 20 没有分类字段。继续到 Step 11,而不声称零评论或已完成的分类。此可选分类不阻塞 ship。

否则,打印 `+ {total} Greptile comments ({valid_actionable} valid, {already_fixed} already fixed, {false_positive} FP)`。

对于 `comments` 中的每个评论:

**VALID & ACTIONABLE:** 使用 AskUserQuestion:
- 评论(文件:行 或 [顶级] + 正文摘要 + 永久链接 URL)
- `RECOMMENDATION: Choose A because [one-line reason]`
- 选项: A) 立即修复, B) 确认并仍然发布, C) 它是误报
- 如果用户选择 A:排队批准的修复,不要在此处编辑。在该修复通过审查和测试后,使用来自 greptile-triage.md 的 **Fix reply template**(内联差异 + 解释)并保存每项目/全局 greptile-history(类型: fix)。
- 如果用户选择 C:使用来自 greptile-triage.md 的 **False Positive reply template** 进行回复(包括证据),并保存到每项目和全局 greptile-history(类型: fp)。

**VALID BUT ALREADY FIXED:** 使用来自 greptile-triage.md 的 **Already Fixed reply template** 进行回复 — 无需 AskUserQuestion:
- 包括完成的内容和修复提交 SHA
- 保存到每项目和全局 greptile-history(类型: already-fixed)

**FALSE POSITIVE:** 使用 AskUserQuestion:
- 显示评论并说明你认为错误的原因(完全正确 或 [顶级] + 正文摘要 + 永久链接 URL)
- 选项:
  - A) 回复 Greptile 解释误报(如果是明显错误的建议)
  - B) 无论如何修复(如果是微不足道的)
  - C) 默默忽略
- 如果用户选择 A:使用来自 greptile-triage.md 的 **False Positive reply template** 进行回复(包括证据),并保存到每项目和全局 greptile-history(类型: fp)
- 如果用户选择 B:排队批准的修复,如上。

**SUPPRESSED:** 静默跳过 — 这些是来自先前分类的已知误报。

**分类后:** 如果已批准修复,请保存其批准和评论引用。
运行 Step 9 的完整审查/修复循环,然后返回此处。完成已保存的回复,而不再次询问已完成的修复,并对新评论进行分类。
没有排队更新的版本,继续到 Step 11。

---