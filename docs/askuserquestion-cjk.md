# AskUserQuestion — 非 ASCII / CJK 字符

当 AskUserQuestion 包含中文(繁體/簡體)、日语、韩语或其他非 ASCII 文本时按需阅读此内容。操作规则在始终加载的 AskUserQuestion 自检中("Non-ASCII characters written directly, NOT \u-escaped");本文档是完整的理由。

## 规则

当任何字符串字段(问题、选项标签、选项描述)包含非 ASCII 文本时,在 JSON 字符串中发出字面 UTF-8 字符。**永远不要将它们转义为 `\uXXXX`。**

Claude Code 的工具参数管道是 UTF-8 原生的,字符原样传递。仅保留 JSON 强制转义:`\n`、`\t`、`\"`、`\\`。

## 为什么转义失败

手动转义需要从训练中回忆每个码位,这对于长 CJK 字符串是不可靠的 — 模型经常发出错误的码位。示例:写 `㄃` 认为它是 管 (U+7BA1),但 `㄃` 实际上是 ㄃,所以用户看到 `管理工具` 呈现为 `㄃3用箱`。

触发器是长的、多行的问题,带有数百个 CJK 字符:那正是反射性转义启动的时候,也正是错误编码最具破坏性的时候。长 ≠ 转义。保持字符为字面。

- 错误:`"question": "請選擇\uXXXX\uXXXX\uXXXX\uXXXX"`
- 正确:`"question": "請選擇管理工具"`
