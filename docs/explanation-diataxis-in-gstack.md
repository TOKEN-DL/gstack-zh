# 为什么 gstack 对文档使用 Diataxis

gstack 中的两个文档 skill — `/document-release` 和 `/document-generate` — 都使用 Diataxis。新实体在四个象限中进行评分。覆盖率空白以象限标记在 PR 主体中出现。本文档解释为什么该词汇是承重的,以及为什么在 gstack 运营的规模上更简单的"只写 markdown"方法会失败。

## 问题

文档腐烂是最容易忽视的腐烂种类。代码停止编译,你会立即注意到。测试失败,CI 会尖叫。文档默默地过时 — README 仍然解析,安装命令仍然可以复制粘贴 — 唯一的信号是几周后提交问题或悄悄走开的困惑用户。

gstack 有超过 45 个 skill。每一个都是一个 SKILL.md 加一个 `.tmpl` 模板,加上(理想情况下)某处的入门教程以及为什么它以这种方式工作的解释。将其乘以 gstack 用户在自己项目中具有类似表面区域的数量,维护负担是真实的。

天真的失败模式是"每个团队以自己的格式编写文档"。一个项目有一个 Wiki。另一个有嵌套的 README 文件。第三个有仅引用的 API 文档,没有教程。第四个有不再编译的教程。你无法编写跨所有这些进行审计的工具,因为没有关于什么是良好覆盖的共享词汇。

第二种失败模式更微妙:即使团队有纪律,他们也倾向于编写与他们当前心态匹配的文档。构建模式下的工程师编写 reference。启动模式下的工程师编写教程。维护模式下的工程师编写故障排除 how-tos。没有人醒来后说"今天我要为我们选择这个架构的原因编写解释文档" — 所以解释腐烂累积最快。

## 方法

Diataxis(Daniele Procida,最初在 Divio,现在被 CPython、Django、NumPy、FastAPI、GitHub 文档等采用)根据**读者意图**将文档分为四个象限:

```
                    THEORETICAL                        PRACTICAL
                    (understanding)                    (doing)

  STUDY            +-----------------------------+----------------------------+
  (learning)       |                             |                            |
                   |   EXPLANATION               |   TUTORIAL                 |
                   |   "Why does X exist?"       |   "Walk me through X       |
                   |                             |    for the first time"     |
                   |   discusses code            |   teaches code             |
                   |                             |                            |
                   +-----------------------------+----------------------------+

  WORK             +-----------------------------+----------------------------+
  (using)          |                             |                            |
                   |   REFERENCE                 |   HOW-TO                   |
                   |   "What is the exact        |   "How do I accomplish Y   |
                   |    signature of Y?"         |    using X?"               |
                   |                             |                            |
                   |   describes code            |   uses code                |
                   |                             |                            |
                   +-----------------------------+----------------------------+
```

处于 tutorial 模式的读者通过做来学习。他们想要保证成功的引导路径。处于 how-to 模式的读者已经知道基础知识,并想要完成特定任务的配方。处于 reference 模式的读者想要 API 的准确、完整的事实表覆盖。处于 explanation 模式的读者想要理解设计决策。

同一个人在不同的时间从每个模式阅读一个项目。同一段落不能服务所有四个 — 教程需要的handholding会减慢 reference 读者的速度;reference 需要的完整性会让教程读者不知所措。

## 为什么这作为覆盖镜头很重要

用 Diataxis 术语编写的覆盖率映射为你提供了"文档是否已更新?"的确定性答案 — 不是"是否有 README",而是"是否有此新技能的教程、是否有常见任务的 how-to、是否有 API 的 reference,以及是否有针对不明显设计选择的 explanation"。

`/document-release` 步骤 1.5 遍历 diff,提取新的公共表面(skill、CLI 标志、配置选项、API 端点),并在四个象限中对每个实体进行评分。零覆盖率的项成为**关键空白**。仅有 reference 覆盖率(gstack 自己历史中最常见的失败模式)的项成为**常见空白**。两者都落在审阅者看到的 PR 主体中。

`/document-generate` 有意在四个象限中编写文档。它拒绝混合它们:tutorial 没有"配置"部分,reference 文档没有"What you'll build"段落。该 skill 的 9 个步骤按 reference → explanation → how-to → tutorial 的顺序进行,因为该顺序匹配依赖关系:reference 修复词汇,explanation 证明设计,how-tos 建立在两者之上,tutorial 是最后和最难的。

## 权衡

**Diataxis 添加读者必须学习的词汇。** 从未听说过"reference 与 explanation"的用户可能会发现标签起初很奇怪。缓解措施是 Diataxis 标签在看过一次后是不言自明的,并且标签永远不会出现在文档本身中 — 它们出现在覆盖率映射和 PR 主体中,审阅者在那里看到它们,而不是最终用户。

**四个文件而不是一个。** 一个小型 skill 可能有一个混合所有四种模式的 `docs/SKILL.md` 文件。Diataxis 将其分为四个。缓解措施:AI 生成使四文件结构变得便宜,象限之间的交叉链接是机械的(每个 reference 文档链接到其 how-to,每个 how-to 链接到其 reference 等),并且审计能力的收益是实质的 — `/document-release` 可以自动评分覆盖率。

**Diataxis 不是唯一的好框架。** "Every page is page one"(Mark Baker)、*Write the Docs* 社区中的四种文档、Google 开发者文档风格指南 — 都有不同的切法。gstack 选择 Diataxis 是因为它具有最强的外部采用(CPython、Django、NumPy、FastAPI 等),这意味着下游用户最有可能以前见过该词汇,并且象限标签清晰地转换为覆盖率映射信号。

## 考虑的替代方案

**"只写 README 部分。"** 在 gstack 的历史中隐含地尝试过。失败模式:教程在 README 中累积,直到 README 是 800+ 行,没有人读过第 50 行之后。Diataxis 将它们分为专用文件,每个文件都可以从 README 的目录中发现。

**自定义内部分类法。** 诱人,因为它可以量身定制。被拒绝,因为每个团队都会发明自己的词汇,而 `/document-release` 将失去其跨项目审计能力。Diataxis 是通用语。

**仅自动生成的 reference。** 通过 JSDoc / TypeDoc / Sphinx 等工具在许多项目中尝试过。没有 explanation 的 reference 文档对新人变得难以理解;没有 tutorial,API 难以上手。Reference 是必要的但不充分。

**根本没有文档框架,只是直觉检查。** 大多数项目的现状。静默失败 — 用户走开而不是提交问题,因此反馈循环被破坏。Diataxis 在用户抱怨之前提供结构化信号。

## 相关

- **实现此的 skill 的参考:** [`document-generate/SKILL.md`](../document-generate/SKILL.md)
- **使用此分类法的审计的参考:** [`document-release/SKILL.md`](../document-release/SKILL.md)
- **使用 `/document-generate` 的教程:** [tutorial-document-generate.md](./tutorial-document-generate.md)
- **How-to:在发布前记录功能:** [howto-document-a-shipped-feature.md](./howto-document-a-shipped-feature.md)
- **Diataxis 主页:** https://diataxis.fr/ — Procida 的框架规范参考
