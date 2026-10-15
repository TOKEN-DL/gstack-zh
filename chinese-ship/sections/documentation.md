<!-- AUTO-GENERATED from documentation.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
# 文档审计闸门

仅商店发布在分发之前以 `read-only` 审计文档,无需分支闸门或源写入权限。

**尝试预算:** 初始审计加上一次调用记录中的修复/重新审计,永远不要第三次尝试,即使在 Step 16 之后。每次启动或内部增加启动之前递增,包括失败的启动;内联工作完成同样的验证闸门。过时的快照既不是新尝试也不是当前审计。保存子句柄。具有缺失输出的退出子级已停止,但其审计被阻止。

**进入:** 第一次进入始终启动初始审计。
在重新进入时,仅重用此调用的经验证审计闸门或命名风险决策,其接受的基/输入哈希仍匹配;保留其实际状态和范围。否则使用阻塞恢复,而非无条件启动。
重新进入从不重置计数或授权启动。

## 准备候选

1. 阅读已安装的 document-release SKILL.md 及其完整的 audit-scope/release-body 内容,链接为章节或内联以供外部主机使用。缺少章节或旧的 `Ship-owned documentation mode` 在之前阻止;永不替换。
2. 选择基础 SHA。仅商店发布将源/构建内容与已知的先前发布进行比较;如果不可用,则检查当前源并披露该限制。
   只读审计不得获取/合并。
3. 发现文档根/已编写模板,根据 audit-scope,并暂停其他写入者。
   一次调用将候选保存在产品树之外:
   `~/.claude/skills/gstack/bin/gstack-docs-candidate snapshot --out <private.json> --audit-id <fresh id> --mode <edit|read-only> --base <sha> --docs <root or generated output>...`
   (`--select <path>` narrows release paths)。它记录 HEAD、分支、索引、路径列表和发布路径、生成输出和文档的哈希,以 NUL 安全地读取。只读取新文件的内容;助手哈希其余的,子级审计它。用字面候选值填写提示符占位符。

## 启动审计

**作为子代理派发 /document-release**(永不 Skill)使用 Agent 工具,`subagent_type: "general-purpose"`。

**所需前台:** 当 Agent 调用上可用时传递 `run_in_background: false` — 自 Claude Code v2.1.198 起,子代理默认在后台运行,因此省略可用标志会提供后台运行。启动回执意味着它进入后台:等待其完成通知。仅通过 Agent 工具派发: 将目标调用为 Skill,或在其自己的上下文中内联执行其工作流,即使该技能可能出现在你可用的技能列表中,也会丧失此派发存在的新上下文隔离。(在某一步定义内联回退的地方,它仅在已派发的子代理失败后才应用。) 保留子项 id。

**子代理提示词:**

> Execute /document-release as a SPAWNED ship-owned subagent. Read `${HOME}/.claude/skills/gstack/document-release/SKILL.md` and its sections. Branch: `<branch>`, base: `<base>`. Candidate: `<candidate-path>`. Audit id: `<audit-id>`. Mode: `<mode>`.
>
> Prefix gstack-skill-start with `GSTACK_SESSION_KIND=spawned `. Report its actual `SESSION_KIND: spawned` echo, never prompt/file claims. Missing marker/inputs/assets blocks immediately.
>
> Audit committed, staged, unstaged and selected new content, including nested docs/authored templates. Follow audit-scope.md's discovery/permissions; read full files before editing. Execute only Steps 1–4 and 6; return doc health and completion.
>
> Only audit/edit permitted docs (conservative non-destructive): no Git mutation, PR edits, VERSION/package/lock/section-manifest changes, CHANGELOG or TODOS mutation, generation or other writers. `read-only` forbids source/doc edits. Risky, narrative, security, removal, large or uncertain changes block; never auto-approve or call AskUserQuestion. Preserve user content.
>
> Return one JSON object on the LAST nonempty line, without fences or trailing prose:
> - `schema_version`: integer 1; `audit_id`: the exact supplied string.
> - `status`: updated/current/blocked.
> - `files_updated`, `files_reviewed`, `blockers`, `decisions`: string arrays. Paths are unique repo-relative files, not globs.
> - `documentation_section`: nonempty Markdown with scope, result and debt, without a ## Documentation heading. No extra or legacy fields.
>
> Completed audits without blockers are `updated` if edited, otherwise `current`; describe scope even without docs. Failed/incomplete audits are `blocked`, with reasons/partial edits. Read-only corrections block. Metadata observations go only in decisions.

