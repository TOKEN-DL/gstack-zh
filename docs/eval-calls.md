# 评估调用计划:为付费 gate 定价而不运行它

付费 gate(基准、评估烟雾,任何为钱调用模型 provider 的命令)在任何人批准它之前已定价。gstack 不运行 gate 或任何其他仓库命令来了解其成本。仓库提交**调用计划文件**:纯数据,命名确切的 gate 命令、它所基于的提交和输入,以及每个调用用途的一行及其 model 和 token 计数。`gstack-eval-plan price-gate` 读取该文件,检查它仍然描述正在批准的内容,从 `lib/pricing.ts` 对其进行定价,并写入收据。计划:字段 wave CD2(`docs/designs/GSTACK_FIELD_WAVE_2026_10_12.md`,gate 决定 d5b 已修正)。

```bash
gstack-eval-plan price-gate --gate-file <file> --cap-usd 20 [--smoke-cap-usd 2] [--plan-file <file>]
```

`--gate-file` 从文件读取确切的 gate 命令(丢弃一个尾随换行符),因此带有引号或 `$` 的命令永远不会通过 shell 字符串;`--gate "<command>"` 内联获取它。给两者之一。

## 文件来自哪里

1. `--plan-file <file>`:操作员给定的文件,从磁盘读取。
2. 否则,`price_file` 在 `.gstack/pregate.json` 中,如 `origin/<base>` 上提交的那样(`--base`,默认 origin/HEAD 的分支):仓库相对路径,其字节从 **HEAD 提交**(`git show HEAD:<path>`)读取,而不是从工作树中读取。

文件中的任何内容都不会被执行:不是 gate 命令,不是先决条件,不是输入路径。该文件被解析为文本。

## 文件格式,有线版本 v1

```
# gstack eval call plan, wire format v1 (docs/eval-calls.md); priced by `gstack-eval-plan price-gate`, never executed
GSTACK_EVAL_GATE v1: bun run eval:synopsis --questions 50 --arms opus,sol
GSTACK_EVAL_HEAD v1: 0123456789abcdef0123456789abcdef01234567
GSTACK_EVAL_INPUT v1: sha256=<64 hex> path=evals/synopsis/questions.jsonl
GSTACK_EVAL_PRECONDITION: ANTHROPIC_API_KEY and OPENAI_API_KEY set at run time
GSTACK_EVAL_CALLS v1: purpose=answer model=anthropic/claude-opus-5-5 calls=100 in_tokens=400000 out_tokens=200000 max_out_tokens=8192 scope=full
GSTACK_EVAL_CALLS v1: purpose=judge model=gpt-6.1-sol calls=100 in_tokens=300000 out_tokens=50000 max_out_tokens=2048 scope=full
GSTACK_EVAL_CALLS v1 end: lines=2
```

| 行 | 计数 | 含义 |
|---|---|---|
| `GSTACK_EVAL_GATE v1: <command>` | 恰好 1 | 此计划定价的确切 gate 命令;通过字符串相等性与 `--gate` 比较 |
| `GSTACK_EVAL_HEAD v1: <sha>` | 恰好 1 | 计算此计划的完整提交 sha |
| `GSTACK_EVAL_INPUT v1: sha256=<hex> path=<path>` | 0 或更多 | 计数来自的每个输入(行、数据集、提示),仓库相对,没有 `..` |
| `GSTACK_EVAL_PRECONDITION: <text>` | 0 或更多 | 运行时必须保持的内容;每个都回显为 `GATE_PRECONDITION:` |
| `GSTACK_EVAL_CALLS v1: …` | 0 或更多 | 每个调用用途一行(唯一的 `purpose` + `model`) |
| `GSTACK_EVAL_CALLS v1 end: lines=<n>` | 恰好 1,最后 | `n` = 调用行数;缺失或错误计数的结束行意味着部分生产者并拒绝 |

空行和以 `#` 开头的行被忽略。每个其他行必须是上述之一;未知的种类、未知的版本(`v2`)、缺失或额外的键、重复的键、非整数计数、`calls=0` 或结束行之后的任何内容都以 `EVAL_DRY_RUN_FAILED` 拒绝,命名行。

### 调用行键

