<!-- AUTO-GENERATED from test-coverage.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## Step 7: 测试覆盖率审计

### 共享子代理调度

对于 Step 7、8 和 10,使用 Agent 工具,设置 `run_in_background: false`。省略该标志会在后台运行子代理。显式标志在保持新鲜性的同时触发结果。不要调用目标作为 Skill 或改为内联运行它。仅在该章节的文档化回退下允许内联工作,在失败的子代理已停止之后。

通过 Agent 调度审计,设置 `subagent_type: "general-purpose"` 和 `run_in_background: false`,使用上面的共享前台调度规则。在应用覆盖率闸门之前等待其 LAST-line JSON。

**生成允许:** 每个调用总生成传递最多 2 次。在调度/内联执行之前,包括初始审计、失败和零测试结果,每次迭代时每次生成授权尝试计数。重新输入永远不会重置它。
已使用两次通过意味着不再允许再次生成;只读重新评估不使用传递。

**子代理提示词:** 提供 `<base>`、Step 4 的框架/引导决策、允许的路径/命令、剩余的差距、已使用的传递和生成允许,以及下面的闸门读取的 CLAUDE.md `## Test Coverage` 值(`Generation cap:`、`Base control:`、`Base control budget:`)。无允许意味着仅审计;缺少权限不是批准。保留 30 路径/每次传递 5 个测试/2 分钟每次测试的上限。

**在第一次调度之前,**清理先前中断的运行留下的基础控制工作树:

```bash
git worktree prune
find "${TMPDIR:-/tmp}" -maxdepth 1 -name 'gstack-base-control.*' -mmin +10 2>/dev/null | while IFS= read -r d; do git worktree remove --force "$d/wt" >/dev/null 2>&1; rm -rf "$d"; done
```

```text
You are running a ship-workflow test coverage audit. Run `git diff origin/<base>` to include uncommitted tracked changes; also read relevant non-ignored untracked source/tests. Do not commit or push. Perform only this audit; return unresolved user decisions to the parent instead of asking or advancing to another workflow step.

Generation: <allowed|audit-only>; passes used: <N> of 2. Audit-only overrides every generation instruction below.

Coverage goal: every changed behavior is protected by a test that would catch a real regression. Test count is not a goal. Evaluate what was ACTUALLY coded (from the diff), not what was planned.

### Test Framework Detection

Before analyzing coverage, detect the project's test framework:

1. **Read CLAUDE.md** — look for a `## Testing` section with test command and framework name. If found, use that as the authoritative source.
2. **If CLAUDE.md has no testing section, auto-detect:**

```bash
setopt +o nomatch 2>/dev/null || true  # zsh 兼容
# 检测项目运行时(标记是证据,不是要盲目运行的命令)
[ -f manage.py ] && echo "RUNTIME:python FRAMEWORK:django"
{ [ -f pyproject.toml ] || [ -f pytest.ini ] || [ -f tox.ini ] || [ -f setup.cfg ] || [ -f requirements.txt ]; } && echo "RUNTIME:python"
{ [ -f Gemfile ] || [ -f Rakefile ] || [ -f .rspec ]; } && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
[ -f pom.xml ] && echo "RUNTIME:jvm BUILD:maven"
{ [ -f build.gradle ] || [ -f build.gradle.kts ]; } && echo "RUNTIME:jvm BUILD:gradle"
# 检查现有测试基础设施 — 配置文件、脚本和测试文件
ls jest.config.* vitest.config.* playwright.config.* cypress.config.* .rspec pytest.ini tox.ini phpunit.xml 2>/dev/null
[ -f package.json ] && grep -q '"test"[[:space:]]*:' package.json && echo "SCRIPT:package.json test"
[ -f Makefile ] && grep -qE '^(test|check):' Makefile && echo "TARGET:make test"
git ls-files | grep -cE '(^|/)(tests?|spec|__tests__)/|(^|/)tests?\.py$|(^|/)test_[^/]+\.py$|_test\.(go|py|rb|ts|js|exs)$|\.(test|spec)\.[jt]sx?$|_spec\.rb$|Test\.(java|kt)$' | sed 's/^/TESTFILES:/'
```

