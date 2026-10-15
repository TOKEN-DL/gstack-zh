---
name: document-generate
preamble-tier: 2
version: 1.0.0
description: Diataxis 文档生成器:教程、how-to、参考、解释。(gstack)
triggers:
  - document this feature
  - write docs for
  - generate documentation
  - fill documentation gaps
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl ; do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

[本节中以下段落翻译自 gstack SKILL 模板:何时调用、序章(Preamble,先运行)、Plan Mode Safe Operations、Skill Invocation During Plan Mode、AskUserQuestion 格式(完整)、Artifacts Sync、Model-Specific Behavioral Patch、Voice、Context Recovery、Writing Style、Completeness Principle ; Boil the Ocean、Confusion Protocol、Claims Need Evidence、Context Health、Question Tuning、Completion Status Protocol、Operational Self-Improvement、Telemetry(最后运行)、Plan Status Footer、Step 0:Detect platform and base branch。完整内容已由 chinese-autoplan/SKILL.md 译出模板,本文件遵循同一模板,正文从 Step 0 后的 # Document Generate 部分开始翻译。]

---

# Document Generate:Diataxis 文档编写器

你正在运行 `/document-generate` 工作流。你的工作是:为功能、模块或整个项目生成**高质量的、结结构化的文档**。在编写任何一行文档之前,你先彻底研究代码。

可以通过两种方式调用此技能:
1. **独立调用** ; 用户指向某个功能、模块或项目并说「document this」
2. **从 /document-release 调用** ; 覆盖图识别出缺口;你填补它们

你遵循 **Diataxis 框架** ; 文档的四个象限,每个都服务于不同的读者需求:
- **Tutorial** ; 面向学习,逐步骤引导新手完成一个工作示例
- **How-to** ; 面向任务,展示如何完成特定目标(假设读者具备基本熟悉度)
- **Reference** ; 面向信息,完整且准确的技术描述
- **Explanation** ; 面向理解,解释为什么事情以这种方式工作

**理念:先研究整体,再写部分。** 像建筑师在绘制任何一间房间之前先勘察整个场地一样,你在编写任何文档之前先阅读完整的代码库表面。这可以防止「描述了一半功能的文档」这种失败模式。

---

## 第 0 步:范围与意图

1. 确定要记录什么:
   - **如果用特定目标调用**(功能、模块、文件、技能):范围是该目标
   - **如果为整个项目调用**:范围是整个项目
   - **如果从带有关键文档的 /document-release 调用**:范围是覆盖图中特定实体

2. 使用 AskUserQuestion 确认范围并询问文档目标:

   - A) 在现有文件中内联编写文档(README、ARCHITECTURE 等)
   - B) 创建独立的文档文件(例如 `docs/` 目录)
   - C) 两者 ; 在现有文件中内联摘要 + 在独立文件中深度文档

   推荐:选择 C,因为它同时最大化可发现性和深度。

3. 确定输出格式:
   - 如果项目已经有 `docs/` 目录,请遵循其约定
   - 如果项目使用文档框架(Nextra、Docusaurus、MkDocs、VitePress),跟随其格式
   - 否则,在 `docs/` 中使用纯 Markdown 文件

---

## 第 1 步:代码库考古(研究阶段)

文档的质量取决于你对代码的理解程度,因此这一步最为重要。

### 1. 映射项目结构:

```bash
find . -type f -not -path "./.git/*" -not -path "./node_modules/* */ \.gstack/*" -not -path "./dist/*" -not -path "./build/*" -not -path "./.next/*" | head -200
```

### 2. 阅读入口点。识别并阅读:
   - README.md、ARCHITECTURE.md、CONTRIBUTING.md、CLAUDE.md / AGENTS.md
   - package.json / Cargo.toml / pyproject.toml / go.mod(了解项目类型)
   - 主入口文件(index.ts、main.rs、app.py、cmd/main.go)
   - 配置文件和示例

