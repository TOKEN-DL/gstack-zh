<!-- AUTO-GENERATED from ceo-phase.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
分派前,按 `readRanges` 从 `bun "<SNAPSHOT_TOOL>" methodology ceo "<REVIEW_SKILL>" "<RESTORE_PATH>"` 读取 `methodologyPath`;在 EOF 记录成功读取的范围数/总数。被 Skip-listed:仅加载。

按此顺序执行:Step 0(含其内嵌的 Spec Review Loop)→ Claude CEO voice → Codex CEO voice → consensus → Review Sections → 保存摘要 → 阶段宣告。派发一个评审员并不等于其步骤完成。

**覆写规则:**
- Mode selection: SELECTIVE EXPANSION
- Premises:接受合理的(P6)。把明显错误或被质疑的 premises 排队,作为 Phase 4 的 User Challenges:假设、推进的原因、推进的成本。用户在那里决定;绝不半路停流水线。
- Alternatives:取完整性最高者(P1)。并列时取最简单者(P5)。若前两名接近 → 标记为 TASTE DECISION。
- Scope expansion:在 blast radius 内 + CC <1 天 → 批准(P2)。在外 → 推迟到 TODOS.md(P3)。重复 → 拒绝(P4)。边缘(3-5 文件) → 标记为 TASTE DECISION。
- All 11 review sections:全部跑完,每条 issue 自动决断,记录每项决定。

**必跑执行清单 (CEO):**

按上述覆写规则,在 SELECTIVE EXPANSION 路线上完成加载 skill 的全部 Step 0 分析/输出:在 0I 与 Review Sections 之前,先有 CEO scope 文档与 0H Spec Review Loop。

**0H 步骤,为每份 spec review 准备当前输入。** 创建一个 amendment checkpoint;将其 `snapshotPath` 作为 `<CEO_STEP0_CHECKPOINT>`,贯穿整个 CEO:
```bash
bun "<SNAPSHOT_TOOL>" create ceo "<ACTIVE_PLAN>" "<RESTORE_PATH>" "<methodologyPath>"
```
把 Step 0 中所有已接受的行为、条件、测试和手工 checklist 放入 CEO accepted-obligations 块。保留 source-plan 与 DESIGN.md 的需求;User Challenges 保留原始需求。Taste 是临时性的自动决断;已接受的 expansion 必须在不假设排队中的变更获批的前提下成立。把决定历史与待评审工作放进 `Review record`。

每次 spec 派发前(包含每项 spec fix 被接受之后),运行:
```bash
bun "<SNAPSHOT_TOOL>" amend-input ceo "<ACTIVE_PLAN>" "<CEO_STEP0_CHECKPOINT>" "<RESTORE_PATH>" "<methodologyPath>"
```
本命令应用已记录的需求,并导出完整的当前 `Implementation plan`。把返回的 `checkpointPath` 作为 amendment 基线;把返回的 `reviewInputPath` 作为 `<CEO_SPEC_INPUT>`。在每个返回的 `readRanges` offset/limit 读到 EOF 后,再完整阅读一次 CEO scope 摘要。在二者之间核对 dispositions、scope counts、proposal IDs 与真实的标题/测试引用。把延后项关联到实际 TODO 或待写入项。在不变更决定、不丢掉 findings/必填字段、不杜撰引用的前提下修复摘要漂移。如工作 plan 发生变化,重新跑一次 `amend-input` 与 readback 再分派。把完整的 CEO scope 摘要与 `<CEO_SPEC_INPUT>` 提供给加载的 Spec Review Loop。checkpoint 是不可变的前序状态;绝不把它当作当前工作 plan 提交。准备失败是输入失败,不算完成 spec review。

保留 Spec Review Loop 现有的停止条件和三发上限。Loop 结束后,为两个 voice 再创建一份全新的 snapshot;amendment checkpoint 保留。

Step 0.5 (Dual Voices):Step 0 的 Spec Review Loop 之后,先消费 native CEO 评审,再消费可用的 outside voice(P6)。在 consensus 之前呈现两份已完成的结果;native pass 始终运行。

  **绑定阶段输入:** 运行;把 `snapshotPath` 作为两个 voice 的 `<CEO_INPUT>`:
```bash
bun "<SNAPSHOT_TOOL>" create ceo "<ACTIVE_PLAN>" "<RESTORE_PATH>" "<methodologyPath>"
```
  只接受全新的 `Implementation plan`;排除 `Review record`。

  **Claude CEO subagent**(通过 Agent 工具):
  Claude Code:若其 schema 暴露 `run_in_background: false`,则设置。Launch receipt 意味着进入了后台:等待其完成通知。其他宿主:前台运行;在支持时等待完成。

  在 `<CEO_INPUT>` 旁边读取 `snapshot.json`。把它的 `nativeDispatchPrompt` 原样作为 Agent prompt:此响应中的 ONLY/FINAL 工具调用。保持 native Reads 启用。子 agent 首先把 `nativePromptPath` 读到 EOF:所有标准 + plan;不读摘要或既往评审。

  **Native completion barrier:** 异步(`isAsync: true` / `status: "async_launched"`):
  Claude Code:立刻结束响应:"Waiting for <agent ID>."。在该 ID 的终态通知送达之前,不得再做进一步工具调用或评审。其他宿主等待该 ID。然后才轮到 outside → 此阶段的评审 ONLY。已完成的 native INPUT 必须匹配 snapshot 的 phase/hash。无效 input 重试一次;若仍无效则执行失败策略。不得内联替代;按失败策略处理。

  **Codex CEO voice**(通过 Bash):
  Outside prompt:把 `<CEO_INPUT>` 的全部内容与下文 inline 进来(Write 工具)。