3. **If no framework detected:** use the bootstrap decision already made in Step 4; report diagram-only coverage if setup was declined. Do not restart bootstrap from this audit.

**0. Before/after test count:**

```bash
# 在任何生成之前计算测试文件数
git ls-files 2>/dev/null | grep -E '(\.test\.|\.spec\.|_test\.|_spec\.)' | wc -l
```

将此数字存储用于 PR 主体。

**1. Trace every codepath changed** using `git diff origin/<base>`:

Read every changed file. For each one, trace how data flows through the code; don't just list functions, actually follow the execution:

1. **Read the diff.** 对于每个更改的文件,阅读完整文件(不仅仅是差异块)以了解上下文。
Definition: a **targeted audit** reviews named concrete source/test files or a
branch diff. A **prototype** is existing runnable code referenced by the plan,
not a proposed future component.

When grounded in concrete source and test files, read them in a dedicated tool
call before drawing the diagram. Finish this source read before tracing data
flow in audit item 2 below; map user flows afterward. Do not mix diff, grep,
package/config, git, or commentary into that read; use separate calls for
context. Base the diagram on that read.
2. **Trace data flow.** 从每个入口点开始(路由处理程序、导出函数、事件侦听器、组件渲染),通过每个分支跟踪数据:
   - 输入来自哪里?(请求参数、props、数据库、API 调用)
   - 什么转换它?(验证、映射、计算)
   - 它去哪里?(数据库写入、API 响应、渲染输出、副作用)
   - 每一步可能出现什么问题?(null/undefined,无效输入,网络故障,空集合)
3. **Diagram the execution.** 对于每个更改的文件,绘制一个 ASCII 图,显示:
   - 每个添加或修改的函数/方法
   - 每个条件分支(if/else、switch、三元、保护子句、早期返回)
   - 每个错误路径(try/catch、rescue、错误边界、回退)
   - 每次调用另一个函数(跟踪到它 — 它有未测试的分支吗?)
   - 每个边缘:null 输入会发生什么?空数组?无效类型?

This is the critical step; you're building a map of every line of code that can execute differently based on input. Every branch in this diagram needs coverage that would catch a real regression; the test value bar below decides whether that is a new test, an extension of an existing one, or already covered.

**2. Map user flows, interactions, and error states:**

Code coverage isn't enough; you need to cover how real users interact with the changed code. For each changed feature, think through:

- **User flows:** What sequence of actions does a user take that touches this code? Map the full journey (e.g., "user clicks 'Pay' → form validates → API call → success/failure screen"). Each step in the journey needs coverage.
- **Interaction edge cases:** What happens when the user does something unexpected?
  - Double-click/rapid resubmit
  - Navigate away mid-operation (back button, close tab, click another link)
  - Submit with stale data (page sat open for 30 minutes, session expired)
  - Slow connection (API takes 10 seconds; what does the user see?)
  - Concurrent actions (two tabs, same form)
- **Error states the user can see:** For every error the code handles, what does the user actually experience?
  - Is there a clear error message or a silent failure?
  - Can the user recover (retry, go back, fix input) or are they stuck?
  - What happens with no network? With a 500 from the API? With invalid data from the server?
- **Empty/zero/boundary states:** What does the UI show with zero results? With 10,000 results? With a single character input? With maximum-length input?

Add these to your diagram alongside the code paths. A user flow with no test is just as much a gap as an untested if/else.

**3. Check each branch against existing tests:**

Go through your diagram branch by branch — both code paths AND user flows. For each one, search for a test that exercises it:
- Function `processPayment()` → look for `billing.test.ts`, `billing.spec.ts`, `test/billing_test.rb`
- An if/else → look for tests covering BOTH the true AND false path
- An error handler → look for a test that triggers that specific error condition
- A call to `helperFn()` that has its own branches → those branches need tests too
- A user flow → look for an integration or E2E test that walks through the journey
- An interaction edge case → look for a test that simulates the unexpected action

Quality scoring rubric:
- ★★★  Tests behavior with edge cases AND error paths
- ★★   Tests correct behavior, happy path only
- ★    Smoke test / existence check / trivial assertion (e.g., "it renders", "it doesn't throw"); weak, never counts as coverage

**Test value bar.** Propose or write a test only with all four answers; otherwise extend an existing test or drop it:

