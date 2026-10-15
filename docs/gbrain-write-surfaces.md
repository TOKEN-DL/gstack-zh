# gbrain 写入表面 — 内容落在哪里,以及如何验证

本文档服务于两个受众:

1. **智能体**:当规划 skill 渲染紧凑的 `## Brain Context Load` 或 `## Save Results to Brain` 块时,这些块引用本文档。当你实际使用 gbrain 时按需阅读 §Context Load 或 §Save Template。如果 `gbrain` 不在 PATH 上,请完全跳过。
2. **人类**:在针对真实 brain 运行规划 skill 后,使用手动探针部分确认页面确实已落地。

## 内容落在哪里

| 主机 + 检测状态 | 在规划-skill SKILL.md 中呈现的内容 |
|---|---|
| 任何主机 + `gstack-config gbrain-refresh` 报告 `gbrain_local_status: "ok"` | 呈现压缩的 brain-aware 块。智能体在实际保存时按需阅读本文档。每个规划 skill 约 250 个 token 开销。 |
| 任何主机 + gbrain 未检测到 | 在 gen-time 抑制块。零 token 开销。校准任务仍会渲染(单独的解析器,与主机无关)。 |
| GBrain 或 Hermes 主机 | 无论检测如何,块始终呈现 — 这些主机将 gbrain 集成作为一等公民提供。 |

`.gbrain-source` 为从该目录进行的**读取和写入**选择源,包括 `gbrain put`。它不选择存储引擎:引擎在 `~/.gbrain/config.json` 中单独配置。GBrain 首先解析显式 `--source`,然后 `GBRAIN_SOURCE`,然后最近的受信任 `.gbrain-source`,然后是已注册的路径和默认源规则。如果页面落在意外的源中,请在将成功的 `put` 视为已到达预期语料库的证据之前检查有效的源选择。

信任策略(`personal` 与 `shared`,按 endpoint 哈希)控制自动推送和写回。通过 `gstack-config set brain_trust_policy@<endpoint-hash> personal` 设置。本地 PGLite 安装自动默认为 `personal`;远程 MCP 安装在 `/setup-gbrain` 步骤 9.5 期间提示。

## §Context Load(智能体在运行规划 skill 时阅读此内容)

在开始之前,在 brain 中搜索相关上下文:

1. **提取 2-4 个关键词** 从用户的请求中。挑选名词、错误名称、文件路径、技术术语 — 而不是动词或形容词。例如:对于"登录页面在部署后损坏",搜索 `login broken deploy`。
2. **搜索**:`gbrain search "<keyword1 keyword2>"`。返回类似 `[slug] Title (score: 0.85) - 内容第一行...` 的行。
3. **如果结果很少**(少于 3 个):扩展到单个最具体的关键词并再次搜索。如果仍然很少,请在没有 brain 上下文的情况下继续。
4. **阅读前 3 个结果**:`gbrain get "<slug>"` 获取每个。3 个后停止 — 之后收益递减。
5. **使用上下文** 来通知你的分析。当 brain 页面改变了你的想法时,在输出中引用特定的 slug。

如果 `gbrain search` 返回任何非零退出(gbrain 不在 PATH 上、网络抖动、节流),则视为瞬态:在没有 brain 上下文的情况下继续。不要内联重试 — 用户可以稍后重新运行该 skill。

## §Save Template(智能体在实际保存时阅读此内容)

完成 skill 后,保存输出。紧凑的解析器块已经显示了你的特定 skill 的 slug 前缀 + 标题 + 标签(例如 `gbrain put "ceo-plans/<feature-slug>" ...`)。页面是自由格式文本,因此它永远不会进入 shell 命令、heredoc 或带引号的参数:在仓库根目录创建一个私有文件,使用你的文件写入工具将页面写入其中,然后让 `gbrain put` 从 stdin 读取它。

```bash
mkdir -p .gstack/tmp && mktemp .gstack/tmp/page.XXXXXX
```

将页面按原样写入打印的文件:

```markdown
---
title: "<Title>: <feature name>"
tags: [<tag>, <feature-slug>]
---
<skill output in markdown — the actual deliverable, not a summary>
```

然后保存它,将打印的路径替换为 `<page-file>`:

```bash
gbrain put "<slug-prefix>/<feature-slug>" < "<page-file>" && rm -f "<page-file>"
```

如果文件写入失败或被拒绝,请不要保存:报告原因并为用户保留输出。

从**相同的工作目录和帐户**读取页面,然后将其报告为已保存:

```bash
gbrain get "<slug-prefix>/<feature-slug>"
```

确认返回的页面包含预期的标题和实质性输出。如果读取失败或返回另一个页面,请将保存报告为未验证并保留输出以供重试;不要声称为持久结果。同样的规则也适用于实体存根。

**Slug 指导**:`<feature-slug>` 应为 kebab-case(短横线连接的小写),并且在前缀内唯一。选择具体的项目/功能名称而不是抽象标签。例如:`auth-rate-limit` 而不是 `security-fix`。

**标题指导**:常量前缀(例如 "CEO Plan"、"Eng Review")是固定的;后缀是功能/主题的人类可读名称。

**标签指导**:第一个标签是 skill 元数据中的常量 `<tag>`(例如 `ceo-plan`、`eng-review`)。第二个标签是 `<feature-slug>`,以便跨页面遍历有效。如果明显的关系存在(例如 `[ceo-plan, auth-rate-limit, security]`),添加更多标签。

### 实体存根丰富

保存主页后,从输出中提取提到的人名和组织名称。对于每一个:

