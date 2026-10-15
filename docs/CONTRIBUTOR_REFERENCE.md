# 贡献者参考

仅贡献者的详细信息逐字从 CLAUDE.md 移出,Claude Code 将其加载到每个会话中,上限为 40,000 个字符(#2096,`test/claude-md-size.test.ts`)。CLAUDE.md 保留每个部分的一行指针。

## Skill 大小预算

**Token 上限:** 生成的 SKILL.md 文件超过 160KB(~40K tokens)时触发警告。这是一个"注意功能膨胀"的护栏,而不是硬性 gate。现代旗舰模型具有 200K-1M 上下文窗口,因此 40K 是窗口的 4-20%,而 prompt 缓存使较大 skill 的边际成本变小。该上限的存在是为了捕获失控的 preamble/resolver 增长,而不是强制对精心调整的大型 skill(`ship`、`plan-ceo-review`、`office-hours` 合法打包 25-35K tokens 的行为)进行压缩。如果你超过 40K,正确的修复通常是:(1)查看增长的内容,(2)如果一个解析器在单个 PR 中增加了 10K+,质疑它是否应该内联或作为参考文档,(3)仅将精心调整的散文压缩作为最后的手段 — 对覆盖范围审计、评审军或语音指令的削减有真正的质量成本。

第二个、更难的上限保护发现表面:`test/catalog-budget.test.ts` 将所有 skill 的 frontmatter `name` + `description` 聚合限制为 `CATALOG_BUDGET_TOKEN_EQUIVALENTS`(目前为 1,194;每个新 skill 都会增加它)token 等价物(每个 skill 260 字节子上限),通过 `test/helpers/skill-census.ts` 中的共享普查进行计数。这是强制执行的,而不是警告——每个主机在每个会话中加载完整目录,因此这里的增长会消耗每次对话。这个失败的 message 携带重新测量 + ratchet 协议。`bin/gstack-context-bill` 显示技能树的完整 token 材料清单(always-on 与 per-invocation、`--diff`、`--budget`;`--exact` 选择真实 tokenizer 并通过出口收据 POST 文件文本到 api.anthropic.com)。

上下文预算 ratchet(`test/context-budget-ratchet.test.ts`,免费,在 `bun run test` 中运行)在另外两个账本上钉死绝对上限:always-on FULL-frontmatter 聚合(catalog-budget 仅计算 name+description)和每个 skill 的 per-invocation eager token(SKILL.md + 强制读取的引用 — 大小下限和奇偶校验比率相对保护这些,而不是绝对的),针对 `test/fixtures/context-budget.json` 进行评级。增长超过其上限的 skill 失败;新 skill 失败,直到它被有意识地预算。对于合法增长或已落地的减少,重新运行 `bun test/helpers/capture-context-budget.ts` 并在同一提交中提交刷新的 fixture,以便上限向下 ratchet 并锁定每个胜利。

## 出口收据

**每个外部接收器的出口收据**(v1.63.0.0+)。gstack 发起的每次发送**必须**在发送之前将哈希链收据写入 `~/.gstack/security/egress.jsonl`:TypeScript 调用者使用 `lib/egress-receipt.ts` 中的 `writeReceipt`;shell 脚本 source `bin/gstack-egress-lib.sh` 并使用 `_receipted_curl` / `_receipted_git`。失败极性是按类划分的:对敏感接收器失败关闭(brain-sync、memory-ingest、gbrain-sync、telemetry、ngrok tunnels、mcp-verify、supabase-provision 以及 Memorable bridge 的每个 prompt memorable-recall 交接),对面向用户的失败打开 + stderr 警告(设计 OpenAI 调用、update-check、dashboards、git-class ops)。`test/egress-receipt-wiring.test.ts` 中的新接收器扫描器在未收据的 `curl` / `git push` / `fetch` 到非 loopback 主机上使 CI 失败,除非文件在其 `SCANNER_EXEMPT` 列表中带有合理的条目(用户定向页面获取、reachability probes、instruction strings、skill prose)——如果你添加新的外部接收器,请通过 helpers 连接它并将其添加到枚举的接收器列表中。`aside exec`(发送给 Aside 智能体的 gstack 组合提示)是一个面向用户的失败打开接收器:skills 通过 `scripts/resolvers/aside.ts` 渲染的 `_aside_exec` 包装器调用它,从不裸露。使用 `bin/gstack-egress`(`list` | `verify`,在篡改时退出 3 | `grants`)检查。威胁模型:对**尝试的**出口的法医可观察性,而不是渗出控制。

## Skill 链接和所有权

**前缀设置:** Setup 在顶层创建真实目录(不是符号链接),内部带有 SKILL.md 符号链接(例如,`qa/SKILL.md -> gstack/qa/SKILL.md`),以及指向每个 skill 的运行时资产的链接(sections/、templates、checklists — 除 SKILL.md、测试、构建输出和 `.tmpl` 源之外的所有内容)。别名 skill(`_gstack-command`、`connect-chrome`)安装为重写的副本,而不是符号链接。这确保 Claude 将它们发现为顶级 skill,而不是嵌套在 `gstack/` 下。名称要么简短(`qa`),要么命名空间化(`gstack-qa`),由 `~/.gstack/config.yaml` 中的 `skill_prefix` 控制。传递 `--no-prefix` 或 `--prefix` 以跳过交互式提示。

**所有权 gate(#2119):** `setup` 将 `.gstack-owned` 标记写入它创建的每个 skill 目录,而 `setup`(链接器、别名安装程序、两个前缀翻转清理以及退役技能修剪)和 `bin/gstack-relink` 仅在能够证明是 gstack 的情况下删除或链接到条目。强证据(解析为 gstack 的符号链接,或标记)允许删除或刷新整个目录。弱证据(与源字节相同的真实 SKILL.md,或带有 gen-skill-docs 两行横幅)仅涵盖该一个文件,而弱证据文件的差异在 gstack 链接覆盖之前移动到 `~/.gstack/backups/skills/<ts>/<skill>/SKILL.md`。任何其他内容都是外部 skill:跳过并在 setup 的最终摘要中命名。该规则存在于两个副本(`setup` 和 `bin/gstack-relink`)中;在共享帮助器在 TODOS.md 中落地之前保持它们同步。退役技能修剪(`_prune_stale_generated`)对不再存在的 skill 的渲染应用相同的强/弱分割,通过其自己的 gate(`_owned_for_windows_refresh`:真实的主机目录仅在其 SKILL.md 带有生成的横幅时才是候选;不咨询标记和字节同一性):它扫描渲染树和每个主机 skills 目录,删除真实渲染目录,仅当主机符号链接解析为 gstack 时才删除,仅当带有横幅的真实目录才通过 `_cleanup_weak_dir` 清理,从不跟随渲染树内的符号链接,并通过其 frontmatter `name:` 识别重命名的技能。由 `test/setup-link-ownership.test.ts`、`test/setup-cleanup-orphans.test.ts`、`test/setup-prune-stale-generated.test.ts` 和 `test/relink.test.ts` 固定。

## 修订保护

共享修订引擎在内容到达外部接收器(codex 派发、GitHub issue/PR 正文、推送的提交)之前捕获凭据、个人身份信息(PII)和法律/有害内容。它是一个**护栏,而不是密不透风的强制执行**——`git push --no-verify`、直接的 `gh issue create` 和 `GSTACK_REDACT_PREPUSH=skip` 都会绕过它。它捕获事故和粗心,99% 的情况。不要声称它阻止了有决心的泄露者(不会失败的 CHANGELOG 行将使敌意的屏幕截图失败)。

- **引擎 + 分类法:** `lib/redact-patterns.ts`(单一事实来源 — 3 个 tier;HIGH = 真正阻止的秘密凭据,MEDIUM = PII/法律/内部 + 高 FP 凭据形状通过 AskUserQuestion 确认,LOW = 仅供参考)和 `lib/redact-engine.ts`(纯 `scan()` + `applyRedactions()`)。校准很重要:一个哭泣狼的 gate 会被忽略,因此上下文变量形状(Stripe `pk_live_`、Google `AIza`、JWT、env `*_KEY=`)位于 MEDIUM。
- **CLI:** `bin/gstack-redact`(退出 0 clean / 2 MEDIUM / 3 HIGH;`--json`、`--auto-redact`、`--repo-visibility`、`--from-file`)。`bin/gstack-redact-prepush` 是可选的 git hook。
- **Skill 文档是从** `scripts/resolvers/redact-doc.ts`(`{{REDACT_INVOCATION_BLOCK:<sink>}}`)生成的,以便 /spec、/cso、/ship、/document-release、/document-generate 永远不偏离引擎。
- **扫描到接收器:** 始终扫描将要发送的确切字节 — 写入临时文件,扫描该文件,将**同一**文件传递给 `gh`/`git`。永远不要扫描字符串然后重新渲染(这会重新打开扫描与发送之间的差距)。
- **可见性(无 tier 提升):** 每次运行解析一次,顺序 = 本地配置(`gstack-config get redact_repo_visibility`,~/.gstack 因此从不提交)→ gh → glab → unknown(=public-strict)。公共仓库对每个发现获得**更严格**的确认(无批量确认,无静默继续);MEDIUM 永远不会被自动提升为 HIGH。
- **工具归属的 fence:** 将 Codex/Greptile/eval 输出包装在 ` ```codex-review ` / ` ```greptile ` fence 中,以便这些工具引用的示例凭据 WARN 降级而不是阻塞。fence 内的实时格式凭据仍然阻塞。
- **配置键:** `redact_repo_visibility`(public|private|unknown,gh/glab 无法读取的仓库的仅本地覆盖)、`redact_prepush_hook`(true|false)。故意**没有**禁用 HIGH 阻塞的键。
- **提交的允许列表:** 推送的提交根目录中的 `.gstack-redact-allowlist` 列出确切的匹配范围,被审查为良性;pre-push hook 仅抑制范围等于条目的发现,并在每次推送时列出每个抑制。
- **审计:** /spec 语义传递将无内容记录(类别 + body sha256,无 spec 文本)附加到 `~/.gstack/security/semantic-reviews.jsonl`(0600)。

## 从 garrytan-agents checkout PR

当用户说"checkout <PR link>"并且 PR 来自 `garrytan-agents/gstack`(或任何不是 `garrytan/gstack` 协作者的其他 fork),不要只是 `gh pr checkout`。Fork PR 不会收到基础仓库的秘密(`ANTHROPIC_API_KEY`、`OPENAI_API_KEY` 等),因此 eval/E2E CI 作业因空环境身份验证错误而失败,而不管在基础仓库上设置了什么。

**工作流:** 将分支推送到 `garrytan/gstack`(基础仓库),并从那里重新定位 PR。

具体来说,`gh pr checkout <N>` 后:

1. 记下原始 PR 编号和 head 分支名称。
2. 将同一分支推送到基础仓库:`git push origin HEAD:<branch-name>`(origin = `garrytan/gstack`,因为 worktree 是使用该远程设置的)。
3. 关闭 fork PR(`gh pr close <N> --comment "moving to base-repo branch for secret access"`)。
4. 从基础仓库分支打开新 PR:`gh pr create --base main --head <branch-name>`。
5. 新 PR 的工作流将自动获得秘密。

为什么不修复 fork 端?`garrytan-agents` 在 `garrytan/gstack` 上不是协作者。将其添加为协作者(选项 A)或翻转仓库范围的"将秘密发送到 fork PR"开关(选项 B)将允许秘密从任何人到达 fork PR — 影响范围比仅移动此分支更广。选项 C(本节)保持秘密分发范围紧凑。

如果用户要求你跳过移动(例如,"就让它作为 fork PR"),尊重 — eval CI 将因空环境身份验证而失败,但 check-freshness、workflow-lint 和 windows-tests 仍将在 fork PR 上通过。
