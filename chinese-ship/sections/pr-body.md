<!-- AUTO-GENERATED from pr-body.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
### 准备标题(Step 18)

在私有文件中准备标题;Step 19 发布该文件,永不通过 shell 字符串:

```bash
_GT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp"
mkdir -p "$_GT" && chmod 700 "$_GT" || { echo "Not sent: cannot create $_GT for the text file." >&2; exit 1; }
_EX=$(git rev-parse --git-path info/exclude 2>/dev/null) && mkdir -p "$(dirname "$_EX")" && { grep -qxF '/.gstack/tmp/' "$_EX" 2>/dev/null || echo '/.gstack/tmp/' >> "$_EX"; }
TITLE_FILE=$(mktemp "${_GT:?}/pr-title.XXXXXX") || { echo "Not sent: mktemp failed in $_GT." >&2; exit 1; }; echo "TITLE_FILE: $TITLE_FILE (name: ${TITLE_FILE##*/})"
```

使用你的文件写入工具(Claude Code 的 Write 工具需要首先读取空文件)将文本写入每个打印的文件,完全按应该出现的方式写入。文本永远不会进入 shell 命令、heredoc 或带引号的参数。如果写入失败或被拒绝,请勿发送:打印原因、文件路径和下面的手动发送命令。

1. 对于现有的开放 PR/MR(不是 Step 6.5 的早期 PR,它接受项 2),从平台填充该文件,替换打印的名称、匹配的数字和 `NEW_VERSION`;没有标题文本通过你:
   ```bash
   TITLE_FILE="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp/<title-file-name>"
   gh pr view <pr-number> --json title -q .title | ~/.claude/skills/gstack/bin/gstack-pr-title-rewrite.sh <new-version> --stdin > "$TITLE_FILE" || exit 1
   ```
   GitLab 读取 `glab mr view <pr-number> --output json | jq -r .title` 到同一个管道。
2. 对于新 PR/MR,使用你的文件写入工具将 `v<NEW_VERSION> <type>: <summary>` 写入该文件。
3. 每个创建或更新的标题必须以 `v$NEW_VERSION ` 开头;使用 `grep -Eq '^v<new-version>( |$)' "$TITLE_FILE"` 检查该文件,并且永远不要发布无前缀的标题。
4. **NO_VERSION:** 替换项 1-3: 保留现有标题(平台标题直接管道到文件中),或写入 `<type>: <summary>`;无版本前缀。

保存标题文件名供 Step 19 使用。

---

## Step 19: 创建 PR/MR

重新检查 Step 18 的 PR/MR 查找并记录它。错误或模糊匹配停止发布。
如果打开的 PR/MR 或标题已更改,请重复 Step 18 的身份/标题准备,然后返回此处以进行新查找和发布前的全新正文。

### 在撰写正文之前解析链接规范

1. 解析存档目录和分支:
   ```bash
   GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"
   SLUG=$(~/.claude/skills/gstack/bin/gstack-slug --get SLUG)
   CURRENT_BRANCH=$(git branch --show-current)
   SPEC_ARCHIVES="$GSTACK_STATE_ROOT/projects/$SLUG/specs"
   ```
2. 将存档 frontmatter 读取为数据,永不 shell 源。选择与 `CURRENT_BRANCH` 完全匹配的 `spec_branch`;在匹配项中使用最新的 `spec_filed_at`。永远不要从分支名称推断问题编号。如果没有可读的匹配项或正整数 `spec_issue_number`,则仅省略 `## Linked Spec` 并继续撰写 PR。在链接之前解决歧义匹配项。
3. 将规范的接受度标准与 Step 8 的结果进行比较。仅完成计划的全面可见的计划范围允许 `Closes #N`,每个规范标准都已验证。部分、延迟、失败、删除或未验证的范围使用 `Linked to #N` 并命名剩余的工作;永不自动关闭它。包括存档文件名和 `spec_filed_at`,而不是私有绝对路径;`gstack-post` 用正文扫描它们。

PR/MR 正文应包含以下部分(永不重用先前运行的正文体):