### 3. 阅读每个目标实体的源代码。对于你要记录的每个功能/模块:
   - 完整地阅读实现文件(不仅仅是签名)
   - 阅读测试 ; 它们揭示预期行为、边缘情况和使用模式
   - 阅读目标依赖或被依赖的相关模块
   - 阅读任何现有的内联注释,特别是 `// NOTE:`、`// DESIGN:`、`// WHY:`

### 4. 构建概念图。在编写之前,产生一个内部大纲:

```
Target: [feature/module name]
Purpose: [one sentence; what problem does it solve?]
Key concepts: [list the 3-5 concepts a reader must understand]
Public surface: [commands, functions, config options, API endpoints]
Dependencies: [what it needs from other modules]
Dependents: [what relies on it]
Edge cases: [from reading tests and code]
Design decisions: [any non-obvious "why" choices]
```

### 5. 输出:「Researched N files, identified K public surface items, M concepts, and J design decisions.」

---

## 第 2 步:Diataxis 分区

对于每个目标实体,决定要生成哪些 Diataxis 象限。并非每个实体都需要全部四个象限。

**决策矩阵:**

| 实体类型 | Tutorial? | How-to? | Reference? | Explanation? |
|---|---|---|---|---|
| 用户交互的新功能 | ✅ | ✅ | ✅ | Maybe |
| CLI 命令或 flag | Maybe | ✅ | ✅ | No |
| 内部模块/架构 | No | No | ✅ | ✅ |
| 配置选项 | No | ✅ | ✅ | No |
| 设计模式 / 哲学 | No | No | No | ✅ |
| API 端点 | Maybe | ✅ | ✅ | No |
| 工作流(多步骤流程) | ✅ | ✅ | No | Maybe |

输出分区计划:

```
Documentation plan:
  [entity]              [tutorial] [how-to] [reference] [explanation]
  Widget system         ✅ new     ✅ new   ✅ new      ✅ new
  --verbose flag        ❌        ✅ new   ✅ inline   ❌
  Bayesian scheduler    ❌        ❌       ✅ new      ✅ new
```

如果计划要创建超过 5 个文档,请使用 AskUserQuestion 确认后再继续。
对于较小的范围,请直接进行。

---

## 第 3 步:先写参考文档

参考文档是基础。它们是事实性的、完整的、直接从代码派生的。
在教程或 how-to 之前写这些,因为它们建立了词汇表。

**参考文档模板:**

```markdown
# [Entity Name]

[One paragraph: what it is, what it does, when you'd use it.]

## API / Interface

[Complete listing of public surface: functions, commands, config options, parameters.
Include types, defaults, and constraints. Pull directly from code; do not paraphrase
loosely.]

## Options / Configuration

[If applicable: every option with its type, default, and effect.]

## Examples

[2-3 concrete examples showing actual usage. Prefer real command output or code that
would actually compile/run.]

## Related

[Links to other reference docs, how-tos, or explanations that provide context.]
```

**参考文档规则:**
- 准确性优先于优雅。每个声明都必须可追溯到代码。
- 包括类型、默认值和约束。「接受字符串」是不够的 ; 「接受字符串(最多 256 个字符,必须匹配 `^[a-z-]+$`)」才是参考级别。
- 显示真实示例,如果复制粘贴的话实际上可以工作。
- 不要解释*为什么* ; 那属于解释文档。

---

## 第 4 步:编写解释文档

解释文档回答「为什么事情以这种方式工作?」它们是设计原理。

**解释文档模板:**

```markdown
# [Concept / Design Decision]

[Opening paragraph: the problem this design solves, stated in terms a smart reader
who hasn't seen the code would understand.]

## The problem

[Concrete description of what goes wrong without this design. Real failure modes,
not abstract risks.]

## The approach

[How the design solves the problem. Include diagrams (ASCII or Mermaid) for
architectural concepts.]

## Trade-offs

[What was given up. Every design decision trades something; name it explicitly.]

## Alternatives considered

[If discoverable from code comments, ADRs, or git history: what was tried or
rejected and why.]
```

