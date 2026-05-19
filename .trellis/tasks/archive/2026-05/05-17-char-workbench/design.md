# char-workbench Design

## 策略

角色弹窗从当前左右分栏改为 3 列工作台。由于现有 `openAssetModal` 同时服务 character/scene/prop，改造方案为：

- **角色**打开时：隐藏现有左右分栏，显示新的 3 列工作台 HTML
- **场景/道具**打开时：保持现有左右分栏（后续 scene-prop-modal 任务改造）

这样避免一次性重写整个弹窗，降低风险。

## HTML 结构

在 `#assetModal > .asset-modal` 内新增一个 `#charWorkbench` 容器，与现有 `.asset-modal-left` + `.asset-modal-right` 并列：

```html
<div id="charWorkbench" class="char-workbench hidden">
  <div class="cw-header">
    <h2 id="cwTitle"></h2>
    <button id="cwClose">×</button>
  </div>
  <div class="cw-columns">
    <div class="cw-col" id="cwColFullBody">
      <div class="cw-col-header">Full Body</div>
      <div class="cw-col-img" id="cwImgFullBody"></div>
      <div class="cw-col-variants variant-strip" id="cwVarFullBody"></div>
      <textarea class="cw-col-prompt" id="cwPromptFullBody"></textarea>
      <button class="primary-btn cw-gen-btn" id="cwGenFullBody">↺ Generate</button>
    </div>
    <div class="cw-col" id="cwColThreeView">
      <div class="cw-col-header">Three View</div>
      <div class="cw-col-img" id="cwImgThreeView"></div>
      <div class="cw-col-variants variant-strip" id="cwVarThreeView"></div>
      <textarea class="cw-col-prompt" id="cwPromptThreeView"></textarea>
      <button class="primary-btn cw-gen-btn" id="cwGenThreeView">↺ Generate</button>
    </div>
    <div class="cw-col" id="cwColHeadshot">
      <div class="cw-col-header">Headshot</div>
      <div class="cw-col-img" id="cwImgHeadshot"></div>
      <div class="cw-col-variants variant-strip" id="cwVarHeadshot"></div>
      <textarea class="cw-col-prompt" id="cwPromptHeadshot"></textarea>
      <button class="primary-btn cw-gen-btn" id="cwGenHeadshot">↺ Generate</button>
    </div>
  </div>
  <div class="cw-footer">
    <div class="cw-footer-left">
      <label><input type="checkbox" id="cwApplyStyle" checked> Apply Style</label>
      <button id="cwArtDirToggle">▶ Art Direction</button>
      <div id="cwArtDirBody" class="hidden">...</div>
    </div>
    <div class="cw-footer-right">
      <textarea id="cwNegativePrompt" placeholder="Negative prompt..."></textarea>
    </div>
  </div>
</div>
```

## CSS

```css
.char-workbench {
  display: flex; flex-direction: column; width: 100%; height: 100%;
}
.char-workbench.hidden { display: none; }
.cw-header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 16px 24px; border-bottom: 1px solid var(--border);
}
.cw-columns {
  flex: 1; display: flex; overflow: hidden;
}
.cw-col {
  flex: 1; display: flex; flex-direction: column; padding: 16px;
  border-right: 1px solid var(--border); overflow-y: auto;
}
.cw-col:last-child { border-right: none; }
.cw-col-header { font-size: 13px; font-weight: 700; margin-bottom: 12px; }
.cw-col-img {
  flex: 1; display: flex; align-items: center; justify-content: center;
  min-height: 200px; background: var(--stage-bg, #f8fafc); border-radius: 12px;
  overflow: hidden; margin-bottom: 12px;
}
.cw-col-img img { max-width: 100%; max-height: 100%; object-fit: contain; }
.cw-col-variants { margin-bottom: 12px; }
.cw-col-prompt {
  min-height: 60px; font-size: 12px; font-family: monospace;
  border: 1px solid var(--border); border-radius: 8px; padding: 8px;
  resize: vertical; margin-bottom: 8px;
}
.cw-gen-btn { width: 100%; height: 36px; font-size: 13px; }
.cw-footer {
  display: flex; align-items: flex-start; gap: 16px;
  padding: 12px 24px; border-top: 1px solid var(--border);
}
```

## JS 逻辑

### `openAssetModal` 修改

```javascript
// 在 openAssetModal 中：
if (assetType === 'character') {
    // 隐藏旧布局，显示工作台
    document.querySelector('.asset-modal-left').classList.add('hidden');
    document.querySelector('.asset-modal-right').classList.add('hidden');
    document.getElementById('charWorkbench').classList.remove('hidden');
    openCharWorkbench(asset);
} else {
    // 场景/道具保持旧布局
    document.querySelector('.asset-modal-left').classList.remove('hidden');
    document.querySelector('.asset-modal-right').classList.remove('hidden');
    document.getElementById('charWorkbench').classList.add('hidden');
    // ... 现有逻辑
}
```

### `openCharWorkbench(asset)` 新函数

- 填充 3 列的图片、变体条、prompt
- prompt 初始化逻辑同 luna-studio 的 `getInitialPrompt`
- 每列变体条复用 `renderVariantStrip` 的逻辑，但目标元素不同

### 每列变体条渲染

新增 `renderColumnVariantStrip(stripEl, variants, selectedId)` 通用函数，接受目标元素参数。
现有 `renderVariantStrip` 改为调用此通用函数。

### 每列生成按钮

点击 → 调用 `regenerateAsset(assetType, assetId, prompt, applyStyle, negativePrompt, batchSize)`
- Full Body: `asset_type = "character"`
- Three View: `asset_type = "character_three_view"`
- Headshot: `asset_type = "character_headshot"`

### 每列变体条事件委托

每个 `.cw-col-variants` 添加事件委托，逻辑同现有 `#variantStrip` 的 click handler，但 assetType 根据列确定。

## 弹窗尺寸

角色工作台模式下 `.asset-modal` 改为 `max-width: 1280px`（原 1040px）。

## 数据流

1. 打开 → 读 character 数据 → 填充 3 列
2. 生成 → 调用 API → 轮询/回调 → 刷新对应列
3. 变体操作 → 调用 select/delete/favorite API → 刷新对应列
4. 关闭 → 隐藏弹窗

## 不做的事

- Motion 按钮（child 4）
- 描述编辑（角色工作台不需要，描述在 Step3 卡片上编辑）
- 上传功能（保持现有上传逻辑，后续迭代）
