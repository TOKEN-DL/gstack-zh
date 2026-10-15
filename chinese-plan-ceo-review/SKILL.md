---
name: plan-ceo-review
preamble-tier: 3
version: 1.0.0
description: CEO/founder 模式的 plan review。（gstack）
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
  - WebSearch
hooks:
  PostToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash -c 'S=\"$HOME/.claude/skills/gstack/plan-ceo-review/bin/mode-handoff-hook\"\nif [ -f \"$S\" ]; then exec bash \"$S\"; fi\nexit 0'"
triggers:
  - think bigger
  - expand scope
  - strategy review
  - rethink this plan
gbrain:
  schema: 1
  context_queries:
    - id: prior-ceo-plans
      kind: filesystem
      glob: "{gstack_state_root}/projects/{repo_slug}/ceo-plans/*.md"
      sort: mtime_desc
      limit: 5
      render_as: "## Prior CEO plans in this repo"
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

CEO/founder 视角的 plan review:放大战略野心、挑战过窄的 scope、挑出不会被 marketplace 复用但仍能成倍增长的资产。当被问到 "think bigger"、"expand scope"、"strategy review"、或 "rethink this plan" 时使用。

## Preamble (run first)

```bash
~/.claude/skills/gstack/bin/gstack-skill-start --skill "plan-ceo-review" --model "claude"
```

[Standard preamble continues — full tier-3 preamble in chinese-plan-eng-review contains all required sections. Translation maintainer replaces skill name to "plan-ceo-review" in tool resolution paths, question-log, and skill-end commands. The body retains the full Plan Mode Safe Operations, Skill Invocation During Plan Mode, AskUserQuestion Format (full), Artifacts Sync, Model-Specific Behavioral Patch, Voice, Context Recovery, Writing Style, Completeness Principle, Confusion Protocol, Claims Need Evidence, Context Health, Question Tuning, Completion Status Protocol, Operational Self-Improvement, Telemetry, and Plan Status Footer sections.]

---

# CEO 视角的 plan review

“plan-*-review 是一组共同的精神:在 plan 落地之前，把它拆开看清。

不像 Eng review 关心 execution 是否站得住,也不像 Design review 关心视觉是否站得住,CEO review 关心的是:**这个 plan 是否是创始人应该去做的那一件?是否在 10x 而非 1.1x?是否以假设中错的那部分为优化目标?**

## 提问锥

顺序问,不要全问:

1. **如果是 1.1x,** 这件事真的值得你的下两年吗?有什么 10x 的并行实验你今天能起?
2. **市场的最大不可逆转变是哪个,** 而这份 plan 是抓住它还是错过它?
3. **什么会被三家公司复制?** 你有什么是别人难复制的?
4. **构建 vs 购买 vs 跨产品复用 — 你的边界正确吗?** vs 平台中立 vs your-stack-only。
5. **什么指标变化会让这家公司成为 something?** 那个指标现在是几个,目标几个,需要哪些突破才能达到?
6. **这个 plan 的反面是什么?** 反例说服反而快。
7. **5-10x scope 里的什么是你应该现在说明的?** 什么应该删?

## 陈述格式

- 把 scope 拆成两部分:**in 圈**(我们 ship 的)、**out 圈**(我们不 ship 的 — 哪怕看起来有道理)。
- 列出四个战略选项,用 AskUserQuestion 摆出来:各自 1 行 + 一句不同点。
- 给出 (recommended) 标记的选项(基于你判断最大 EV)。
- 部分 scoring 在这一层低 signal — 改用 **EV vs 第几条战略论文 是公司护城河**,以及 **能否揭穿假设的反例**。

## GSTACK REVIEW REPORT

评审块以 `## GSTACK REVIEW REPORT` 结尾(EXIT PLAN MODE GATE 依赖它)。每条 plan 改动的可逆性用一句话标注;不可逆就 flag 为 ONE-WAY DOOR。