| 键 | 含义 |
|---|---|
| `purpose` | 调用用途的名称(`[A-Za-z0-9._:-]+`),例如 `answer`、`judge`、`embed` |
| `model` | model id;接受 provider 前缀(`anthropic/`、`openai:`、`voyage:`) |
| `calls` | 此用途的调用数(至少 1) |
| `in_tokens` | **聚合** 所有此用途调用的输入 token |
| `out_tokens` | **聚合** 所有此用途调用的预期输出 token |
| `max_out_tokens` | **每调用** 输出预留(调用发送的 `max_tokens`) |
| `scope` | `full`(计数的运行)或 `smoke`(小型预运行检查) |

聚合总数是该用途预期的成本:`usd = price(in_tokens, out_tokens)`。每调用预留是单次调用可能的最大成本,即 provider 或支出账本预留的内容:`reservation_usd = price(ceil(in_tokens / calls), max_out_tokens)`。

## 绑定检查(`EVAL_PLAN_STALE`)

该计划拒绝,没有收据,当它不再描述正在批准的内容时:

- `GSTACK_EVAL_GATE` 命令与 `--gate` 相差一个字符;
- `GSTACK_EVAL_HEAD` 提交不是 HEAD,它们之间的任何其他文件都已更改(已提交的 plan file 位于它计算所在的 head 的一个提交之上,因此允许其自己的提交);
- 输入在 HEAD 提交中的字节的 sha256 与其 `GSTACK_EVAL_INPUT` 行不同,或输入不在 HEAD 中;
- 基本策略声明了一个不在 HEAD 中的 `price_file`。

`--allow-unpriced` 永远不会覆盖这些:只有根本没有声明 gate 的 gate,或具有零调用行的计划,是未定价的。

## 定价结果

```
GATE_PLAN: file=evals/gate.calls source=price_file sha256=<hex> head=<sha12> inputs=1 verified
GATE_PRECONDITION: ANTHROPIC_API_KEY and OPENAI_API_KEY set at run time
GATE_CALLS: purpose=answer model=anthropic/claude-opus-5-5 calls=100 usd=5.6 reservation_usd=0.17984 scope=full
GATE_CALLS: purpose=judge model=gpt-6.1-sol calls=100 usd=1.1 reservation_usd=0.02648 scope=full
CHECK: reservation-fits-cap ok (largest one-call reservation $0.1798)
GATE_PRICE: total=6.7 calls=200 purposes=answer,judge receipt=.gstack/tmp/price-gate/price-gate-<gate12>-<head12>.json
```

| 结果 | 退出 | 代码 |
|---|---|---|
| 定价,低于上限 | 0 | — |
| `lib/pricing.ts` 中没有行的 model:该行和总计打印 `unknown`,永远不是零 | 1 | `EVAL_PRICE_MISSING` |
| 总计高于 `--cap-usd` | 1 | `EVAL_GATE_OVER_CAP` |
| 一个调用的预留高于 `--cap-usd`,或对于 `scope=smoke` 行高于 `--smoke-cap-usd`(`CHECK: reservation-fits-cap exceeds (…)`) | 1 | `EVAL_GATE_OVER_CAP` |
| 没有声明计划(没有 `--plan-file` 和没有 `price_file` 键)或具有零调用行的文件 | 1 | `EVAL_GATE_UNPRICED` |
| 带有 `--allow-unpriced "<reason>"` 的相同内容 | 0 | 打印 `GATE_UNPRICED_OVERRIDE: <reason> (owner-authorized override; not authorization to spend)` 并记录它 |
| 格式错误或部分文件 | 1 | `EVAL_DRY_RUN_FAILED` |
| 过时的绑定,或声明的 `price_file` 从 HEAD 缺失(不可覆盖) | 1 | `EVAL_PLAN_STALE` |
| 读取 `price_file` 时 `origin/<base>` 不可解析 | 3 | `POLICY_SOURCE_UNAVAILABLE` |
| 基本上的 `.gstack/pregate.json` 无效 | 3 | `PREGATE_CONFIG_INVALID` |
| 未知标志或缺失 `--gate` | 2 | 使用 |

价格是一个估计:该表定价标准文本 token 并排除 `lib/pricing.ts` 中列出的计费类别(缓存写入、工具费、服务层和长上下文附加费、媒体 token、座位计量运行)。

## 收据