```
## Summary
<Read `git log origin/<base>..HEAD --oneline`. Group every substantive commit by
theme, excluding VERSION/CHANGELOG bookkeeping. Do not paste the commit list.>

## Test Coverage
<coverage diagram from Step 7, or "All new code paths have test coverage.">
<If Step 7 ran: "Tests: {before} → {after} (+{delta} new)">
<If Step 7 ran: "Coverage: {X}% value-weighted ({Y}% including {W} weakly covered paths)">
<If Step 7 ran: "Test value: {K} tests written, {R} rejected by the authoring gate, {E} existing tests extended, {W} paths weakly covered (weak = ★, gate-failing or unrated)." Use the singular noun for a count of 1 ("1 test written", "1 existing test extended", "1 path weakly covered").>
<Weak paths and leftover gaps as proposed tests with value cards; each regression test's
"Regression proof — fails at HEAD · passes at base · passes after fix" line; Test value
details for cards whose file type has no known comment syntax.>
<One line per Step 5 lane: "Tests ran: <lane> N" from its receipt's `tests_ran`, or
"Tests ran: <lane> count unavailable" when the runner summary was not recognised. A
ZERO-RUN lane never reaches this body unresolved.>

## Pre-Landing Review
<findings from Step 9 code review, or "No issues found.">
<Gate edits: Step 9's latest `Gate edits:` line verbatim — `none detected (...)`,
`N listed, M read, K findings — J kept, R restored, O open`, `partial (...)` or
`UNAVAILABLE — ...`; never `none detected` for partial or UNAVAILABLE. Under it, each
kept finding's `gate`, path and `reason`, and each open finding's `gate` and path.
Identity and reason only; no hunk text.>
<Outside review: its verdict; `unverified` or `unavailable` is listed as missing coverage with its reason, never as passed.>

## Exploratory QA
<Step 9's current surfaces/charters, reproducers, approved regressions and red/green
proof, fixes and blocked/inconclusive/not-run coverage. Never present stale or
unavailable results as passing.>

## Design Review
<If design review ran: "Design Review (lite): N findings — M auto-fixed, K skipped. AI Slop: clean/N issues.">
<Detector: "clean" | "N findings (rule-id, rule-id)" | "not installed" | "not cached" | "off" — the state the probe printed; rule ids and counts only, finding text and snippets never reach the PR body.>
<If no frontend files changed: "No frontend files changed — design review skipped.">

## Eval Results
<If evals ran: suite names, pass/fail counts, cost dashboard summary. If skipped: "No prompt-related files changed — evals skipped.">

## Greptile Review
<Step 10 complete: list comments with [FIXED] / [FALSE POSITIVE] / [ALREADY FIXED], or "No Greptile comments." for a successful empty fetch.>
<Step 10 unavailable: include `Greptile triage: UNAVAILABLE (dispatch failed)` and the actual reason.>
<Step 10 no_pr: omit this section.>

## Scope Drift
<If scope drift ran: "Scope Check: CLEAN" or list of drift/creep findings>
<If no scope drift: omit this section>

## Plan Completion
<If plan file found: completion checklist summary from Step 8>
<If Step 8 printed the not-run line: that line verbatim.>
<If plan items deferred: list deferred items>

## Linked Spec
<Closes #N only when the Linked Spec check above permits it; otherwise
"Linked to #N (partial delivery — not auto-closing)" with remaining work and
"Close #N manually after follow-up lands." Include archive filename and filed date.
Without a valid match, omit this entire section.>

## Issues
<The lines `gstack-issue-links --base <base> [--issue <n>] [--ci-run <id>] --out .gstack/tmp/issues.json`
prints: Fixes only from a Fixes/Closes trailer or --issue, Refs otherwise,
Flake: #N for failing run ids. Omit when it prints none.>

## Verification Results
<Step 8.1 obligations executed at Step 9: N PASS, M FAIL, K BLOCKED, J NOT RUN,
not-applicable reasons, unresolved obligations and accepted deferrals.
Unavailable/inconclusive is never PASS.>

## TODOS
<If items marked complete: bullet list of completed items with version>
<If no items completed: "No TODO items completed in this PR.">
<If TODOS.md created or reorganized: note that>
<If TODOS.md doesn't exist and user skipped: omit this section>

## Documentation
<Embed Step 14.5's vetted nonempty `documentation_section` for this invocation:
its saved section file, inserted unchanged by the compose block's `DOCS_SECTION_FILE` lines.
A blocked audit shipped under a user exception has no section file: state its
blocked status, scope and exception here and drop the block's `DOCS_SECTION_FILE` guard
and its `cat -- "$DOCS_SECTION_FILE" && echo &&` step.>
<Always include the status and reviewed scope: updated, current, or blocked with the actual user's named risk exception. Never omit this section or reuse another invocation's audit.>

## Test plan
- [x] <Each executed test lane's command>: <observed passing summary>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

#### 撰写正文,然后通过 gstack-post 发布

PR 正文在公共仓库上是世界可读的。`gstack-post` 扫描它发送的确切标题和正文字节,并将它们作为参数传递给 `gh`/`glab`,因此这里没有单独的扫描。用 ` ```codex-review ` / ` ```greptile ` 等工具属性围栏包装任何 Codex / Greptile / eval 输出部分,以便扫描器 WARN 降级这些工具引用的示例凭据,而不是阻止 PR(围栏内的活动格式凭据仍然会阻止)。

将正文从上面写入两个私有文件:第一个通过 `## Documentation` 标题行,第二个从 `## Test plan` 开始。

