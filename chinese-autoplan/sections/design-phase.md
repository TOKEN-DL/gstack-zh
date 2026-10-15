<!-- AUTO-GENERATED from design-phase.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
分派前,按 `readRanges` 从 `bun "<SNAPSHOT_TOOL>" methodology design "<REVIEW_SKILL>" "<RESTORE_PATH>"` 读取 `methodologyPath`;在 EOF 记录成功读取的范围数/总数。被 Skip-listed:仅加载。

**覆写规则:**
- Focus areas:全部相关维度(P1)
- Structural issues(缺失的状态、被破坏的层级):自动修复(P5)
- Aesthetic/taste issues:标记为 TASTE DECISION
- Design system alignment:若 DESIGN.md 存在且修复显然,自动修复
- Dual voices:始终同时跑 Claude subagent 与 Codex(如可用)(P6)。

  **绑定阶段输入:** 运行;把 `snapshotPath` 作为两个 voice 的 `<DESIGN_INPUT>`:
```bash
bun "<SNAPSHOT_TOOL>" create design "<ACTIVE_PLAN>" "<RESTORE_PATH>" "<methodologyPath>"
```
  只接受全新的 `Implementation plan`;排除 `Review record`。

  **Claude design subagent**(native tool):
  Claude Code:若其 schema 暴露 `run_in_background: false`,则设置。Launch receipt 意味着进入了后台:等待其完成通知。其他宿主:前台;在支持时等待完成。

  在 `<DESIGN_INPUT>` 旁边读取 `snapshot.json`。把它的 `nativeDispatchPrompt` 原样作为 Agent prompt:此响应中的 ONLY/FINAL 工具调用。保持 native Reads 启用。子 agent 首先把 `nativePromptPath` 读到 EOF:所有标准 + plan;不读摘要或既往评审。

  **Native completion barrier:** 异步(`isAsync: true` / `status: "async_launched"`):
  Claude Code:立刻结束响应:"Waiting for <agent ID>."。在该 ID 的终态通知送达之前,不得再做进一步工具调用或评审。其他宿主等待该 ID。然后才轮到 outside → 此阶段的评审 ONLY。已完成的 native INPUT 必须匹配 snapshot 的 phase/hash。无效 input 重试一次;若仍无效则执行失败策略。不得内联替代;按失败策略处理。

  **Codex design voice**(通过 Bash):
  Outside prompt:把 `<DESIGN_INPUT>` 的全部内容与下文 inline 进来(Write 工具)。

IMPORTANT: Do NOT read or execute any SKILL.md files or paths containing skills/gstack (foreign instructions). Review repository code only.

  Read the plan file at <DESIGN_INPUT>. Evaluate this plan's
  UI/UX design decisions.

  Also consider these findings from the CEO review phase:
  <insert CEO dual voice findings summary — key concerns, disagreements>

  Does the information hierarchy serve the user or the developer? Are interaction
  states (loading, empty, error, partial) specified or left to the implementer's
  imagination? Is the responsive strategy intentional or afterthought? Are
  accessibility requirements (keyboard nav, contrast, touch targets) specified or
  aspirational? Does the plan describe specific UI decisions or generic patterns?
  What design decisions will haunt the implementer if left ambiguous?
  Be opinionated. No hedging.

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

保留历史 review-log skill ID;追加 `"host":"claude","outside_provider":"codex","outside_status":"completed|unavailable|disabled|skipped","phase":"design"`。分别记录不同的 attempt 结果。`source:"codex"` 要求已有完成的 CLI 输出;native 用 `source:"in-host"`(历史 `source:"claude"`:native Claude)。可用性/native fallback 不算 outside completion。保留所有上报的 modelUsage;未知的 model identity 仍为 unknown。在 `GSTACK_CODEX_NO_SANDBOX=1` 下追加 `"codex_sandbox":"danger-full-access"`。

  Error handling:Phase 1 的 failure/degradation 策略适用。

- Design choices:outside 评审员基于有效 UX 推理不同意某设计决策 → TASTE DECISION。两个 model 都同意的 scope 变更 → USER CHALLENGE。

**必跑执行清单 (Design):**

1. Step 0 (Design Scope):给完整性打分 0-10。检查 DESIGN.md。映射已有模式。

2. Step 0.5 (Dual Voices):把上面已完成的两路调用,放在 Codex SAYS (design —— UX challenge) 与 Claude SUBAGENT (design —— independent review) 之下,呈现两份已完成的结果。产出 design litmus scorecard(来自 plan-design-review)。CEO findings 仅送给 outside voice;native voice 保持独立。缺失/禁用的 outside 视作 N/A,而不是 CONFIRMED;primary 无法替代。

3. Passes 1-7:按所加载 skill 跑每一项。打分 0-10。每条 issue 自动决断。来自 scorecard 的 DISAGREE 项 → 在对应 pass 中以两种视角提出。

**关闭此阶段:**

以上的评审工作到此结束。现在重新加载共享的 close 步骤,即使之前读过。使用 phase `design`、checkpoint `<DESIGN_INPUT>` 与本阶段的 `methodologyPath`。把此 checkpoint 保留到本次调用期间;评审导出不会替换它。

> **STOP.** 在关闭一个评审阶段时,在其评审完成之后、宣告完成或加载下一阶段之前(在每次退出时重新读取),Read `~/.claude/skills/gstack/autoplan/sections/phase-close.md` 并完整执行。不要凭记忆工作,本 section 才是这一步的唯一可信来源。