绑定检查之后的每个结果都会将 JSON 收据写入 `--out`(默认 `<repo>/.gstack/tmp/price-gate/`),命名为 `price-gate-<gate sha256 12>-<head 12>.json`,并在 `GATE_PRICE:` 行上打印其路径。`--json` 打印收据(加上 `receipt_path`)而不是行。

| 字段 | 含义 |
|---|---|
| `schema_version` | `1` |
| `kind` | `gstack-eval-price-gate` |
| `created_at`、`gstack_revision` | 何时,以及哪个 gstack(VERSION 和 commit)为其定价 |
| `gate_command`、`gate_sha256` | 已批准的命令及其 sha256 |
| `head_sha` | 定价时的 HEAD |
| `plan` | `file`、`source`(`flag`、`price_file`、`none`)、plan file 字节的 `sha256`、`declared_head` |
| `inputs` | 每个 `{ path, sha256 }`,在 HEAD 验证 |
| `preconditions` | 每个先决条件行 |
| `rows` | 每个调用行加上 `usd` 和 `reservation_usd`(未定价时为 `null`) |
| `total_usd`、`calls`、`purposes` | 总计;未知时 `total_usd` 为 `null` |
| `cap_usd`、`smoke_cap_usd` | 传递的上限,或 `null` |
| `result`、`code` | `priced`、`price_missing`、`over_cap`、`unpriced`、`unpriced_override`,以及结果代码 |
| `unpriced_reason`、`unpriced_override` | gate 为何未定价,以及所有者的覆盖原因 |
| `excluded_billing` | 估计排除的计费类别 |

## 参考发射器

消费仓库从构建其调用的相同代码编写文件,因此计数不能与运行漂移。`lib/eval-plan.ts` 中的 `formatEvalCalls` 是参考;复制它:

```ts
import { createHash } from 'node:crypto';
import { execFileSync } from 'node:child_process';
import { readFileSync } from 'node:fs';

const KEYS = ['purpose', 'model', 'calls', 'in_tokens', 'out_tokens', 'max_out_tokens', 'scope'] as const;
type Row = { purpose: string; model: string; calls: number; in_tokens: number; out_tokens: number; max_out_tokens: number; scope: 'full' | 'smoke' };

export function formatEvalCalls(p: { gate: string; head: string; inputs: { path: string; sha256: string }[]; preconditions: string[]; rows: Row[] }): string {
  return [
    '# gstack eval call plan, wire format v1 (docs/eval-calls.md); priced by `gstack-eval-plan price-gate`, never executed',
    `GSTACK_EVAL_GATE v1: ${p.gate}`,
    `GSTACK_EVAL_HEAD v1: ${p.head}`,
    ...p.inputs.map(i => `GSTACK_EVAL_INPUT v1: sha256=${i.sha256} path=${i.path}`),
    ...p.preconditions.map(t => `GSTACK_EVAL_PRECONDITION: ${t}`),
    ...p.rows.map(r => `GSTACK_EVAL_CALLS v1: ${KEYS.map(k => `${k}=${r[k]}`).join(' ')}`),
    `GSTACK_EVAL_CALLS v1 end: lines=${p.rows.length}`,
  ].join('\n') + '\n';
}

// Count tokens the way the run will, without calling a provider, then:
const head = execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim();
const input = 'evals/synopsis/questions.jsonl';
const sha256 = createHash('sha256').update(readFileSync(input)).digest('hex');
// writeFileSync('evals/gate.calls', formatEvalCalls({ gate, head, inputs: [{ path: input, sha256 }], preconditions, rows }));
}
```

按已提交的形式对输入进行哈希(HEAD 处的文件),在干净的 HEAD 上生成计划,在自己的提交中提交计划文件,并在 `.gstack/pregate.json` 中设置 `"price_file": "evals/gate.calls"`。

## 相关

- `gstack-eval-plan dry-run --cap <usd>` 为 eval 的 arms 运行相同的预留适合上限检查:一个调用(最大行的输入加上输出预算),已定价,必须适合上限下。
- 结果代码:[EVAL_GATE_UNPRICED](troubleshooting.md#eval-gate-unpriced),[EVAL_GATE_OVER_CAP](troubleshooting.md#eval-gate-over-cap),[EVAL_PLAN_STALE](troubleshooting.md#eval-plan-stale),[EVAL_PRICE_MISSING](troubleshooting.md#eval-price-missing),[EVAL_DRY_RUN_FAILED](troubleshooting.md#eval-dry-run-failed)。
