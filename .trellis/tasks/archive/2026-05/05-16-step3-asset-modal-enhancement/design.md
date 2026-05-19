# Step3 资产弹窗增强 — 技术设计

## 架构边界

- **后端**：`main.py` — 扩展 PATCH 接口 + 变体结构新增字段
- **前端**：`static/project.html` — 弹窗 HTML 结构 + JS 逻辑

---

## R1：每资产提示词编辑器

### 后端变更

**`AssetDescriptionUpdate` Pydantic 模型**（当前只有 `description`、`image_url`、`full_body_image_url`）：
- 新增字段：`prompt: Optional[str] = None`

**PATCH 接口**（行 3123-3146）：
- 在 `if body.description is not None` 之后，添加：
  ```python
  if body.prompt is not None:
      item["prompt"] = body.prompt
  ```
- `asset_type` 校验扩展支持 `character_headshot`（与其他变体接口保持一致）

### 前端变更

**HTML**（在 `assetModalNegativePrompt` textarea 附近新增）：
```html
<label>生成提示词 / Prompt</label>
<textarea id="assetModalPrompt" ...></textarea>
```

**`openAssetModal`**：读取 `asset.prompt` 填充 textarea

**`saveAssetModalDescription`** → 重命名为 `saveAssetModal`，同时保存 `description` + `prompt`

**`handleRegenAsset`**：调用时传入 `prompt: assetModalPrompt.value`，后端 regenerate 接口已接受 `prompt` 参数（需确认）

---

## R2：Art Direction 风格折叠预览

### 当前状态
- `assetModalStylePreview`（行 857）：monospace div，只显示 `style_prompt` 文本
- `assetModalApplyStyle` checkbox 在其上方

### 变更方案

将现有区域改为可折叠面板：

```html
<div id="artDirectionPanel">
  <div id="artDirectionHeader" onclick="toggleArtDirection()">
    <input type="checkbox" id="assetModalApplyStyle">
    <span>应用风格方向</span>
    <span id="artDirectionArrow">▶</span>
  </div>
  <div id="artDirectionBody" style="display:none">
    <div>正向提示词：<pre id="artDirPositive"></pre></div>
    <div>负向提示词：<pre id="artDirNegative"></pre></div>
  </div>
</div>
```

**`openAssetModal`** 中：
- 读取 `state.project.art_direction.style_prompt` → 填充 `artDirPositive`
- 读取 `state.project.art_direction.negative_prompt` → 填充 `artDirNegative`
- 无风格时隐藏整个 `artDirectionPanel`

**`state`** 新增：`modalArtDirectionOpen: false`

---

## R3：逆向生成提示词注入

### 后端变更

**`_make_variant`**（行 556）：新增 `is_uploaded_source` 参数：
```python
def _make_variant(url: str, is_uploaded_source: bool = False) -> dict:
    return {"id": ..., "url": url, "is_favorited": False,
            "created_at": time.time(), "is_uploaded_source": is_uploaded_source}
```

**PATCH 接口**（行 3140-3142）：上传 URL 时标记：
```python
if upload_url:
    _append_variant(item, upload_url, is_uploaded_source=True)
```

**`_append_variant`** 签名扩展：透传 `is_uploaded_source` 给 `_make_variant`

### 前端变更

**`openAssetModal`** 中检测：
```javascript
const hasUploadedSource = (asset.variants || []).some(v => v.is_uploaded_source);
if (hasUploadedSource) {
    const prefix = 'STRICTLY MAINTAIN the SAME character appearance, face, hairstyle, skin tone, and clothing as the reference image. ';
    const currentPrompt = asset.prompt || '';
    if (!currentPrompt || currentPrompt === prefix) {
        assetModalPrompt.value = prefix;
    }
}
```

---

## R4：角色弹窗双面板布局

### 当前状态
- `assetModalCharTabs`（行 871-872）：Full Body / Headshot 两个 tab 按钮已存在
- `state.modalCharType`：已有 `character` / `character_headshot` 切换
- 但左侧图片区和变体条没有随 tab 切换

### 变更方案

**`syncAssetModalCharTabs`** 函数（已存在，行 975 附近）扩展：
- 切换 tab 时调用 `refreshAssetModalImage()` + `renderVariantStrip()`
- 根据 `state.modalCharType` 决定显示 `full_body_image_url` 还是 `headshot_image_url`

**`openAssetModal`** 中角色资产：
- 初始化时根据 `state.modalCharType` 选择正确的图片 URL

**`handleRegenAsset`** 中：
- `effectiveType` 已正确使用 `state.modalCharType`（行 2157），无需改动

**`renderVariantStrip`** 中：
- 角色资产的 headshot tab 需要独立的变体列表（当前所有变体共用一个 strip）
- 简化方案：headshot 变体与 full_body 变体共用同一 variants 数组，通过 `asset_type` 字段区分（需后端支持）
- **本轮简化**：headshot tab 显示同一 variants 数组，生成时 effectiveType=character_headshot 即可区分后端行为

---

## 数据流

```
openAssetModal(assetType, assetId)
  ├── 读取 asset.prompt → 填充 assetModalPrompt
  ├── 检测 is_uploaded_source → 注入 STRICTLY MAINTAIN 前缀
  ├── 读取 art_direction → 填充折叠面板
  └── 角色资产：根据 modalCharType 选择图片 URL

saveAssetModal()
  └── PATCH { description, prompt }

handleRegenAsset(assetType, assetId, { prompt, applyStyle, negativePrompt, batchSize })
  └── POST regenerate_asset { ..., prompt }
```

---

## 兼容性

- 旧资产无 `prompt` 字段 → 前端读取时用 `asset.prompt || ''`
- 旧变体无 `is_uploaded_source` → 前端读取时用 `v.is_uploaded_source || false`
- `_migrate_asset_variants()` 无需修改（字段缺失时默认 false 即可）