```bash
# Check if a page exists first
gbrain search "<entity name>"

# If no match, create a stub from a private file you write with your
# file-write tool (same pattern as the page above)
mktemp .gstack/tmp/entity.XXXXXX
gbrain put "entities/<entity-slug>" < "<entity-file>" && rm -f "<entity-file>"
```

存根文件包含:

```markdown
---
title: "<Person or Company Name>"
tags: [entity, person]
---
Stub page. Mentioned in <skill name> output. Replace with real bio when relevant.
```

**仅提取真实名称** — 实际的人名(例如 "Garry Tan")和公司/组织名称(例如 "Y Combinator")。跳过产品名称、功能名称、章节标题、技术术语(CSS 类名、函数名)和文件路径。如有疑问,请跳过。

`tags: [entity, person]` 用于人员,`tags: [entity, organization]` 用于公司/团队。

### 错误处理

- **节流**:退出代码 1,stderr 包含 `throttle`、`rate limit`、`capacity` 或 `busy`。推迟保存并继续 — brain 忙;内容不会丢失,只是本次运行未持久化。
- **任何其他非零退出**:视为瞬态失败。不要内联重试 — 用户可以稍后重新运行该 skill 或运行 `gstack-config gbrain-refresh`(如果他们怀疑 gbrain 本身配置错误)。
- **`gbrain: command not found`**:gbrain 不在 PATH 上。紧凑的解析器块告诉你跳过 — 你不应该到达此代码。如果你确实到达了,请静默跳过并继续。

### 反向链接

如果你的保存输出通过名称或主题提及另一个 brain 页面,请在 markdown 主体底部添加反向链接行:

```
Related: [[other-page-slug]], [[another-slug]]
```

gbrain 自动将 `[[slug]]` 语法解析为渲染页面中的可点击链接。仅在关系具体时添加反向链接(例如"此 CEO 计划依赖于 `eng-reviews/auth-rate-limit` 处的 eng 评审")。不要编造连接。

### 完成摘要

在最终的 skill 输出中,用一行注明 brain 利用情况:"Brain: read 3 pages, saved and read back 1 page, enriched 2 entity stubs, 0 throttles."。仅在其读回成功后,才将页面计为已保存。这有助于用户看到 brain 覆盖率随时间增长。

## 持久性验证(自动化)

"我们希望保存的数据是否真的被保存?"的匹配对问题由 `test/skill-e2e-gbrain-roundtrip-local.test.ts` 涵盖:针对隔离的临时 HOME 进行真实的 `gbrain init --pglite` + `gbrain put` + `gbrain get` 往返。周期性。当 `VOYAGE_API_KEY` 未设置或 gbrain CLI 缺少 PATH 时跳过。

在打开触及解析器的 PR 之前运行它:

```bash
EVALS=1 EVALS_TIER=periodic VOYAGE_API_KEY=$VOYAGE_API_KEY \
  bun test test/skill-e2e-gbrain-roundtrip-local.test.ts
```

如果你确实要在自己的 brain 上对真实规划 skill 运行后进行手工抽查(调试智能体本应保存的特定页面):

```bash
gbrain get "<prefix>/<slug>"           # expect markdown + frontmatter
gbrain search "<slug fragment>"        # expect slug in top results
gbrain sources list                    # confirm gstack-brain-<user> source
gbrain get "entities/<person>"         # expect stub per named person
```

## 远程 / Supabase / 精简客户端 MCP 路由

解析器发出单个 CLI 形状 — `gbrain put "<slug>" < "<page-file>"`,stdin 上的页面 — 可与 gbrain 支持的每个引擎一起使用。CLI 根据用户的 `~/.gbrain/config.json` 在内部路由到本地 PGLite、远程 Supabase 或远程 MCP 端点。**gstack 不测试该路由**:存储层是 gbrain 要遵守的合约,而我们针对本地 PGLite 测试的相同 CLI 调用是针对任何其他引擎触发的调用。

如果你在 Supabase 或精简客户端 MCP 上并且写入未落地:

1. `gbrain doctor --fast --json` — 引擎健康检查。如果任何内容报告 `error`,请先修复。
2. `gstack-config get brain_trust_policy@<endpoint-hash>` 必须是 `personal` 才能自动写入。运行 `gstack-config endpoint-hash` 以获取活动哈希。如果是 `shared`,智能体在写入前会提示 — 如果你拒绝,请重新运行 skill。
3. 如果信任策略是 `personal` 并且 `gbrain doctor` 是干净的,但页面仍不存在,请针对 gbrain 提交问题 — gstack 的 CLI 调用形状与 T11(`gbrain-roundtrip-local`)练习的相同。

## 自动化未验证的内容

- **校准任务(`takes_add`)**:今天这些回退到 `gbrain put` 内的 fence-block 写入,因为 `BRAIN_CALIBRATION_WRITEBACK` 在 gbrain v0.42+ 发布 `takes_add` MCP 操作之前为 FALSE。当标志翻转时,针对 `/office-hours` 重新运行本文档中的探针,并确认 `gbrain takes_list` 显示一个 `kind=bet` 条目,其具有预期的权重(`scripts/brain-cache-spec.ts:151-157` 中的 office-hours 为 0.9)。
- **其他 4 个规划 skill 的每个 skill E2E**:只有 `/office-hours` 具有假 CLI E2E 覆盖(`test/skill-e2e-office-hours-brain-writeback.test.ts`)。解析器单元测试(`test/resolvers-gbrain-save-results.test.ts`)涵盖所有 5 个的连接。每个 skill E2E 扩展在 TODOS.md 中跟踪。
- **`.gbrain-source` 写入语义**:gstack 将已记录的仅读取合约视为承重,但未独立验证 gbrain CLI 是否永远不会基于 pin 重新路由写入。如果你发现它确实这样做了,那就是要在上游提交的 gbrain 错误。
