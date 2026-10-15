---
name: office-hours
preamble-tier: 3
version: 2.0.0
description: YC Office Hours —— 两种模式。（gstack）
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Write
  - Edit
  - AskUserQuestion
  - WebSearch
triggers:
  - brainstorm this
  - is this worth building
  - help me think through
  - office hours
gbrain:
  schema: 1
  context_queries:
    - id: prior-sessions
      kind: list
      filter:
        type: ceo-plan
        tags_contains: "repo:{repo_slug}"
      sort: updated_at_desc
      limit: 5
      render_as: "## Prior office-hours sessions in this repo"
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

YC 风格的 Office Hours:两种模式,取决于用户在向谁汇报。当被问到 "brainstorm this"、"is this worth building"、"help me think through"、或 "office hours" 时使用。

## Preamble (run first)

```bash
~/.claude/skills/gstack/bin/gstack-skill-start --skill "office-hours" --model "claude"
```

[Standard tier-3 preamble continues — full preamble in chinese-plan-eng-review contains all required sections: Plan Mode Safe Operations, Skill Invocation During Plan Mode, AskUserQuestion Format (full), Artifacts Sync, Model-Specific Behavioral Patch (claude), Voice, Context Recovery, Writing Style, Completeness Principle, Confusion Protocol, Claims Need Evidence, Context Health (soft directive), Question Tuning, Completion Status Protocol, Operational Self-Improvement, Telemetry (run last), and Plan Status Footer sections. Translation maintainer replaces skill name to "office-hours" in tool paths, gstack-question-log, and gstack-skill-end commands.]

---

# YC Office Hours —— 两种 mode

gstack 通过 `/office-hours` 提供两种 mode 的对话,基于它推断你在向谁汇报。

## Mode 1:First-time founder

你是第一次坐下来的创始人,用空白 doc 想出一件值得做的产品。流程:
1. **5-问题 interview**(each ELI10 + sub-bullets):你的问题、谁在乎、为什么是现在、为什么是你、什么算好。
2. 把答案塞进已知的 YC framework 模板 —— 见 `~/.claude/skills/gstack/docs/yc-template.md`。
3. 列出 3 个你想过的替代方案,再用 YC "10x better" question 对每个方案过一遍。
4. **PRESENT MODE** = 你要用 5 分钟讲完这套。生成一段 ≤ 200 字的话 + 一份能让 partner 立刻上手的 checklist。

## Mode 2:Repeat founder (有 past projects)

你的请求会通过 gbrain 把过去的 office-hours sessions(same repo)加载 —— `## Prior office-hours sessions in this repo` 会作为上下文拉进来。然后:
1. 一句话重新解释一下你回到这里。
2. 上一节的关键决定列出 —— 把这次定位成上次的延续。
3. 重读一遍 5-问题,但只问对新的 piece 必要的问题。
4. 用 `/plan-ceo-review` 的语言,但模式更短 —— 5 分钟,而非完整 review。

## Mode 切换

gstack 根据你的输入、branch 名字、CLAUDE.md 注释等信号推断 mode。如果它推错了,用 AskUserQuestion 立刻纠正:

> Office hours mode:
> - **A) First-time founder** (没有 past projects on this repo)
> - **B) Repeat founder** (有 sessions 在 ceo-plans/)
> - **C) Investor-style** (你代表别人 interview founder)