**解释文档规则:**
- 以问题开头,而不是解决方案。
- 对架构使用 ASCII 图。它们可 grep、可 diff-friendly、在任何地方渲染。
- 明确命名权衡。「我们选择 X 而不是 Y 是因为 Z」是黄金标准。
- 不要重复参考材料 ; 链接到它。

---

## 第 5 步:编写 How-To 指南

How-to 是面向任务的:它们假设读者了解基础知识并希望完成特定任务。

**How-to 文档模板:**

```markdown
# How to [accomplish specific task]

[One sentence: what you'll accomplish and the end result.]

## Prerequisites

[What the reader needs before starting. Be specific; versions, installed tools,
config state.]

## Steps

1. [Action verb] [specific instruction]

   ```bash
   [exact command]
   ```

   [Expected output or result, if non-obvious.]

2. [Next step...]

## Verification

[How to confirm it worked. A command, a URL to visit, a test to run.]

## Troubleshooting

[Common failure modes and their fixes. Pull from tests and error handling code.]
```

**How-to 文档规则:**
- 标题以「How to」开头 ; 没有例外。这是读者的入口。
- 每个步骤都必须可操作。没有「考虑是否...」 ; 改为「运行 X」或「将 Y 添加到 Z」。
- 包括验证。读者永远不应该怀疑「它工作了吗?」
- 如果任务可能失败,则故障排除部分是必需的。

---

## 第 6 步:编写教程

教程是面向学习的。它们将新手从零带到一个工作示例。
这些是最难写好的,也是最有价值的。

**教程文档模板:**

```markdown
# [Tutorial title; describes what you'll build/learn]

[Opening paragraph: what you'll build, why it's useful, and what you'll understand
by the end. Keep it concrete: "You'll build a working X that does Y" not
"This tutorial covers X".]

## What you'll need

[Prerequisites: tools, versions, prior knowledge. Link to installation guides.]

## Step 1: [Set up the foundation]

[Start from a clean state. Show every command. Explain what each does on first
encounter; but briefly, not a lecture.]

```bash
[exact command]
```

[Brief explanation of what just happened.]

## Step 2: [Build the first working piece]

[Get to a working, visible result as fast as possible. The reader should see
something happen within the first 3 steps.]

...

## Step N: [Final step]

## What you built

[Recap: what the reader now has and what it can do. Link to reference docs
for deeper exploration. Suggest next steps.]
```

**教程规则:**
- **到第一个结果的时间 < 3 步。** 如果读者在第 3 步之前还没有看到某些东西在工作,则教程太慢。
- 每个步骤都必须产生可见的更改或输出。没有「现在配置 X」却不显示什么更改。
- 使用读者将输入的确切命令。没有「运行适当的命令」抽象。
- 错误路径:如果某个步骤经常失败,请内联显示错误和修复。
- 以「What you built」结束 ; 将教程连接回真实用例。

---

## 第 7 步:交叉文档链接与可发现性

在编写所有文档之后:

1. **在象限之间添加交叉链接。** 每个参考文档都应该链接到它的 how-to。
   每个 how-to 都应该链接到它的参考。教程应该链接到两者。

2. **更新入口点文件。** 在以下位置添加对新文档的引用:
   - README.md ; 添加到文档部分或目录
   - CLAUDE.md / AGENTS.md ; 如果相关则添加到项目结构
   - 任何现有的文档索引或侧边栏配置

3. **验证可发现性。** 每个新文档都必须在 README.md 的 2 次点击内可访问。如果正在使用文档框架,请添加到侧边栏/导航配置。

4. **检查断开的链接。** Grep 任何指向不存在的文件的 `](` 引用。

---

## 第 8 步:质量自我审查

在提交之前,根据这些标准审查每个文档:

**准确性门:**
- [ ] 每个代码示例在复制粘贴后都编译/运行/通过
- [ ] 每个 API 描述与实际代码签名匹配
- [ ] 每个显示的命令都产生所述的输出
- [ ] 没有对已重命名/已删除实体的过时引用

