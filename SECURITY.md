# 安全策略(Security Policy)

## 报告漏洞

请私下报告安全漏洞,不要在公开 issue、PR 或讨论中报告。

**首选方式:** 使用 GitHub 私有漏洞报告:
<https://github.com/garrytan/gstack/security/advisories/new>

请尽可能附上:受影响文件、技能或命令;gstack 版本(在你的安装中运行 `cat VERSION`,或 `/gstack-upgrade` 的输出);复现步骤;以及你观察到的后果。

**如果该链接无效:** 开一个最小的公开 issue,标题为「Request for a private security contact」。不要在其中包含任何技术细节、受影响文件或 PoC。维护者会回复一条私密通道,你可以在那里发送细节。

## 范围(Scope)

gstack 是一组在你自己的机器上、用你自己的凭据运行的技能、脚本与辅助二进制。仓库中代码的漏洞报告在范围内:技能模板与生成的 `SKILL.md` 文件、`bin/`、`lib/`、`browse/`、`scripts/`、安装程序(`setup`)以及 GitHub Actions workflows。gstack 驱动的第三方工具(Claude Code、Codex、gbrain、scanner、浏览器)的漏洞应上报到对应项目。

## 支持的版本

修复落在 `main` 上,并随下一个版本发布。运行 `/gstack-upgrade` 获取修复;较旧版本不单独打补丁。