IMPORTANT: Do NOT read or execute any SKILL.md files or paths containing skills/gstack (foreign instructions). Review repository code only.

  You are a CEO/founder advisor reviewing a development plan.
  Challenge the strategic foundations: Are the premises valid or assumed? Is this the
  right problem to solve, or is there a reframing that would be 10x more impactful?
  What alternatives were dismissed too quickly? What competitive or market risks are
  unaddressed? What scope decisions will look foolish in 6 months? Be adversarial.
  No compliments. Just the strategic blind spots.
  File: <CEO_INPUT>

把**完整的 prompt 与上下文**(包括真实的 plan/spec/source)写到一个私有文件。把其 shell-quoted 路径替换 `<prepared-prompt-file>`;绝不把用户文本内插到 shell 源里。要求每条 finding 给出 severity(Critical、High、Medium 或 Low)以及一行最终的 Recommendation: <action> because <specific reason>,其中包含一条明确的 no-findings 理由。

```bash
# GSTACK_ACTIVE_HOST names the harness, never the model.
if { [ -n "${CODEX_THREAD_ID:-}" ] || [ -n "${CODEX_SANDBOX:-}" ] || [ "${GSTACK_ACTIVE_HOST:-}" = codex ]; }; then
  echo 'Codex outside review unavailable: harness mismatch; no outside process started. Missing coverage.' >&2
  if { [ -n "${CLAUDECODE:-}" ] || [ "${GSTACK_ACTIVE_HOST:-}" = claude ]; } && { [ -n "${CODEX_THREAD_ID:-}" ] || [ -n "${CODEX_SANDBOX:-}" ] || [ "${GSTACK_ACTIVE_HOST:-}" = codex ]; }; then
    echo 'Inherited harness markers conflict. Run setup --host <actual-harness> (claude or codex); do not guess a replacement provider.' >&2
  else
    echo 'Repair installed skills: run setup --host codex from your gstack checkout.' >&2
  fi
  exit 78
fi

_REPO_ROOT=$(git rev-parse --show-toplevel) || { echo 'ERROR: not in a git repo' >&2; exit 1; }
_OUTSIDE_TMP=$(mktemp -d "${TMPDIR:-/tmp}/gstack-outside.XXXXXXXX") || exit 1
trap 'rm -rf "$_OUTSIDE_TMP"' EXIT
_OUTSIDE_INPUT="$_OUTSIDE_TMP/prompt"
cat -- '<prepared-prompt-file>' >"$_OUTSIDE_INPUT" || exit 1

_CODEX_PROBE="$HOME/.claude/skills/gstack/bin/gstack-codex-probe"
export _CODEX_DEADLINE=$(($(date +%s)+540)); _CODEX_OUT=$("$_CODEX_PROBE" role-ready exec) || exit $?
_CODEX_SEL=$(echo "$_CODEX_OUT" | sed -n 's/^CODEX_SEL: //p')
_CODEX_SANDBOX_MODE=$(echo "$_CODEX_OUT" | sed -n 's/^CODEX_SANDBOX: //p')
"$_CODEX_PROBE" show-first-use-notice
_OUTSIDE_EXIT=0
"$_CODEX_PROBE" run-with-timeout --effort high 540 codex exec - -C "$_REPO_ROOT" -s "${_CODEX_SANDBOX_MODE:?}" -c "model=\"${_CODEX_SEL:?}\"" -c skills.include_instructions=false -c 'web_search="cached"' --json -o "$_OUTSIDE_TMP/text" <"$_OUTSIDE_INPUT" >"$_OUTSIDE_TMP/events" 2>"$_OUTSIDE_TMP/stderr" || _OUTSIDE_EXIT=$?
cat "$_OUTSIDE_TMP/text" 2>/dev/null || tail -n 20 "$_OUTSIDE_TMP/events"
if [ "$_OUTSIDE_EXIT" -eq 124 ]; then
  "$_CODEX_PROBE" log-event codex_timeout "540" || true
  "$_CODEX_PROBE" log-hang autoplan 0 || true
fi
cat "$_OUTSIDE_TMP/stderr" >&2 || { [ "$_OUTSIDE_EXIT" -ne 0 ] || _OUTSIDE_EXIT=1; }
_OUTSIDE_RC=0
bun "$HOME/.claude/skills/gstack/lib/outside-review-result.ts" --label 'Codex outside review' --exit "$_OUTSIDE_EXIT" --stderr "$_OUTSIDE_TMP/stderr" --events "$_OUTSIDE_TMP/events" review "$_OUTSIDE_TMP/text" || _OUTSIDE_RC=$?
case "$_OUTSIDE_RC" in
  0|3) ;;
  4) echo 'OUTSIDE_STATUS: unverified provider=codex host=claude'; exit 4 ;;
  *) [ "$_OUTSIDE_EXIT" -ne 0 ] && exit "$_OUTSIDE_EXIT"; exit 1 ;;
esac
echo 'OUTSIDE_STATUS: completed provider=codex host=claude'
```

