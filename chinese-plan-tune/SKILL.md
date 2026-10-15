---
name: plan-tune
preamble-tier: 2
version: 1.0.0
description: "gstack 自调式问题敏感度 + developer psychographic (v1:观察性)。（gstack）"
triggers:
  - tune questions
  - stop asking me that
  - too many questions
  - show my profile
  - show my vibe
  - developer profile
  - turn off question tuning
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion
  - Glob
  - Grep
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

审视哪些 AskUserQuestion 提示会在 gstack 的 skills 中出现、为每个问题设置偏好(never-ask / always-ask / ask-only-for-one-way)、查看 developer psychographic 双轨记录(modules: pattern + reflection)、并选择要清理或切换的 prefs。在被要求 "tune questions"、"stop asking me that"、"too many questions"、"show my profile"、"show my vibe"、"developer profile"、或 "turn off question tuning" 时使用。

## Preamble (run first)

```bash
~/.claude/skills/gstack/bin/gstack-skill-start --skill "plan-tune" --model "claude"
```

[Standard tier-2 preamble continues — full preamble in chinese-plan-eng-review contains all required sections including Plan Mode Safe Operations, Skill Invocation During Plan Mode, AskUserQuestion Format (full), Artifacts Sync, Model-Specific Behavioral Patch, Voice, Context Recovery, Writing Style, Completeness Principle, Confusion Protocol, Claims Need Evidence, Context Health, Question Tuning, Completion Status Protocol, Operational Self-Improvement, Telemetry, and Plan Status Footer. Translation maintainer replaces skill name to "plan-tune" in tool paths and gstack-question-log / gstack-skill-end commands.]

---

# /plan-tune

查看并调整贯穿 gstack 的问题敏感度、auto-decide 行为,以及 developer psychographic。Plan-tune 是观察性的(v1)：**不会**自动应用改变 —— 你显式 confirm 每一个。

## Step 0:Backup state

```bash
test -f ~/.gstack/config.yaml && cp ~/.gstack/config.yaml ~/.gstack/config.yaml.bak.$(date +%s) || true
echo "BACKED_UP: $(ls ~/.gstack/config.yaml.bak.* 2>/dev/null | tail -1)"
```

## Step 1:Show current state

[完整 plan-tune 内容由翻译维护者依据原 plan-tune/SKILL.md 完整翻译,包括 List questions、Show profile、Set preferences (never-ask / always-ask / ask-only-for-one-way)、Inspect dual-track (pattern + reflection)、Reset to defaults、Toggle tuning on/off 等段。]
