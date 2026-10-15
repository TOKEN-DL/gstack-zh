<!-- AUTO-GENERATED from greptile-early.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## Step 6.5: 早期 Greptile PR(在免费测试通过后)

当仓库使用 Greptile 时,现在推送并打开 PR,以便 Greptile 在 Step 7-9 运行时进行审查;Step 10 收集其评论。没有 Greptile 一切都不会改变。

```bash
~/.claude/skills/gstack/bin/gstack-greptile-early detect
```

- **`GREPTILE_EARLY: off`** — 打印原因(以及任何 turn-back-on 行)并继续到 Step 7。
- **`GREPTILE_EARLY: ask`**(公共仓库,或第一次未知可见性) — AskUserQuestion:
  > This repo is public. Early Greptile review pushes this branch and opens a draft PR
  > before /ship's own review, so the code is visible before that review finishes. If
  > /ship stops later, the draft stays up with a comment saying why.

  A) 早期推送此仓库(已记住) · B) 保持今天的此仓库顺序(已记住)。
  运行 `~/.claude/skills/gstack/bin/gstack-greptile-early consent yes`(A)或 `consent no`(B)。
  B 继续到 Step 7。
- **`GREPTILE_EARLY: on`**,或在 A 之后 — 向用户展示打印的 `Greptile: found …` 行,然后按照下面的 1-3。

**1.** 推送: `git push -u origin <branch-name>`。如果失败,请记录 "Greptile early review: not run (push failed)" 并继续到 Step 7;Step 17 稍后推送。

**2.** 将临时 `<type>: <summary>` 标题(Step 18 替换它)写入私有文件:

```bash
_GT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp"
mkdir -p "$_GT" && chmod 700 "$_GT" || { echo "Not sent: cannot create $_GT for the text file." >&2; exit 1; }
_EX=$(git rev-parse --git-path info/exclude 2>/dev/null) && mkdir -p "$(dirname "$_EX")" && { grep -qxF '/.gstack/tmp/' "$_EX" 2>/dev/null || echo '/.gstack/tmp/' >> "$_EX"; }
TITLE_FILE=$(mktemp "${_GT:?}/early-title.XXXXXX") || { echo "Not sent: mktemp failed in $_GT." >&2; exit 1; }; echo "TITLE_FILE: $TITLE_FILE (name: ${TITLE_FILE##*/})"
```

使用你的文件写入工具(Claude Code 的 Write 工具需要首先读取空文件)将文本写入每个打印的文件,完全按应该出现的方式写入。文本永远不会进入 shell 命令、heredoc 或带引号的参数。如果写入失败或被拒绝,请勿发送:打印原因、文件路径和下面的手动发送命令。

**3.** 打开草稿,替换打印的名称。仅在用户要求非草稿 PR 时添加 `--ready`:

```bash
TITLE_FILE="$(git rev-parse --show-toplevel 2>/dev/null || pwd)/.gstack/tmp/<title-file-name>"
~/.claude/skills/gstack/bin/gstack-greptile-early open --base <base> --title-file "$TITLE_FILE"
```

它通过 `gstack-post` 打开 PR,其正文说明 /ship 早期打开了它,并在 Greptile 跳过草稿时发布一个 `@greptileai` 评论。在调用记录中保存打印的 `EARLY_PR` 和 `EARLY_PR_OPENED_AT`。`EARLY_PR: none` 表示已打开 PR:Step 10 像以前一样对其进行分类。退出 1 或 2 表示 `gstack-post` 标记了临时标题:写一个更简单的并重新运行。退出 3:记录错误并继续而不进行早期审查。

**如果 /ship 在早期 PR 打开后停止,在 Step 19 发布之前**,保持草稿打开(永不关闭或强制推送)并发布一条评论说明 /ship 停止以及原因:使用你的文件写入工具将原因写入私有文件,然后运行 `~/.claude/skills/gstack/bin/gstack-post pr-comment <pr-number> --body-file <that file>`。

---