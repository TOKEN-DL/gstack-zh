# 第三方声明(Third-Party Notices)

gstack 是 MIT 协议。下方列出的文件包含源自 Apache-2.0 作品、以 gstack 自己的口吻重写并修改的素材。重写并非豁免:协议文本位于 `licenses/Apache-2.0.txt`,每个派生文件都附有「已变更」的声明。未修改的副本(规则注册中心 fixture、协议文本)不带头部,在此列出。

## impeccable — 版权归 Paul Bakaus 所有 — Apache License 2.0

https://github.com/pbakaus/impeccable

派生、修改:

- `lib/design-catalog.ts`:规则 id 与名称来自 `crates/live/assets/antipatterns.json`;散文是 gstack 的。
- `scripts/resolvers/design.ts`:Persuade / Operate / Read / Experience 访客模式,craft-floor 的反射(浏览器表面、一个被编排的动效瞬间、深度有偏移、二级文本带色调、标题上方留白、明暗由使用场景决定),以及三种外观校准,来自 `SKILL.md`、`reference/craft-floor.md` 与 `reference/new-work.md`,已经重写。
- `design-consultation/sections/proposal-and-preview.md.tmpl`:字体选择流程与校准段落,已重写;以及由这些源生成的 `SKILL.md` / `sections/*.md` 文件。

未修改副本:

- `test/fixtures/impeccable-antipatterns.json`:`crates/live/assets/antipatterns.json` 在 commit 87d8f6d6(engine-v0.1.3)的版本,被包裹在 `_source` 出处对象中。

未分发:`bin/gstack-design-detect.ts` 调用的是用户自行安装的 impeccable 引擎。gstack 既不分发也不镜像该引擎,也不会运行 impeccable 的安装器或启动器。gstack 唯一可进行的下载,是该引擎二进制本身——仅在用户接受某个设计技能的一次性提议之后:从 impeccable 自己的 GitHub release 取到 `~/.impeccable/bin/<version>/`,与 `lib/design-detect-contract.ts` 中固定的校验和核对,并先在出账台账中记录。gstack 不审计该引擎的网络行为;包装器拒绝 URL 目标,因此 gstack 从不指示它接触网络。

## DESIGN.md 规范 — 版权归 Google LLC 所有 — Apache License 2.0

https://github.com/google-labs-code/design.md

`lib/design-md.ts`、`bin/gstack-design-md.ts` 与 `design-consultation/sections/proposal-and-preview.md.tmpl` 中的 Phase 6 模板,实现了该格式(分五组的 YAML token frontmatter、按规范顺序的八个规范章节、`{path}` token 引用)。未复制任何规范文本。
