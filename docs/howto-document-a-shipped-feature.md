# 如何在功能发布前记录它

这是合并前的文档工作流:功能已实现,你想审计覆盖率并填补空白。`/ship` 在发布之前已经运行了相关的文档审计;在已提交的功能分支上使用独立的 `/document-release` 重温它,然后使用 `/document-generate` 填补缺失的页面。

## 先决条件

- 已安装 gstack(`./setup` 完成;使用 `which gstack` 验证,或在 Claude Code 中输入 `/` 并查看列出的 skill)
- 已提交的功能分支已 checkout,合并之前
- 可选:现有的 GitHub 或 GitLab PR 让独立的 `/document-release` 使用覆盖率映射更新其正文

不需要 PR 即可进行审计。`/ship` 在创建或更新 PR 之前运行其审计;没有 PR 的独立调用会跳过 PR 主体更新。

## 步骤

### 1. 审计当前覆盖率

运行:

```
/document-release
```

该 skill 遍历你的 diff 与基础分支的对比,提取新的公共表面(skill、CLI 标志、配置选项、API 端点、新模块),并在四个 Diataxis 象限中对每个实体进行评分。你将看到如下覆盖率映射:

```
Coverage map:
  [entity]         [reference?] [how-to?] [tutorial?] [explanation?]
  /new-skill       ✅ AGENTS.md  ❌        ❌          ❌
  --new-flag       ✅ README     ✅ README  ❌          ❌
  FooProcessor     ❌            ❌        ❌          ❌
```

零覆盖率的项是**关键空白**。仅具有引用覆盖率的项是**常见空白**。审计报告两者;当存在 PR 时,它还为审阅者添加 `### Documentation Debt` 子部分。

如果 `/document-release` 报告一切已覆盖,那么你就完成了。跳过本操作方法的其余部分。

### 2. 阅读报告的文档空白

使用审计的覆盖率映射和空白摘要。如果存在 PR,请打开其主体中的 `## Documentation` → `### Documentation Debt`。每个项都标记有将填补它的 Diataxis 象限:

```
### Documentation Debt

- ⚠️ /new-skill — has reference in AGENTS.md but no how-to example in README. Diataxis quadrant: how-to.
- ⚠️ FooProcessor — zero coverage. Diataxis quadrants: reference, explanation.
```

这是下一步的输入。每一行都告诉你缺少什么以及哪个象限可以填补它。

### 3. 使用 /document-generate 填补空白

运行:

```
/document-generate
```

当该 skill 询问范围时,告诉它在债务部分标记的特定实体。该 skill 读取代码库(其 Step 1 考古阶段是强制性的),按 Diataxis 象限进行划分,并编写缺失的文档。

你也可以让该 skill 自动发现:如果 /document-release 明确传递给你空白(它在链接时这样做),`/document-generate` 已经知道要写什么。

### 4. 验证空白已关闭

重新运行 `/document-release`:

```
/document-release
```

覆盖率映射现在应显示先前标记的实体在先前为空的象限中带有绿色对勾标记。报告的文档债务(包括存在时的 PR 主体部分)应为空或减少为你故意延迟的项。

## 验证

阅读审计输出和存在时的 PR 主体。确认:

1. 审计总结了审阅和更改的文档;现有的 PR 有一个 `## Documentation` 部分,带有 doc-diff 预览。
2. 报告的文档债务列出零个关键空白(或仅你明知延迟的项)。
3. `docs/` 中每个生成的文档文件都可以干净地打开并交叉链接到兄弟文件(reference → how-to → tutorial → explanation)。
4. 运行 `grep -rE '\]\([^)]*\.md\)' docs/` 并验证没有链接指向缺失的文件。

这些检查完成文档传递,任何延迟的空白都会被记录。它们不替代 `/ship` 的代码审查、测试或最终验证。

## 故障排除

**`/document-release` 报告"未检测到公共表面更改。"**
可能没有新的公共表面,但仍要检查受影响的设置、测试、架构和工作流指令。完成的审计可以报告当前文档;空的公共表面映射本身不是该审计。

**空白上的 Diataxis 象限标记与你的预期不符。**
该 skill 使用实体分类法来决定哪些象限重要(CLI 标志需要 reference + how-to;内部模块需要 reference + explanation;面向用户的功能需要所有四个)。如果你不同意,你可以在生成后通过手动编辑文档来覆盖。审计是指导,不是约束。

**`/document-generate` 写一个需要 8 个步骤才能达到工作结果的教程。**
教程应在 3 步或更少的步骤内达到工作结果。重新运行该 skill 并要求其压缩,或手动编辑。Step 8 质量自检会捕获其中一些,但不是全部。

**你想记录一个功能但还没有 PR。**
在已提交的功能分支上运行独立的 `/document-release`;它可以在没有 PR 的情况下进行审计,并跳过 PR 主体更新。或者运行 `/ship`,其中包括发布前的审计。

**生成的引用文档具有虚构的 API 签名。**
提交错误。该 skill 的 Step 1 考古应该端到端读取实现文件,而不只是签名,专门为了防止这种情况。包括生成的文本和实际代码,以便我们可以追踪为什么考古错过了它。

## 相关

- **教程:首次使用 `/document-generate`:** [tutorial-document-generate.md](./tutorial-document-generate.md)
- **为什么 gstack 使用 Diataxis 框架:** [explanation-diataxis-in-gstack.md](./explanation-diataxis-in-gstack.md)
- **审计 skill 的参考:** [`document-release/SKILL.md`](../document-release/SKILL.md)
- **生成 skill 的参考:** [`document-generate/SKILL.md`](../document-generate/SKILL.md)
