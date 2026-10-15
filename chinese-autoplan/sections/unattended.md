<!-- AUTO-GENERATED from unattended.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## Unattended profile

`SESSION_KIND: unattended`:父 agent 通过 `~/.claude/skills/gstack/bin/gstack-autoplan`(`next`/`submit`/`answer --gate-rev`/`export`;docs/unattended.md)驱动本次运行,只读取 artifact,不读散文。阶段自关闭,Eng 在最后跑,outside voice 通过 `gstack-outside-voice` 跑;运行会在 `status=gate_pending` 处停下,不批准它。`gstack-artifact validate <dir>/run.json` 是完成度检查。最后以 `GSTACK_RESULT: skill=autoplan status=<status> run=<dir>` 结尾。