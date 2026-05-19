# UI 可访问性与质量规范

## 概述

本项目所有 `static/*.html` 页面需遵循的 a11y 和 UI 质量基线。
源自 2026-05-20 UI 审查修复（ui-a11y-fix, ui-a11y-fix-2）。

---

## ARIA 与语义化

### Convention: 图标按钮必须有 aria-label

**What**: 无可见文本的 `<button>` 必须有 `aria-label`。

**Example**:
```html
<!-- Good -->
<button aria-label="删除" onclick="del()">×</button>

<!-- Bad -->
<button onclick="del()">×</button>
```

### Convention: select 元素必须有可访问名称

**What**: `<select>` 需要 `aria-label` 或被 `<label>` 包裹。

**Example**:
```html
<select aria-label="选择模型">...</select>
<!-- 或 -->
<label>模型 <select>...</select></label>
```

### Convention: 不使用 div onclick

**What**: 可点击元素用 `<button>` 而非 `<div onclick>`。

**Why**: button 自带 focus、Enter/Space 触发、屏幕阅读器语义。

---

## 焦点与键盘

### Convention: outline:none 必须配合 :focus-visible

**What**: 移除默认 outline 时，必须提供 `:focus-visible` 替代样式。

**Example**:
```css
button { outline: none; }
button:focus-visible { box-shadow: 0 0 0 2px #3b82f6; }
```

### Convention: 自定义交互组件需键盘支持

**What**: 自定义选择/拖拽组件需响应 Enter/Space。

**Example** (history-bulk-manager.js):
```javascript
masonry.addEventListener('keydown', e => {
    if (e.key !== 'Enter' && e.key !== ' ') return;
    const card = e.target.closest('.masonry-item');
    if (card) { e.preventDefault(); toggleCard(card); }
});
```

---

## 动效安全

### Convention: prefers-reduced-motion 全局规则

**What**: theme.css 必须包含 reduced-motion 媒体查询。

**Example**:
```css
@media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
        animation-duration: 0.01ms !important;
        transition-duration: 0.01ms !important;
    }
}
```

### Convention: 禁止 transition: all

**What**: 必须列出具体属性，不用 `all`。

**Why**: `all` 会触发不必要的属性动画（如 height/width），影响性能和 reduced-motion 用户。

**Example**:
```css
/* Good */
transition: opacity .2s, transform .2s;

/* Bad */
transition: all .2s;
```

---

## 性能与 CLS

### Convention: 静态图片需 width/height

**What**: 已知尺寸的 `<img>`（如品牌 logo）必须声明 `width`/`height`。

**Why**: 防止 CLS（Cumulative Layout Shift）。

**Example**:
```html
<img src="/static/modelscope.gif" width="88" height="16" alt="ModelScope">
```

> 动态图片（用户上传、AI 生成）由 CSS 容器约束尺寸，不需要 HTML 属性。

### Convention: 列表/网格图片用 loading="lazy"

**What**: JS 动态渲染的图片列表（历史记录、资产 chip）加 `loading="lazy"`。

**Example**:
```javascript
`<img src="${url}" alt="output" loading="lazy">`
```

### Convention: Modal 加 overscroll-behavior:contain

**What**: 全屏/浮层 modal 的 CSS 需包含此属性。

**Why**: 防止滚动穿透到底层页面。

### Convention: 触摸画布加 touch-action:manipulation

**What**: 画布/拖拽区域需声明 `touch-action: manipulation`。

**Why**: 消除 300ms 点击延迟，防止浏览器默认缩放手势干扰。

---

## 文本规范

### Convention: UI 省略号用 Unicode `…`

**What**: 用户可见的省略号必须用 `…`（U+2026），不用 `...`。

**Why**: 排版正确，屏幕阅读器读作"省略号"而非"点点点"。

---

## 布局抖动

### Convention: 拖拽框选时缓存 getBoundingClientRect

**What**: 框选逻辑中，卡片 rect 在 mousedown 时一次性缓存，mousemove 中使用缓存值。

**Why**: 每帧对 N 个卡片调用 getBoundingClientRect 会强制回流。

**Example**:
```javascript
// mousedown — 缓存
drag = { cardRects: cards.map(c => ({ card: c, rect: c.getBoundingClientRect() })) };

// mousemove — 使用缓存
drag.cardRects.forEach(({ card, rect }) => { /* hit test */ });
```

---

## Gotcha: 全局替换 `...` 会破坏 JS spread

> **Warning**: `sed 's/\.\.\./…/g'` 会把 `...payload` 变成 `…payload`。
>
> 替换 UI 省略号后必须恢复 spread 运算符：
> ```bash
> sed -i 's/…\([a-zA-Z_\[\(({]\)/\.\.\.\1/g' file.html
> ```
> 或使用更精确的逐行替换而非全局 sed。

---

## 文档结构

### Convention: color-scheme 声明

**What**: theme.css 需在 `:root` / dark 变体中声明 `color-scheme`。

**Why**: 让浏览器原生控件（scrollbar、form）匹配主题。

```css
:root { color-scheme: light; }
html.studio-theme-dark { color-scheme: dark; }
```