**父级处理:**

### 收集,然后验证

1. **收集。** 在约 10 分钟内检查子句柄的终端完成和最终输出。启动元数据缺失完成。在失败/截止时,在另一个写入者之前使用恢复。
2. **检查输出。** 仅解析最后非空行。需要每个字段/类型、确切的审计 id、模式、状态不变和上面回显的实际生成标记,如子级输出中所述;状态文件不是其证据。永远不要默认或重构缺失值。
3. **检查所有权。** 运行 `gstack-docs-candidate compare <candidate>`。
   将实际更改与候选进行比较,强制执行 prompt/audit-scope 权限和受保护文件排除。HEAD 和索引必须未更改,现有的脏/未跟踪用户内容保留,并且更改的路径与 `files_updated` 完全相同。拒绝任何只读写入。验证 `files_reviewed` 与事实范围和证据,而不是返回的声明。
4. **检查新鲜度。** 将保存的基础和输入哈希与当前内容进行比较。仅经验证的允许子级编辑可能不同。其他编辑或基础更改使审计变得过时,即使返回后。单独的父级提交不会使未更改的内容无效;不会跨调用重用审计。

### 继续或恢复

失败的检查或 `blocked` 结果转到恢复,即使使用有效的 JSON。
否则仅为 Step 16 仅子级之后的哈希保存(使用重组中的 `--out <audit-id>-post.json` 重新运行 Prepare 步骤;Step 16 `compare`s 它;永不键入哈希)、状态以及该部分作为私有 `<audit-id>-documentation.md` 的唯一内容原样复制。该文件是该部分的唯一来源;报告和 Step 19 通过命令(`cat`)在可能的位置插入,而不是重新输入或编辑。保存记录一次;按路径引用文件,从不复制其内容。
打印 `Documentation: updated` 带路径或 `Documentation: current` 带范围。
以后的更改需要剩余的重新审计或风险决策,而不是默默刷新的哈希。这不是说明;私下引用决策。仅父级阶段已批准的文件;Step 19 扫描并包括结果。

## 阻塞恢复

报告 `Documentation: blocked` 以及原因和实际路径。保留部分和现有内容以及被拒绝的输出。永远不要重置/清理、未打"醒用户文件、自动提交或推送意外的子级提交。

1. **在任何修复、重试、内部接管或其他写入者之前确认子级已停止。** 终端完成或确认终止就足够了。对于正在运行/未知的句柄,请求停止并检查其状态;仅请求不够。如果再过一个 5 分钟窗口后仍未确认,则停止 ship。拒绝来自已放弃 id 的延迟结果。
2. 如果尝试剩余且审计输入已更改或具体的启动/输入/权限更正或已审查的补丁修复可用,则在用户批准进行风险编辑的情况下应用任何修复。
   使用当前输入和新的遗保留/快照重复 Prepare,运行剩余的尝试,然后通过父级处理验证它。
3. 否则在提交/发布之前停止,不启动另一个子级。
   AskUserQuestion: 停止修复(推荐),或使用特定的命名文档风险发布。仅当选择继续进行实际用户异常才算数,从不是默认值、超时、推荐或更早/无关的批准。保存其范围/内容;报告和 PR 保留阻塞状态、不完整的范围以及任何保留或排除的部分更改。未确认的写入者、所有权违规、未授权的提交或编辑以及编辑/安全闸门不能被豁免。
   在继续之前调和这些。