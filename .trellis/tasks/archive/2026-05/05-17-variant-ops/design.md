# variant-ops Design

## 现状分析

### 后端已有
- `_make_variant` / `_append_variant` / `_migrate_asset_variants` — 完整变体基础设施
- `_get_variant_fields(asset_type)` — 返回 (variant_key, selected_key, prompt_key)，当前支持 `character_headshot` 和默认
- select/delete/favorite 三个 API，白名单：`character`, `character_headshot`, `scene`, `prop`

### 前端已有
- `renderVariantStrip(asset, assetType)` — 渲染变体条（含删除×、收藏☆）
- `selectedUrl(asset, assetType)` — 获取当前选中变体 URL
- `variantStrip` 事件委托 — 处理 select/delete/favorite 点击
- 支持 `character_headshot` 分支

### 缺失
- 后端：`character_three_view` 类型不在白名单中
- 后端：`_get_variant_fields` 无 three_view 分支
- 后端：`_migrate_project_assets` 未初始化 three_view 字段
- 前端：`selectedUrl` / `renderVariantStrip` 未处理 three_view

## 修改方案

### 后端 main.py

**1. `_get_variant_fields`（行574）**
```python
def _get_variant_fields(asset_type: str) -> tuple:
    if asset_type == "character_headshot":
        return "headshot_variants", "headshot_selected_variant_id", "headshot_prompt"
    if asset_type == "character_three_view":
        return "three_view_variants", "three_view_selected_variant_id", "three_view_prompt"
    return "variants", "selected_variant_id", "prompt"
```

**2. `_migrate_project_assets`（行580）**
在 character 循环中新增：
```python
char.setdefault("three_view_variants", [])
char.setdefault("three_view_selected_variant_id", None)
char.setdefault("three_view_prompt", "")
```

**3. select/delete/favorite API 白名单**
三个接口的 `asset_type not in (...)` 检查加入 `"character_three_view"`。
collection 映射加入 `"character_three_view": "characters"`。

### 前端 project.html

**4. `selectedUrl` 函数（行1261）**
新增 three_view 分支：
```javascript
function selectedUrl(asset, assetType) {
    const isHeadshot = assetType === 'character_headshot';
    const isThreeView = assetType === 'character_three_view';
    const variants = (isThreeView ? asset.three_view_variants : isHeadshot ? asset.headshot_variants : asset.variants) || [];
    const sid = isThreeView ? asset.three_view_selected_variant_id : isHeadshot ? asset.headshot_selected_variant_id : asset.selected_variant_id;
    ...
}
```

**5. `renderVariantStrip` 函数（行1227）**
同样新增 three_view 分支获取 variants 和 sid。

## 不做的事

- 变体条/网格的 CSS 样式差异（留给 char-workbench / scene-prop-modal）
- 新增 HTML 元素（当前单一 `#variantStrip` 足够，多列时由 char-workbench 处理）
- prompt 相关逻辑（由 char-workbench 处理）

## 风险

- 无。改动是纯扩展，不影响现有 character/scene/prop 逻辑。