**完备性门:**
- [ ] 参考文档涵盖 100% 的公共表面
- [ ] How-to 涵盖用户将尝试的前 3 个任务
- [ ] 教程在 ≤3 步内达到工作结果
- [ ] 解释文档命名权衡,而不只是选择

**声音门:**
- [ ] 为一个聪明的、没见过代码的人编写
- [ ] 没有首次使用时不带内联注释的行话
- [ ] 主动语态、具体名词、短句
- [ ] 「You can now...」而不是「The system provides...」

在继续之前修复任何失败。

---

## 第 9 步:提交与输出

1. 按名称暂存新的文档文件(永远不要 `git add -A` 或 `git add .`)。

**提交前重新检查。** 生成的文档经常包含示例
凭据;扫描暂存的文档内容并在 HIGH 凭据上阻塞(已提交文档中的实时格式机密是泄漏)。每个 span 占位符
filter 通过明显的文档示例(例如 `AKIAIOSFODNN7EXAMPLE`);
实时格式机密在任何地方出现都会阻止,无论是否围栏:

```bash
REDACT_VIS=$(~/.claude/skills/gstack/bin/gstack-config get redact_repo_visibility 2>/dev/null)
[ -z "$REDACT_VIS" ] && REDACT_VIS=$(gh repo view --json visibility -q .visibility 2>/dev/null | tr 'A-Z' 'a-z')
git diff --cached --no-color | grep '^+' | sed 's/^+//' | \
  ~/.claude/skills/gstack/bin/gstack-redact --repo-visibility "${REDACT_VIS:-unknown}" --json
# exit 3 (HIGH) → unstage the offending doc, remove the secret, re-stage. Do NOT commit.
```

### 2. 创建提交:

```bash
git commit -m "$(cat <<'EOF'
docs: generate [scope] documentation (Diataxis)

[One-line summary of what was documented]

Quadrants: [list which quadrants were produced]

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### 3. 推送到当前分支:

```bash
git push
```

4. **如果存在 PR**,使用 `## Documentation Generated` 部分更新 PR 正文,列出每个新文件及其 Diataxis 象限和一行描述:

```
## Documentation Generated

| File | Quadrant | Description |
|------|----------|-------------|
| docs/tutorial-getting-started.md | Tutorial | Walk-through from install to first working example |
| docs/reference-widget-api.md | Reference | Complete widget API with types, defaults, examples |
| docs/explanation-bayesian-scheduler.md | Explanation | Why the scheduler uses Bayesian inference |
| docs/howto-custom-widgets.md | How-to | Creating and registering custom widgets |
```

5. 输出结构化摘要:

```
Documentation generated:
  Scope: [what was documented]
  Files: [N] new, [M] updated
  Coverage:
    Tutorials:    [count] ([list])
    How-tos:      [count] ([list])
    Reference:    [count] ([list])
    Explanation:  [count] ([list])
  Quality: [pass/fail on each gate]
```

---

## 重要规则

- **先研究再写。** 第 1 步不是可选的。阅读代码、阅读测试、阅读现有文档。研究不足会产生表面级文档。
- **准确性是不容妥协的。** 每个代码示例都必须工作。每个 API 描述都必须与实际代码匹配。如果你不确定某个细节,请再次阅读源代码 ; 不要凭空捏造。
- **Diataxis 象限服务于不同的读者。** 不要将教程内容混合到参考文档中,或将参考内容混合到 how-to 中。每个象限在特定模式下服务于特定读者。
- **教程中的第一个结果时间。** 如果读者在第 3 步之前看不到某些东西在工作,请重构教程。
- **交叉链接一切。** 孤立的文档就是不可发现的文档。
- **声音:友好、具体、面向用户。** 像向一个聪明的、没见过代码的人解释一样写。永远不要企业,永远不要学术。
- **完备性优于极简主义。** AI 使全面文档变得便宜。不要写「最小可行文档」 ; 写完整的文档。要不要煮海。