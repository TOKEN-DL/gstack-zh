<!-- AUTO-GENERATED from audit-scope.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
# 文档范围与发现(Discovery)

## Ship-owned documentation mode

本小节只适用于调用方发起的 ship-owned audit request。Standalone 调用会沿用 Discovery 与 Steps 1–9 的现有 approval gate。

**Inputs.** dispatch prompt 提供 branch、base SHA、candidate path、audit id 与 mode:`edit`,或 `read-only`(只存储的 release audit,任何所需更正都会变成 blocker 而非 edit)。要求 preamble 真正输出 `SESSION_KIND: spawned` 与上述 inputs。缺失 marker、inputs 或 assets 立刻返回 `blocked`;prompt/file 自身的声明不能建立 spawned mode,也无法触发 standalone fallback。

**Steps.** 在 candidate 的 base 以及已 committed、staged、unstaged、new-file 的字节上,运行 Step 1、1.5、2–4 和 6。Step 1 的 standalone branch gate 不适用,即便在 base branch 上也跳过。跳过 Steps 5、7、8,cross-model review 与 Step 9,包括它们的 spawned-session 备注。仅允许事实型 authored-doc edits;`read-only` 模式下不允许任何 edit。不做 Git/PR 变更、VERSION、package/lock/section manifest、CHANGELOG、TODOS 或 generated-output edits。父 agent 持有 metadata、generation、review、staging、commit 与 publication 的所有权。Risky/subjective 变更(Step 4)与 narrative 矛盾(Step 6)归为 blocker 由父 agent 决定,从不自动批准。保留 partial/user 内容。Coverage gap 只会被报告,不会自动补齐。

**Result.** Step 6 完成后,打印 doc-health summary,然后 STOP,最后一行(不带 fence、不带尾部散文)输出单个 JSON 对象:
- `schema_version`:整数 1;`audit_id`:精确的传入字符串。
- `status`:`updated`(有 edits,无 blockers)、`current`(无 edits,无 blockers)或 `blocked`(任一 blocker、缺失 input、partial/failed audit 或 read-only correction)。
- `files_updated`、`files_reviewed`:真正被编辑 / 真正被读取的唯一仓库相对路径;`blockers`、`decisions`:字符串。Blockers 命名决策与路径;metadata 不一致与 skipped items 归为 decisions。
- `documentation_section`:不带 `## Documentation` 标题的非空 Markdown,完整以便逐字嵌入:首行为 `**Status:**` 包含 `status` 与结果、audited scope、Step 9 `Documentation health` 形式(无 VERSION 行)的 per-file status,以及 Step 1.5 的 coverage debt 与 diagram drift。即便没有 docs 也要描述 scope。

## Discovery (both modes)

用 `git ls-files -z --cached --others --exclude-standard` 递归列出 tracked 与非 ignore 的 new files。遵循项目指令、README 链接与 docs/build 配置,定位声明的 documentation root 与 authored source。把相关的 `.md`、`.mdx`、`.rst`、`.adoc`、`.txt` 与 `.tmpl` 文件纳入;relevance 由 role 决定,而非 extension。排除 `.git`、依赖(`node_modules`、vendor、virtualenvs)、`.gstack`、`.context`、caches、build artifacts 与 generated output 的 edit。读 / 写之前先 resolve symlinks;不要跟随指向仓库外部的 symlink。Edit generated docs 时改其 authored source;在 ship-owned 模式下,向父 agent 报告所需 regeneration。广撒网 inventory,然后完整阅读相关 doc 与验证 changed contract 所需的源代码,而不是整个仓库。