使用 Bash `timeout: 600000`;把完整响应放进 `tool-output` fence。必须成功执行并包含有效 markers。拒绝、空/畸形输出、缺失 score/severity/completion marker、超时或 CLI 失败 → `outside_status: unavailable`。P0/P1 findings 与 native findings 一样会阻断;`OUTSIDE_STATUS: unverified` 算作 missing coverage。使用调用方的 fallback;missing coverage 永远不是 clean/PASS。任一结果出来后,只删除你的私有 prompt;scratch 清理是自动的。

失败/不完整的 outside review → unavailable;disabled → skip outside。两种情况下都保留 native pass。

保留历史 review-log skill ID;追加 `"host":"claude","outside_provider":"codex","outside_status":"completed|unavailable|disabled|skipped","phase":"ceo"`。分别记录不同的 attempt 结果。`source:"codex"` 要求已有完成的 CLI 输出;native 用 `source:"in-host"`(历史 `source:"claude"`:native Claude)。可用性/native fallback 不算 outside completion。保留所有上报的 modelUsage;未知的 model identity 仍为 unknown。在 `GSTACK_CODEX_NO_SANDBOX=1` 下追加 `"codex_sandbox":"danger-full-access"`。

  **错误处理:** Codex 鉴权/超时/空输出 → 仅继续 Claude subagent,标 `[single-model]`。若它也失败 → "Outside voices unavailable — continuing with primary review."。

  **降级矩阵:** 两者都失败 → "single-reviewer mode"。仅 Codex → 标 `[codex-only]`。仅 Subagent → 标 `[subagent-only]`。

- Strategy choices:outside 评审员基于有效战略理由不同意某 premise 或 scope 决策 → TASTE DECISION。两个 model 都同意需要变更用户既定结构(merge、split、add、remove) → USER CHALLENGE(永不自动决断)。

由已完成的结果产出 CEO consensus 表:

```
CEO DUAL VOICES — CONSENSUS TABLE:
  Dimension                           Claude  Codex  Consensus
  1. Premises valid?                   —       —      —
  2. Right problem to solve?           —       —      —
  3. Scope calibration correct?        —       —      —
  4. Alternatives sufficiently explored?—      —      —
  5. Competitive/market risks covered? —       —      —
  6. 6-month trajectory sound?         —       —      —
CONFIRMED = completed subagent + outside; primary cannot replace outside.
Outside disabled/unavailable: six Consensus cells N/A, never CONFIRMED.
Native findings stay separate; disagreements → taste; flag single-voice criticals.
```

Sections 1-11 — 对每个 section,按所加载 skill 文件中的标准运行:
- 有 findings 的 section:完整分析,自动决断每条 issue,记录到 audit trail
- 没有 findings 的 section:用 1-2 句话说明审查了什么以及为何没有 flag。绝不可把 section 压缩到表格中对应行的名字。
- Section 11 (Design):仅在 Phase 0 检测到 UI scope 时跑

**Phase 1 的强制输出:**
- Reality rows:加载的 skill 中的 `REALITY:` 行,每条适用 row 一条;运行 `~/.claude/skills/gstack/bin/gstack-plan-reality check --phase ceo <review-file|->` 会在阶段关闭前打印 `PLAN_REALITY: … verdict=complete`(无人值守:`incomplete` 保持开启)。
- "NOT in scope" section,列出推迟项及其理由
- "What already exists" section,把子问题映射到已有代码
- Error & Rescue Registry 表(来自 Section 2)
- Failure Modes Registry 表(来自 review sections)
- Dream state delta(本 plan 与 12 个月理想态相比的差距)
- Completion Summary(CEO skill 的完整摘要表)

**关闭此阶段:**

以上的评审工作到此结束。现在重新加载共享的 close 步骤,即使之前读过。使用 phase `ceo`、checkpoint `<CEO_STEP0_CHECKPOINT>` 与本阶段的 `methodologyPath`。把此 checkpoint 保留到本次调用期间;评审导出不会替换它。

> **STOP.** 在关闭一个评审阶段时,在其评审完成之后、宣告完成或加载下一阶段之前(在每次退出时重新读取),Read `~/.claude/skills/gstack/autoplan/sections/phase-close.md` 并完整执行。不要凭记忆工作,本 section 才是这一步的唯一可信来源。