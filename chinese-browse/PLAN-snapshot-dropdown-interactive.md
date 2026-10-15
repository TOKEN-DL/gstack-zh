# 方案:检测 Snapshot 中的下拉/自动补全可交互元素

## 问题

`snapshot -i` 在现代 Web 应用上会漏掉 dropdown/autocomplete(下拉/自动补全)项。这些元素:
1. 经常是带 click handler 但没有语义 ARIA role 的 `<div>`/`<li>`
2. 位于动态创建的 portal/popover(浮动容器)中
3. 不会出现在 Playwright 的 accessibility tree(`ariaSnapshot()`)中

`-C` 标志(cursor-interactive scan,光标可交互扫描)本来是为这种情况设计的,但:
- 需要额外标志 —— 用 `-i` 的 agent 不会自动得到它
- 跳过已经拥有 ARIA role 的元素(即便 ARIA tree 也漏掉了它们)
- 不会优先扫描 dropdown 项所在的 popover/portal 容器

## 根因

Playwright 的 `ariaSnapshot()` 由浏览器的 accessibility tree 构建。如果发生以下情况,动态渲染的 popover(React portals、Radix Popover 等)可能不在 accessibility tree 中:
- 组件没有设置 ARIA roles
- portal 在作用域内 `body` locator 子树时机的外侧渲染
- DOM 变更之后浏览器还没来得及更新 accessibility tree

## 改动

### 1. 使用 `-i` 标志时自动启用 cursor-interactive scan

**文件:** `browse/src/snapshot.ts`

当传入 `-i`(interactive)时,自动包含 cursor-interactive scan。这意味着 agent 在请求可交互元素时,总能看见带 click handler 的非 ARIA 元素。

`-C` 标志保留为独立选项,用于非 interactive snapshot。

```
if (opts.interactive) {
  opts.cursorInteractive = true;
}
```

### 2. 新增 popover/portal 优先扫描

**文件:** `browse/src/snapshot.ts`(在 cursor-interactive evaluate 块内部)

在常规的 cursor:pointer 扫描之前,专门扫描可见的浮动容器(popovers、dropdowns、menus),并将其所有直接子元素视作可交互:

浮动容器的启发式检测:
- `position: fixed` 或 `position: absolute` 且 `z-index >= 10`
- 带有 `role="listbox"`、`role="menu"`、`role="dialog"`、`role="tooltip"`、`[data-radix-popper-content-wrapper]`、`[data-floating-ui-portal]` 等
- DOM 中是新出现的(不在初始页面加载里)
- 可见(`offsetParent !== null` 或 `position: fixed`)

对每个浮动容器,纳入满足以下条件的子元素:
- 有文本内容
- 可见
- 带有 cursor:pointer 或 onclick 或 role="option" 或 role="menuitem"
- 标记 reason 为 `popover-child` 以便识别

### 3. 在 cursor-interactive scan 中移除 `hasRole` 跳过逻辑

**文件:** `browse/src/snapshot.ts`

当前逻辑:`if (hasRole) continue;` —— 会跳过任何带有 ARIA role 的元素,假设 ARIA tree 已经捕获了它。

问题:如果 ARIA tree 也漏掉了该元素(时序、portal、DOM 结构错误),它会从两个系统同时漏掉。

修复:仅当元素的 role 出现在 `INTERACTIVE_ROLES` 中、并且它确实已被主 refMap 捕获时才跳过,否则将其纳入。

由于 `page.evaluate()` 内部很难查询 refMap,更简单的修复是:对位于已检测浮动容器内的元素,完全移除 `hasRole` 跳过逻辑。对浮动容器外的元素,继续保留当前的 `hasRole` 跳过逻辑(以避免在常规页面内容中产生重复项)。

### 4. 新增 dropdown 测试 fixture 与测试

**文件:** `browse/test/fixtures/dropdown.html`

HTML 页面包含:
- 聚焦/输入时显示 dropdown 的 combobox input
- 以 `<div>`(无 ARIA roles)+ click handler 形式存在的 dropdown 项
- 以 `<li>`(带 `role="option"`)形式存在的 dropdown 项
- 一个 React-portal 风格容器(`position: fixed`,高 z-index)

**文件:** `browse/test/snapshot.test.ts`

新增测试用例:
- 在 dropdown 页面上 `snapshot -i` 通过 cursor scan 找到 dropdown 项
- 在 dropdown 页面上 `snapshot -i` 包含 popover-child 元素
- 来自 dropdown scan 的 `@c` ref 可点击
- 即便 ARIA tree 漏掉,位于浮动容器内、带有 ARIA role 的元素仍被捕获

## 发布风险

**低。** `-C` 扫描是累加性的 —— 它只会新增 `@c` ref,不会移除 `@e` ref。在 `-i` 下自动启用 `-C` 会让输出变大,但 agent 已经能处理混合 ref 类型。

**一个隐患:** `-C` 扫描会查询全部元素(`document.querySelectorAll('*')`),在内容很重的页面上可能慢。针对 popover 专项扫描,我们把范围限制在已检测浮动容器内的元素,这就很快(小子树)。

## 测试

```bash
cd /data/gstack/browse && bun test snapshot
```

## 文件变更

1. `browse/src/snapshot.ts` —— 配合 `-i` 自动启用 `-C`,popover 扫描,在浮动容器内移除 hasRole 跳过
2. `browse/test/fixtures/dropdown.html` —— 新测试 fixture
3. `browse/test/snapshot.test.ts` —— 新 dropdown/popover 测试用例