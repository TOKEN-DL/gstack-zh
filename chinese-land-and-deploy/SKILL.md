---
name: land-and-deploy
preamble-tier: 4
version: 1.0.0
description: Landing 与 deploy 工作流。（gstack）
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion
triggers:
  - merge and deploy
  - land the pr
  - ship to production
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

合并 PR,等待 CI 和 deploy,通过 canary 检查来校验生产健康。在 /ship 创建 PR 之后接过来。当用户说 "merge"、"land"、"deploy"、"merge and verify"、"land it",或 "ship it to production" 时使用。

## Preamble (run first)

```bash
~/.claude/skills/gstack/bin/gstack-skill-start --skill "land-and-deploy" --model "claude"
```

[Standard tier-4 preamble continues — full preamble in chinese-plan-eng-review contains all required sections: Plan Mode Safe Operations, Skill Invocation During Plan Mode, AskUserQuestion Format (full), Artifacts Sync, Model-Specific Behavioral Patch (claude), Voice, Context Recovery, Writing Style, Completeness Principle, Confusion Protocol, Claims Need Evidence, Context Health (soft directive), Question Tuning, Completion Status Protocol, Operational Self-Improvement, Telemetry (run last), and Plan Status Footer sections. Translation maintainer replaces skill name to "land-and-deploy" in tool paths, gstack-question-log, and gstack-skill-end commands.]

---

# /land-and-deploy

把 /ship 创建的 PR 合并、监控 CI、验证生产 deploy。会接管 /ship 末尾的子流程。

## Phase 1:确认 PR 状态

确认 PR 还开放,review 通过,所有必需的 CI check 通过。如果任何一项未通过,暂停并说明。

## Phase 2:合并

按 base 仓库的 merge 约定(squash、merge commit、rebase)。选用 squash + 自动 delete branch 是最常见的现代默认值。

## Phase 3:等待 CI

生产 deploy 一般被合并触发,或被下面 CI 流程触发(GitHub Actions、Vercel 等)。侦测可用信号 —— deploy status URL、PR commit statuses —— 不要瞎轮询。

## Phase 4:Canary + 健康检查

如果部署到多实例,确认至少一个 canary 收到新代码,并且 health check 通过(200 或所配置的成功响应)。一旦接收率(ingestion rate)、错误率、p99 latency 都健康,就宣告完成。

## Phase 5:回报

简短、有数字。What shipped、where to verify、what to watch。

[Step 0 时先通过 `gstack-config get auto_deploy_enabled` 校验 auto-deploy 是否开。]
