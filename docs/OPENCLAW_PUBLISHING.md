# 将原生 OpenClaw 技能发布到 ClawHub

逐字从 CLAUDE.md 移出(减少 token 加载)。

## 工作流

原生 OpenClaw 技能位于 `openclaw/skills/gstack-openclaw-*/SKILL.md`。这些是手工制作的方法论技能(不是由流水线生成的),发布到 ClawHub,以便任何 OpenClaw 用户都可以安装它们。

**发布:** 命令是 `clawhub publish`(不是 `clawhub skill publish`):

```bash
clawhub publish openclaw/skills/gstack-openclaw-office-hours \
  --slug gstack-openclaw-office-hours --name "gstack Office Hours" \
  --version 1.0.0 --changelog "description of changes"
```

对每个技能重复:`gstack-openclaw-ceo-review`、`gstack-openclaw-investigate`、`gstack-openclaw-retro`。每次更新时增加 `--version`。

**认证:** `clawhub login`(打开浏览器进行 GitHub 认证)。`clawhub whoami` 进行验证。

**更新:** 使用更高的 `--version` 和 `--changelog` 运行相同的 `clawhub publish` 命令。

**验证:** `clawhub search gstack` 确认它们已上线。
