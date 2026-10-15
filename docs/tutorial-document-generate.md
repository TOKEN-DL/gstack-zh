# 教程:在 90 秒内为功能生成文档

你将针对你已有的项目运行 `/document-generate`,观察它在正确的位置编写 tutorial / how-to / reference / explanation 文档,并以你可以放入 PR 的覆盖率映射结束。在结束时,你将了解四个动作:范围、考古、划分、编写。

## 你需要什么

- 已安装 gstack(`git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`)
- 在任何具有至少一个公共表面的项目中运行 Claude Code(CLI 命令、导出函数、配置选项、skill、API 端点)
- 约 90 秒

你不需要提前的 `docs/` 目录 — 如果缺失,该 skill 会创建一个。你不需要知道 Diataxis 术语 — 该 skill 为你标记输出。

## 步骤 1:在任何项目中调用该 skill

在你想要记录的项目中打开 Claude Code。输入:

```
/document-generate
```

你会看到该 skill 询问一个关于输出目标的问题:

```
A) Write documentation inline in existing files (README, ARCHITECTURE, etc.)
B) Create standalone documentation files (e.g., docs/ directory)
C) Both — inline summaries in existing files + deep docs in standalone files

RECOMMENDATION: Choose C because it maximizes both discoverability and depth.
```

选择 C。你将获得 README 指针加上完整的独立文档集。

## 步骤 2:观察考古运行

该 skill 在读取代码库时会静默约 30 秒。这是有意的 — Step 1 "Codebase Archaeology"阶段是工作流中最重要的步骤。该 skill 正在读取:

- 完整的仓库结构
- README、ARCHITECTURE、CONTRIBUTING、CLAUDE.md(入口点)
- 你正在记录的任何内容的实现文件(完整文件,而不仅仅是签名)
- 测试(它们揭示边界情况和预期行为)
- 标记 `// NOTE:`、`// DESIGN:`、`// WHY:` 的内联注释

完成后,你将看到类似以下的一行:

```
Researched 47 files, identified 12 public surface items, 8 concepts, and 4 design decisions.
```

该数字告诉你该 skill 实际上读取了代码,而不是从文件名猜测。

## 步骤 3:查看 Diataxis 划分计划

该 skill 打印一个划分计划,显示它将为哪个实体编写哪些象限:

```
Documentation plan:
  [entity]              [tutorial] [how-to] [reference] [explanation]
  WidgetService         ✅ new     ✅ new   ✅ new      ✅ new
  --verbose flag        ❌        ✅ new   ✅ inline   ❌
  Bayesian scheduler    ❌        ❌       ✅ new      ✅ new
```

并非每个实体都需要所有四个象限。CLI 标志获得 reference + how-to。内部模块获得 reference + explanation。面向用户的功能获得所有四个。该 skill 根据实体类型进行选择。

如果计划有超过 5 个文档,该 skill 会要求你在继续之前确认。否则它会继续。

## 步骤 4:阅读第一个文档

Reference 文档首先落地,因为它们修复了词汇。你将看到如下行:

```
GENERATED: docs/reference-widget-service.md
```

打开该文件。它具有严格的结构:一段引言、带有类型和默认值的完整 API 列表、2-3 个可运行的示例,以及一个 Related 部分,链接到接下来将落地的 how-to 和 tutorial。

这就是 Diataxis 中的 reference 文档的样子:事实、详尽、无叙述。如果你发现自己想要解释*为什么*存在某个选项,那属于该 skill 接下来要编写的 explanation 文档。

## 步骤 5:查看 explanation、how-to 和 tutorial 出现

快速连续地(每个约 5-10 秒),该 skill 编写剩余的象限:

```
GENERATED: docs/explanation-widget-architecture.md
GENERATED: docs/howto-create-a-custom-widget.md
GENERATED: docs/tutorial-build-your-first-widget.md
```

打开每一个。请注意它们不会相互重复:

- **Explanation** 以问题开头,然后是方法,然后是权衡和考虑的替代方案
- **How-to** 具有先决条件、带有确切命令的编号步骤、验证部分和故障排除部分
- **Tutorial** 在不到 3 个步骤内让你达到工作结果,以 "What you built" 结束

该 skill 强制执行这些结构。如果 how-to 缺少验证部分,Step 8 质量自检会在提交之前捕获它。

## 步骤 6:检查交叉链接

每个文档链接到其他文档。Reference 文档 Related 部分:链接到 how-to 和 tutorial。How-to Related 部分:链接到 reference。Tutorial "What you built" 部分:链接到 reference 以进行更深入的探索。

运行 grep 以验证没有断开的链接:

```bash
grep -rE '\]\([^)]*\.md\)' docs/ | head -10
```

每个链接的文件都应该存在。该 skill 的 Step 7 "Cross-Document Linking & Discoverability"在提交之前检查这一点。

## 步骤 7:在 PR 主体中查看覆盖率摘要

如果你在具有打开 PR 的功能分支上,该 skill 会使用 `## Documentation Generated` 表更新 PR 主体:

```
## Documentation Generated

| File | Quadrant | Description |
|------|----------|-------------|
| docs/tutorial-build-your-first-widget.md | Tutorial | Walk-through from install to first working widget |
| docs/reference-widget-service.md | Reference | Complete widget API with types, defaults, examples |
| docs/explanation-widget-architecture.md | Explanation | Why widgets are isolated services |
| docs/howto-create-a-custom-widget.md | How-to | Creating and registering custom widgets |
```

打开 PR 的审阅者会看到该表,并立即知道发布了哪种覆盖率。

## 你构建了什么

你现在有四个服务于四个不同读者的文档:

- 你项目的新来者可以阅读 `tutorial-*.md` 并获得可工作的东西
- 有经验的用户可以阅读 `howto-*.md` 来完成特定任务
- API 调用者可以阅读 `reference-*.md` 以获取确切的签名
- 代码审阅者可以阅读 `explanation-*.md` 来理解设计

每一个都足够短,易于维护。每一个都有单一的工作。PR 主体显示覆盖了哪些象限。如果你稍后运行 `/document-release`,Diataxis 覆盖率映射会将此实体报告为完全覆盖(4/4 象限)。

## 接下来做什么

- **如果你有空白** /document-release 标记但未填补:再次运行 `/document-generate`,专门针对这些实体的范围。
- **如果你想了解为什么存在四个象限:** 阅读 [explanation-diataxis-in-gstack.md](./explanation-diataxis-in-gstack.md)。
- **如果你想在发布前记录一个特定功能**(而不是整个项目):阅读 [howto-document-a-shipped-feature.md](./howto-document-a-shipped-feature.md)。
- **该 skill 本身的参考:** [`document-generate/SKILL.md`](../document-generate/SKILL.md)。
