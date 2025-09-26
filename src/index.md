好的，我们来深入代码，详细分析和讲解 prosemirror-schema-basic 这个模块。

prosemirror-schema-basic 是 ProseMirror 生态系统的基石之一。它的作用是提供一个**基础的、开箱即用的文档蓝图 (Schema)**，它包含了构建一个基本富文本编辑器所需的大部分常见元素，如段落、标题、加粗、斜体等。这个 Schema 的设计大致遵循了 [CommonMark](http://commonmark.org/) 规范。

你可以直接使用这个 Schema，也可以扩展它，或者只从中挑选你需要的节点和标记规范来构建你自己的 Schema。

---

### 核心概念回顾

在深入代码之前，我们先快速回顾一下核心概念：

- **`Schema`**: 定义了文档中可以出现哪些节点和标记，以及它们之间的嵌套规则。它是整个编辑器的结构性约束。
- **`NodeSpec`**: 一个节点的规范。它定义了节点的名称、内容规则、属性、如何从 DOM 解析 (`parseDOM`) 以及如何渲染回 DOM (`toDOM`)。
- **`MarkSpec`**: 一个标记的规范。它定义了标记的名称、属性、以及 `parseDOM` 和 `toDOM` 规则。标记是应用在内联内容上的一种样式（如加粗、链接）。

---

### 代码结构与详细讲解

文件 `schema-basic.ts` 的结构非常清晰：

1.  定义 `nodes` 对象，包含所有节点规范。
2.  定义 `marks` 对象，包含所有标记规范。
3.  用 `nodes` 和 `marks` 创建并导出最终的 `schema` 实例。

#### 第一部分：`nodes` - 文档结构块的定义

这是导出的 `nodes` 对象，包含了所有块级和内联节点的规范。

##### 1. 顶层节点 `doc`

```typescript
// ...existing code...
export const nodes = {
  /// NodeSpec The top level document node.
  doc: {
    content: "block+"
  } as NodeSpec,
// ...existing code...
```

- `doc` 是每个 ProseMirror 文档的根节点。
- `content: "block+"`: 这是内容表达式，意思是 `doc` 节点的内容必须是**一个或多个**（`+`）属于 `"block"` 组的节点。

##### 2. 块级节点 (Block Nodes)

这些是构成文档主体结构的节点，它们都属于 `"block"` 组。

- **`paragraph`**:

  ```typescript
  // ...existing code...
  paragraph: {
    content: "inline*",
    group: "block",
    parseDOM: [{tag: "p"}],
    toDOM() { return pDOM }
  } as NodeSpec,
  // ...existing code...
  ```

  - `content: "inline*"`: 段落可以包含**零个或多个**（`*`）属于 `"inline"` 组的内容（即文本、图片、硬换行等）。
  - `group: "block"`: 将段落归类到 `"block"` 组，使其可以被 `doc` 节点包含。
  - `parseDOM`: 定义了当编辑器解析 HTML 时，遇到 `<p>` 标签就应该创建一个 `paragraph` 节点。
  - `toDOM()`: 定义了将 `paragraph` 节点渲染回 DOM 时，应该创建一个 `<p>` 标签。`pDOM` 是一个 `["p", 0]` 的简写，`0` 代表一个“内容洞”，表示子节点应该被渲染到这个位置。

- **`heading`**:

  ```typescript
  // ...existing code...
  heading: {
    attrs: {level: {default: 1}},
    content: "inline*",
    group: "block",
    defining: true,
    parseDOM: [{tag: "h1", attrs: {level: 1}},
               {tag: "h2", attrs: {level: 2}}, ...],
    toDOM(node) { return ["h" + node.attrs.level, 0] }
  } as NodeSpec,
  // ...existing code...
  ```

  - `attrs: {level: {default: 1}}`: 这是属性的绝佳示例。`heading` 节点有一个 `level` 属性，默认为 1。
  - `parseDOM`: 定义了一系列规则，将 `<h1>` 到 `<h6>` 标签分别解析成带有不同 `level` 属性的 `heading` 节点。
  - `toDOM(node)`: 渲染时，根据节点的 `level` 属性动态地创建 `<h1>` 到 `<h6>` 标签。
  - `defining: true`: 表示这是一个“定义边界”的节点。这意味着光标无法跨越标题的边界，并且其内容被视为一个整体，这对于确保标题结构的完整性很重要。

- **`code_block`**:
  ```typescript
  // ...existing code...
  code_block: {
    content: "text*",
    marks: "",
    group: "block",
    code: true,
    // ...
  } as NodeSpec,
  // ...existing code...
  ```
  - `content: "text*"`: 内容只能是文本节点。
  - `marks: ""`: 一个空字符串，表示**不允许任何标记**（如加粗、斜体）应用在代码块内的文本上。
  - `code: true`: 一个元标志，告诉编辑器这是一个代码块，可以触发特殊的键盘绑定（如 `Tab` 键的行为）。

##### 3. 内联节点 (Inline Nodes)

这些是穿插在文本中的内容，它们都属于 `"inline"` 组。

- **`text`**:

  - 这是最基础的节点，代表纯文本。它没有 `toDOM` 或 `parseDOM`，因为 ProseMirror 对其有特殊的内部处理。

- **`image`**:

  - 一个内联的叶子节点（没有内容）。
  - `attrs`: 定义了 `src`, `alt`, `title` 属性。
  - `draggable: true`: 使其在编辑器中可以被拖拽。

- **`hard_break`**:
  - 代表一个 `<br>` 标签，用于强制换行。
  - `selectable: false`: 使其不能被单独选中。

#### 第二部分：`marks` - 内联样式的定义

- **`link`**:

  ```typescript
  // ...existing code...
  link: {
    attrs: {
      href: {},
      title: {default: null}
    },
    inclusive: false,
    parseDOM: [{tag: "a[href]", getAttrs(dom: HTMLElement) { ... }}],
    toDOM(node) { ... }
  } as MarkSpec,
  // ...existing code...
  ```

  - `attrs`: 定义了 `href` 和 `title` 属性。
  - `inclusive: false`: 一个重要的属性。它定义了当光标位于标记的末尾时，新输入的文本**是否**应该自动应用该标记。对于链接来说，我们通常不希望新输入的文本也成为链接的一部分，所以设为 `false`。

- **`em` (斜体) 和 `strong` (加粗)**:
  ```typescript
  // ...existing code...
  strong: {
    parseDOM: [
      {tag: "strong"},
      {tag: "b", getAttrs: (node: HTMLElement) => node.style.fontWeight != "normal" && null},
      {style: "font-weight", getAttrs: (value: string) => /^(bold(er)?|[5-9]\d{2,})$/.test(value) && null},
    ],
    toDOM() { return strongDOM }
  } as MarkSpec,
  // ...existing code...
  ```
  - 这两个标记展示了 `parseDOM` 强大的灵活性。它不仅可以匹配标签（如 `<strong>` 和 `<b>`），还可以匹配 CSS 样式（如 `font-weight: bold`）。
  - `getAttrs` 在这里被用作一个**验证器**。如果它返回 `null`，表示这个规则匹配成功；如果返回 `false`，则匹配失败。例如，`<b>` 标签只有在 `font-weight` 不是 `normal` 时才被解析为 `strong`，这是为了处理从 Google Docs 粘贴过来的奇怪 HTML。

#### 第三部分：`schema` - 最终的组装

```typescript
// ...existing code...
export const schema = new Schema({ nodes, marks })
```

最后一步非常简单：将前面定义好的 `nodes` 和 `marks` 对象传入 `Schema` 构造函数，创建一个完整的、可供 `EditorState` 使用的 `schema` 实例，并将其导出。

### 总结

prosemirror-schema-basic 是学习如何构建 ProseMirror Schema 的绝佳范例。它清晰地展示了：

- 如何使用**内容表达式** (`content`) 和**组** (`group`) 来定义文档的层次结构。
- 如何为节点和标记添加**属性** (`attrs`)。
- 如何编写强大而灵活的 **`parseDOM`** 规则来处理各种 HTML 输入。
- 如何使用 **`toDOM`** 将内部文档模型可靠地序列化回 HTML。
- 如何使用 `defining`, `inclusive`, `code` 等元属性来微调编辑行为。

通过理解这个文件的每一部分，你就掌握了创建自定义 ProseMirror 编辑器所需的最核心的知识。