```bash
_GT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp"
mkdir -p "$_GT" && chmod 700 "$_GT" || { echo "Not sent: cannot create $_GT for the text file." >&2; exit 1; }
_EX=$(git rev-parse --git-path info/exclude 2>/dev/null) && mkdir -p "$(dirname "$_EX")" && { grep -qxF '/.gstack/tmp/' "$_EX" 2>/dev/null || echo '/.gstack/tmp/' >> "$_EX"; }
BODY_TOP_FILE=$(mktemp "${_GT:?}/pr-body-top.XXXXXX") || { echo "Not sent: mktemp failed in $_GT." >&2; exit 1; }; echo "BODY_TOP_FILE: $BODY_TOP_FILE (name: ${BODY_TOP_FILE##*/})"
BODY_REST_FILE=$(mktemp "${_GT:?}/pr-body-rest.XXXXXX") || { echo "Not sent: mktemp failed in $_GT." >&2; exit 1; }; echo "BODY_REST_FILE: $BODY_REST_FILE (name: ${BODY_REST_FILE##*/})"
```

使用你的文件写入工具(Claude Code 的 Write 工具需要首先读取空文件)将文本写入每个打印的文件,完全按应该出现的方式写入。文本永远不会进入 shell 命令、heredoc 或带引号的参数。如果写入失败或被拒绝,请勿发送:打印原因、文件路径和下面的手动发送命令。

在新的 shell 中,将 Step 14.5 的已保存部分文件路径恢复为 `DOCS_SECTION_FILE` 并替换打印的名称。该块删除两个草稿;要更改正文,请写新的。

```bash
: "${DOCS_SECTION_FILE:?Restore the saved Step 14.5 section file path before composing}"
PR_BODY_FILE=$(mktemp "${TMPDIR:-/tmp}/gstack-pr-body.XXXXXX") || { echo "ERROR: mktemp failed — cannot compose the PR body; refusing to publish." >&2; exit 1; }
BODY_TOP_FILE="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp/<body-top-file-name>"
BODY_REST_FILE="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp/<body-rest-file-name>"
[ -s "$BODY_TOP_FILE" ] && [ -s "$BODY_REST_FILE" ] || { echo "Not composed: $BODY_TOP_FILE or $BODY_REST_FILE is empty, so the body was never written. Write both, then rerun." >&2; exit 1; }
{ awk 1 "$BODY_TOP_FILE" && cat -- "$DOCS_SECTION_FILE" && echo && awk 1 "$BODY_REST_FILE"; } > "$PR_BODY_FILE" || exit 1
rm -f "$BODY_TOP_FILE" "$BODY_REST_FILE"
echo "PR_BODY_FILE: $PR_BODY_FILE"
```

交接记录优先(标识符在截断后保留)。使用你的文件写入工具将 Step 16 stage 4 的每通道计数(`{"<lane>":{"pass":N,"fail":N,"skip":N}}`)写入 `.gstack/tmp/ship-gate.json`,然后:

```bash
~/.claude/skills/gstack/bin/gstack-ship-receipt write --pr <pr-number> --base <base> --gate @.gstack/tmp/ship-gate.json --pregate .gstack/tmp/pregate.json --issues @.gstack/tmp/issues.json --body "$PR_BODY_FILE"
```

