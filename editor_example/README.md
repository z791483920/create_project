# 文本高亮实现方案对比

本目录包含 4 个 Demo，展示了在 `contenteditable` 编辑器中实现文本高亮的不同方案。

## 快速预览

```bash
# 方案1: CSS Highlight API
open highlight-demo.html

# 方案2: DOM 包裹
open highlight-demo-dom.html

# 方案3: 覆盖层（推荐）
open highlight-demo-overlay.html
```

---

## 方案对比

| 特性 | CSS Highlight API | DOM 包裹 | 覆盖层 |
|------|-------------------|----------|--------|
| 圆角边框 | ❌ | ✅ | ✅ |
| 边框样式 | ❌ | ✅ | ✅ |
| 点击事件 | ❌ | ✅ | ✅ |
| 编辑体验 | ✅ 最佳 | ⚠️ 可能异常 | ✅ 良好 |
| 性能 | ✅ 最佳 | ✅ 良好 | ⚠️ 需要实时更新 |
| 浏览器支持 | Chrome 105+ | ✅ 全部 | ✅ 全部 |

---

## 方案1: CSS Highlight API

**文件**: `highlight-demo.html`

使用浏览器原生的 CSS Custom Highlight API，不修改 DOM 结构。

### 核心代码

```javascript
// 1. 创建 Highlight 对象
const highlight = new Highlight()
CSS.highlights.set('my-highlight', highlight)

// 2. 创建 Range 并添加到 Highlight
const range = new Range()
range.setStart(textNode, startOffset)
range.setEnd(textNode, endOffset)
highlight.add(range)
```

```css
/* 3. 用伪元素定义样式 */
::highlight(my-highlight) {
  color: #7B61FF;
  background: rgba(123, 97, 255, 0.15);
}
```

### 支持的 CSS 属性

| 支持 ✅ | 不支持 ❌ |
|---------|----------|
| `color` | `border` |
| `background-color` | `border-radius` |
| `text-decoration` | `padding` |
| `text-shadow` | `margin` |
| `-webkit-text-stroke` | `box-shadow` |

### 适用场景

- 纯展示型高亮
- 对编辑体验要求高
- 不需要交互功能

---

## 方案2: DOM 包裹

**文件**: `highlight-demo-dom.html`

传统方案，用 `<span>` 包裹高亮文本。

### 核心代码

```javascript
const html = text.replace(
  /\{\{(\w+)\}\}/g,
  '<span class="var-highlight">{{$1}}</span>'
)
editor.innerHTML = html
```

```css
.var-highlight {
  color: #7B61FF;
  background: rgba(123, 97, 255, 0.15);
  border: 1px solid #7B61FF;
  border-radius: 4px;
  padding: 1px 4px;
}
```

### 缺点

- 光标进出 span 时行为异常
- 删除/输入时 span 结构容易被破坏
- 需要额外逻辑维护 DOM 结构

### 适用场景

- 只读展示
- 不需要编辑功能
- 需要完整 CSS 样式支持

---

## 方案3: 覆盖层（推荐）

**文件**: `highlight-demo-overlay.html`

在编辑器上方叠加绝对定位的元素，实现圆角边框和点击交互。

### 核心代码

```javascript
// 1. 获取文本位置
const range = createRangeFromTextOffset(editor, start, end)
const rects = range.getClientRects()

// 2. 创建覆盖层
for (const rect of rects) {
  const overlay = document.createElement('div')
  overlay.className = 'highlight-overlay'
  overlay.style.left = rect.left + 'px'
  overlay.style.top = rect.top + 'px'
  overlay.style.width = rect.width + 'px'
  overlay.style.height = rect.height + 'px'

  // 3. 添加点击事件
  overlay.addEventListener('click', () => {
    showPopup(variableName, rect)
  })

  container.appendChild(overlay)
}
```

```css
.highlight-overlay {
  position: absolute;
  pointer-events: auto;  /* 可点击 */
  border: 1px solid #7B61FF;
  border-radius: 4px;    /* 圆角生效！ */
  background: rgba(123, 97, 255, 0.1);
  cursor: pointer;
}

.highlight-overlay:hover {
  background: rgba(123, 97, 255, 0.25);
  transform: scale(1.02);
}
```

### 特性

- ✅ 真正的圆角边框
- ✅ 完整 CSS 样式支持
- ✅ 支持点击事件和 hover 效果
- ✅ `pointer-events: none` 时不影响编辑
- ⚠️ 需要监听 input/scroll/resize 更新位置

### 适用场景

- 需要圆角边框
- 需要点击交互（弹出菜单、跳转等）
- 对样式有较高要求

---

## 技术要点

### 1. 根据文本偏移量创建 Range

```javascript
function createRangeFromTextOffset(root, start, end) {
  const walker = document.createTreeWalker(root, NodeFilter.SHOW_TEXT)
  let currentOffset = 0
  let startNode, endNode, startOffset, endOffset

  while (walker.nextNode()) {
    const node = walker.currentNode
    const len = node.textContent.length

    if (!startNode && currentOffset + len > start) {
      startNode = node
      startOffset = start - currentOffset
    }
    if (currentOffset + len >= end) {
      endNode = node
      endOffset = end - currentOffset
      break
    }
    currentOffset += len
  }

  const range = new Range()
  range.setStart(startNode, startOffset)
  range.setEnd(endNode, endOffset)
  return range
}
```

### 2. 获取文本位置

```javascript
// 单个矩形（不跨行）
const rect = range.getBoundingClientRect()

// 多个矩形（可能跨行）
const rects = range.getClientRects()
```

### 3. 点击穿透控制

```css
/* 不可点击，穿透到下层 */
.overlay { pointer-events: none; }

/* 可点击 */
.overlay { pointer-events: auto; }
```

---

## 浏览器兼容性

| API | Chrome | Firefox | Safari | Edge |
|-----|--------|---------|--------|------|
| CSS Highlight API | 105+ | ❌ | 17.2+ | 105+ |
| Range.getClientRects() | ✅ | ✅ | ✅ | ✅ |
| pointer-events | ✅ | ✅ | ✅ | ✅ |

---

## 相关资源

- [CSS Custom Highlight API 规范](https://drafts.csswg.org/css-highlight-api/)
- [MDN: CSS Custom Highlight API](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Custom_Highlight_API)
- [MDN: Range](https://developer.mozilla.org/en-US/docs/Web/API/Range)