1. What observable behavior, invariant or independent contract does it protect?
2. What credible regression makes it fail?
3. Why does existing coverage not already catch that? Prefer adding a row to an existing table-driven test or shared fixture over a near-duplicate.
4. Does it need a production seam (export, flag, wrapper, injection hook) that no production caller needs? If yes, test at the real boundary instead.

A test that breaks under a behavior-preserving refactor asserts implementation: rewrite it at the owning boundary, unless exact output is the declared contract (goldens, prompt bytes, wire formats).

Value card: `Value: protects=<...>; fails_when=<...>; why_new=<...>; seam=none; no_claim=<...>` (seam: `none` or its name; `no_claim`: one sentence on what green does not prove and which fixtures or mocks stand in for what, e.g. `no_claim=provider acceptance (mocked provider; unit only)` or `no_claim=none beyond protects`; it never waives a required live check); each field at most 160 UTF-8 bytes here (clamp to 157 plus `...`; written JSON keeps full values). Write it as a header comment in each generated test, next to the attribution (wrap, do not truncate); with no known comment syntax, put it in the PR body's Test value details. A missing upstream card never blocks: derive it; ignore unknown fields.

Example: Value: protects=refundPayment rejects an empty reason; fails_when=the reason guard is removed or inverted; why_new=billing.test.ts covers processPayment only; seam=none; no_claim=provider acceptance (mocked provider; unit only)
Rejected (covered_elsewhere): "checkout renders"; checkout.e2e.ts:15 covers it, so extend that test.

Weak tests (★ smoke/existence/trivial, gate-failing or unrated) never count as coverage. X = paths with a ★★/★★★ test / total paths (value-weighted; the gate uses X); Y = paths with any test / total paths. Total paths = the diff's codepath trace, max 30; zero skips the gate. A path with only weak tests is uncovered in X, covered in Y, and goes to `weak_gaps` (reason `star_one|gate_failed|unrated`), not `gaps`. Rate stars only for tests reachable from changed paths.

Retention bar: keep a test that independently enforces a public API, protocol, config, migration, storage, security, platform, default, prompt-byte, generated-output (golden), package, release or architecture contract; static or slow is no reason to delete.

Regression proof: a regression test must fail at HEAD before any repair, in its own assertion (a pass at HEAD drops the regression label; an import, fixture or env failure is a test defect: correct once or drop). It must pass at base as the control (an assertion failure there marks it invalid; any other failure is "base control unavailable: collection error") and pass after the repair. Record: `Regression proof — fails at HEAD: yes · passes at base: yes | unavailable (<reason>) | manual · passes after fix: yes | pending`.

### E2E Test Decision Matrix

When checking each branch, also determine whether a unit test or E2E/integration test is the right tool:

**RECOMMEND E2E (mark as [→E2E] in the diagram):**
- Common user flow spanning 3+ components/services (e.g., signup → verify email → first login)
- Integration point where mocking hides real failures (e.g., API → queue → worker → DB)
- Auth/payment/data-destruction flows — too important to trust unit tests alone

**RECOMMEND EVAL (mark as [→EVAL] in the diagram):**
- Critical LLM call that needs a quality eval (e.g., prompt change → test output still meets quality bar)
- Changes to prompt templates, system instructions, or tool definitions

**STICK WITH UNIT TESTS:**
- Pure function with clear inputs/outputs
- Internal helper with no side effects
- Edge case of a single function (null input, empty array)
- Obscure/rare flow that isn't customer-facing

### REGRESSION RULE (mandatory)

**IRON RULE:** When the coverage audit identifies a REGRESSION — code that previously worked but the diff broke — a regression test is written immediately. No AskUserQuestion. No skipping. Regressions are the highest-priority test because they prove something broke.

A regression is when:
- The diff modifies existing behavior (not new code)
- The existing test suite (if any) doesn't cover the changed path
- The change introduces a new failure mode for existing callers

When uncertain whether a change is a regression, err on the side of writing the test.

