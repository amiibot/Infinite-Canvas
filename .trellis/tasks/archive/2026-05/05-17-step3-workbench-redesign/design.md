# Design：Step3 资产弹窗工作台改造

## 架构边界

单文件 `static/project.html`，纯 HTML/CSS/JS，无构建工具。
所有改动限于该文件内的 CSS 块、HTML 弹窗区域、JS 函数。

## 布局变更

### 弹窗容器
```
max-width: 900px → 1040px
```

### 左侧面板（.asset-modal-left）
```
width: 55% → 65%
flex-direction: column   ← 新增，使图片和变体条垂直排列
```
内部结构：
```
.asset-modal-left
  #assetModalImg          (flex:1, overflow:hidden)  ← 图片填满
  #variantStrip           (flex-shrink:0)             ← 变体条固定底部
```

### 右侧面板（.asset-modal-right）
```
width: 45% → 35%
```
内部结构（从上到下）：
```
.asset-modal-header       ← 不变：badge + title + close
.asset-modal-body
  #assetModalCharTabs     ← 移到最顶部（原在底部）
  #assetModalPromptSection← 提示词（原在第3位，现第2位）
  #assetModalDescSection  ← 描述折叠面板（新增折叠逻辑）
  #assetModalStyleSection ← 风格设置（保持折叠逻辑）
  #assetModalAdvancedSection ← 高级/负向（保持折叠逻辑）
.asset-modal-footer       ← Regenerate(强化) + Done
```

## 描述折叠面板设计

### HTML 结构
```html
<div id="assetModalDescSection">
  <div id="assetModalDescToggle" role="button" tabindex="0">
    <span class="asset-modal-desc-label">描述 / Description</span>
    <span id="assetModalDescArrow">▶</span>
  </div>
  <div id="assetModalDescBody" style="display:none">
    <!-- 现有的 view/edit 切换逻辑不变 -->
    <div id="assetModalDescView" ...></div>
    <div id="assetModalDescEdit" ...></div>
  </div>
</div>
```

### JS 新增函数
```js
function toggleAssetModalDesc() {
    state.modalDescOpen = !state.modalDescOpen;
    document.getElementById('assetModalDescBody').style.display =
        state.modalDescOpen ? '' : 'none';
    document.getElementById('assetModalDescArrow').textContent =
        state.modalDescOpen ? '▼' : '▶';
}
```

`openAssetModal` 中初始化：`state.modalDescOpen = false`

## 变体条迁移

### HTML 变更
将 `<div class="variant-strip" id="variantStrip"></div>` 从
`.asset-modal-left` 和 `.asset-modal-right` 之间，移入 `.asset-modal-left` 内部末尾。

### CSS 变更
`.asset-modal-left` 新增：
```css
flex-direction: column;
align-items: stretch;
```
`#assetModalImg`（图片容器）新增：
```css
flex: 1;
overflow: hidden;
```
`.variant-strip` 在弹窗内：
```css
flex-shrink: 0;
border-top: 1px solid var(--border);
```

## 主按钮强化

```css
/* Regenerate 按钮 */
height: 38px → 42px
font-weight: 800（已有，确认保留）
font-size: 13px → 14px
```

## 兼容性

- `renderVariantStrip` 直接操作 `#variantStrip` innerHTML，迁移后无需改动
- `refreshAssetModalImage` 操作 `#assetModalImg` innerHTML，迁移后无需改动
- `setAssetModalEditing` 操作 `#assetModalDescView` / `#assetModalDescEdit`，
  这两个元素仍在 DOM 中，无需改动
- `syncAssetModalCharTabs` 操作 `.asset-modal-char-tab`，位置变化不影响逻辑

## 回滚

所有改动在单文件内，git checkout static/project.html 即可完整回滚。