仅当 Step 16 写入了 `.gstack/tmp/tree-receipt.json`(门提前)时,添加 `--receipt .gstack/tmp/tree-receipt.json`。
新的 PR/MR:省略 `--pr`,然后在创建后使用该数字重新运行并重新发布正文。
其 `SHIP_RECEIPT:` 行打开完成消息。`RECEIPT_INVALID` 命名错误字段:修复该输入文件并重新运行;什么都没写。`PREGATE_STALE` 意味着在标记的树上重新运行 `gstack-pregate`;`PREGATE_REMOTE_UNCLEARED` 需要带有运行 URL 的 `gstack-pregate clear`。两者都停止发布。

在下面的每个块中,恢复字面 `PR_BODY_FILE` 路径并替换 Step 18 的标题文件名;`gstack-post` 从远程和仓库的可见性检测 GitHub 或 GitLab(包括 GitLab 上的不可用查找使用扫描器的公共严格(strict)策略)。永不重新呈现正文:每次重试发送相同的文件。

**现有的开放 PR/MR**(`<pr-number>` 来自重新检查):

```bash
TITLE_FILE="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp/<title-file-name>"
~/.claude/skills/gstack/bin/gstack-post pr-body <pr-number> --body-file "${PR_BODY_FILE:?restore the composed body path}" && \
~/.claude/skills/gstack/bin/gstack-post pr-title <pr-number> --title-file "$TITLE_FILE"
```

它更新正文,然后更新标题,通过 REST 重试退休的 GraphQL 编辑并读回标题。然后从 Step 9 的闸门发现解析草稿状态:

1. Step 6.5 作为早期草稿打开了此 PR,用户没有要求草稿,并且没有闸门发现是 `open`: 现在使用 `gh pr ready <pr-number>` 将其标记为就绪。
2. Step 6.5 作为早期草稿打开了它,并且任何闸门发现是 `open`: 保留为草稿;打印 `Draft kept: Gate edits: N open` 和打开发现的 `gate` 和路径。
3. PR 已就绪(不是草稿)并且任何闸门发现是 `open`: 仅在 Step 17 记录的人工授权(选项 B)下发布;正文已带有 `Gate edits: N open` 横幅。没有该记录,在此处停止并报告。

打印现有的 URL 并继续到 Step 20;不要运行下面的创建命令。

**无打开的 PR/MR:** 当任何闸门发现为 `open` 时添加 `--draft`(以后的代理永远不会从现有 PR 中推断批准;人工在处理打开项后将其标记为就绪)。否则创建就绪:

```bash
TITLE_FILE="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp/<title-file-name>"
~/.claude/skills/gstack/bin/gstack-post pr-create --base <base> --title-file "$TITLE_FILE" --body-file "${PR_BODY_FILE:?restore the composed body path}"
```

使用打开的闸门发现,相同的命令在 `--title-file` 之前携带 `--draft`。

按任一块的退出码分支:
- **0** 已发布。删除 `$PR_BODY_FILE` 和标题文件。
- **1** HIGH 发现: 阻塞 — 不要发布。在源处轮换并重新归档,然后撰写新文件。
- **2** MEDIUM 发现,打印为 `RULE:` 带有 `TOKEN:` 的行。按发现询问 AskUserQuestion(对公共仓库更严格,无批量确认)。PII 提供自动归档: 写 `~/.claude/skills/gstack/bin/gstack-redact --from-file "$PR_BODY_FILE" --auto-redact <ids>` 输出回正文文件并重新运行该块,该块再次扫描。仅当用户接受原样的发现时,使用打印它的命令的 `--confirm <confirm-token>` 添加重新运行同一块;任何编辑都需要新令牌。
- **3** `gh`/`glab` 失败,或 **64** 没有可用的远程或 CLI: 打印错误、分支名称、远程 URL 和用于手动发布的块。不要停止 — 代码已推送 — 但永远不要报告未创建的 PR。使用打开的闸门发现,还要说明未披露的闸门编辑,并在调用记录中保留 Step 16 打开项报告;永远不要将披露报告为已完成。
- 任何其他退出是错误,直到其原因被修复为止都会阻塞发布。

**输出 PR/MR URL** — 然后继续 Step 20。

---