**Red-first proof.** Apply the value bar's Regression proof to every regression test: the diff at HEAD is the pre-fix code, so run the new test at HEAD before any repair. Then, unless the parent says `Base control: off`, run this base control once per regression test in diff order, within a 3-minute total per /ship run (`Base control budget:` seconds per test, default 90). Past the total, record "base control unavailable: budget" and report "N of M regression tests got base control". The block is one shell invocation; it installs nothing and runs no build or postinstall.

```bash
# Set: BASE = the base branch this /ship run resolved; TEST = the test file; FIXTURES = new
# test-only fixtures it imports (repo-relative, may be empty); RUN = the detected
# runner for one file (e.g. "bun test $TEST"); BUDGET = seconds for this run (default 90).
ROOT=$(git rev-parse --show-toplevel)
CTL_TMP=$(mktemp -d "${TMPDIR:-/tmp}/gstack-base-control.XXXXXX")
cleanup() { git -C "$ROOT" worktree remove --force "$CTL_TMP/wt" >/dev/null 2>&1; rm -rf "$CTL_TMP"; [ -e "$CTL_TMP" ] && echo "BASE_CONTROL_LEFTOVER: $CTL_TMP (run: git worktree prune)"; }
trap cleanup EXIT INT TERM
(
  [ -f "$ROOT/package.json" ] || { echo "BASE_CONTROL: unavailable (ecosystem)"; exit 0; }
  git -C "$ROOT" remote get-url origin >/dev/null 2>&1 || { echo "BASE_CONTROL: unavailable (no base remote)"; exit 0; }
  timeout 30 git -C "$ROOT" fetch --quiet origin "$BASE" || { echo "BASE_CONTROL: unavailable (base not fetched)"; exit 0; }
  git -C "$ROOT" worktree add --quiet --detach "$CTL_TMP/wt" "origin/$BASE" >/dev/null 2>&1 || { echo "BASE_CONTROL: unavailable (worktree add failed)"; exit 0; }
  for f in $(echo "$TEST $FIXTURES"); do mkdir -p "$CTL_TMP/wt/$(dirname "$f")" && cp "$ROOT/$f" "$CTL_TMP/wt/$f"; done
  [ -d "$ROOT/node_modules" ] && ln -s "$ROOT/node_modules" "$CTL_TMP/wt/node_modules"
  cd "$CTL_TMP/wt" && timeout "${BUDGET:-90}" sh -c "$RUN" > "$CTL_TMP/out" 2>&1; rc=$?
  tail -n 40 "$CTL_TMP/out"
  if [ "$rc" -eq 0 ]; then echo "BASE_CONTROL: passes at base"
  elif [ "$rc" -eq 124 ]; then echo "BASE_CONTROL: unavailable (budget)"
  else echo "BASE_CONTROL: fails at base (exit $rc)"; fi
)
```

Classify "fails at base" from the output: a failure in the test's own assertion marks it invalid (correct once or drop it); an import, collection, missing generated artifact or dependency failure is "base control unavailable: collection error" and the test stays. Print each unavailable result as: base control unavailable: <reason>. The fails-at-HEAD result still stands. To check by hand: `git worktree add --detach <tmp> <base>`; copy the test and its new fixtures to the same paths; run the detected test command in <tmp>; `git worktree remove --force <tmp>`. Then report `passes at base: manual`. (see ~/.claude/skills/gstack/docs/test-value-bar.md#base-control-unavailable)

Return `"regression_proof":{"red_at_head":N,"base_green":N,"base_unavailable":N}` counts in the JSON and each test's record line in the diagram.

**4. Output ASCII coverage diagram:**

For targeted audits, start Test review output with the coverage diagram. In full
plan reviews, put it inside the normal Test review section. Required outputs
keep the final terminal report order.

Include BOTH code paths and user flows in the same diagram. Mark E2E-worthy and eval-worthy paths:

```
CODE PATHS                                            USER FLOWS
[+] src/services/billing.ts                           [+] Payment checkout
  ├── processPayment()                                  ├── [★★★ TESTED] Complete purchase — checkout.e2e.ts:15
  │   ├── [★★★ TESTED] happy + declined + timeout      ├── [GAP] [→E2E] Double-click submit
  │   ├── [GAP]         Network timeout                 └── [GAP]        Navigate away mid-payment
  │   └── [GAP]         Invalid currency
  └── refundPayment()                                 [+] Error states
      ├── [★★  TESTED] Full refund — :89                ├── [★★  TESTED] Card declined message
      └── [★   TESTED] Partial (non-throw only) — :101  └── [GAP]        Network timeout UX

LLM integration: [GAP] [→EVAL] Prompt template change — needs eval test

COVERAGE: 5/13 paths tested (38%)  |  Code paths: 3/5 (60%)  |  User flows: 2/8 (25%)
QUALITY: ★★★:2 ★★:2 ★:1  |  GAPS: 8 (2 E2E, 1 eval)
```

Legend: ★★★ behavior + edge + error  |  ★★ happy path  |  ★ smoke check
[→E2E] = needs integration test  |  [→EVAL] = needs LLM eval

Avoid bare `[ ]` or `[x]` in diagrams unless the block includes
`Legend: [x] tested | [ ] no test`. Prefer `[GAP]`, `[★★ TESTED]`,
`[→E2E]`, `[→EVAL]`; keep user-flow markers off code-path rows.

**Fast path:** All paths covered → "Step 7: All new code paths have test coverage ✓" Continue.

**5. Generate tests for uncovered paths:**

If test framework detected (or bootstrapped in Step 4):
- Apply the test value bar before writing each test. Extend an existing test (a new table row, fixture case or assertion) before creating a file. Record every proposal you decline in `tests_rejected` with a `reason_code` from `duplicate_protects, needs_seam, incomplete_card, no_credible_regression, covered_elsewhere, implementation_coupled`.
- Never add a production seam for a test; a seam that is not `none` names its non-test callers: `seam=<name> (non-test callers: N, via <search command>)`.
- Write the value card as a header comment in each generated or extended test.
- Prioritize error handlers and edge cases first (happy paths are more likely already tested)
- Read 2-3 existing test files to match conventions exactly
- Generate the smallest native test for each path: unit tests for logic, mocking only services unrelated to the behavior under test (integration points where mocking hides real failures go to [→E2E]).
- For paths marked [→E2E]: generate integration/E2E tests using the project's E2E framework (Playwright, Cypress, Capybara, etc.)
- For paths marked [→EVAL]: generate eval tests using the project's eval framework, or flag for manual eval if none exists
- Write tests that exercise the specific uncovered path with real assertions
- Run each test. Passes → keep the change and report its path; the parent commits in Step 15.
- Fails → diagnose whether the test/fixture is invalid or a declared product contract is broken. Correct a demonstrated test defect once; preserve a valid red regression and route the reproduced product failure through the parent's fix/approval flow. Never delete or weaken it to manufacture green; retain unresolved coverage in the diagram.

Caps: 30 code paths max; 5 tests per generation pass (code + user flow combined; the parent's `Generation cap:` overrides 5); an extension uses one slot and a rejection uses none; 2-min per-test exploration cap. List each remaining gap below the diagram (inside `diagram`) as a proposed test with its value card.

Do not rate stars for tests you wrote in this pass: count them as unrated (weak, reason `unrated`). The parent's read-only rating dispatch rates them. Counts are disjoint, precedence extended > added > rejected: one gap lands in at most one of `tests_extended`, `tests_added`, `tests_rejected`.

If no test framework AND user declined bootstrap → diagram only, no generation. Note: "Test generation skipped — no test framework configured."

**Diff is test-only changes:** Return a skipped audit with null coverage, zero gaps, and "No new application code paths to audit."

**6. After-count and coverage summary:**

```bash
# 在生成后计算测试文件数
git ls-files 2>/dev/null | grep -E '(\.test\.|\.spec\.|_test\.|_spec\.)' | wc -l
```

For PR body: `Tests: {before} → {after} (+{delta} new)`
Coverage line: `Test Coverage Audit: N new code paths. M covered (Y% any test, X% value-weighted). K tests generated, awaiting parent commit.`

### Test Plan Artifact

After producing the coverage diagram, write a test plan artifact so `/qa` and `/qa-only` can consume it:

```bash
GSTACK_STATE_ROOT=$(~/.claude/skills/gstack/bin/gstack-paths --get GSTACK_STATE_ROOT); : "${GSTACK_STATE_ROOT:?gstack-paths failed; reinstall with ./setup or /gstack-upgrade}"