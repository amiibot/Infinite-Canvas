# 角色双面板数据隔离 — 技术设计

## 数据模型变更

### 新增字段（角色资产）

```
headshot_variants: []                  # 头像变体列表，结构同 variants
headshot_selected_variant_id: null     # 头像当前选中变体 ID
headshot_prompt: ""                    # 头像生成提示词
```

全身图继续使用现有 `variants` / `selected_variant_id` / `prompt`，不做改动。

### 迁移策略（方案 A）

`_migrate_project_assets` 中为旧角色补充默认值，不尝试拆分旧 `variants`：

```python
char.setdefault("headshot_variants", [])
char.setdefault("headshot_selected_variant_id", None)
char.setdefault("headshot_prompt", "")
```

旧数据中混入 `variants` 的头像图片留在全身图侧，用户重生成头像时产生新的独立变体。

---

## 后端路由规则

所有接口按 `asset_type` 决定操作哪组字段：

| asset_type | variants 字段 | selected_variant_id 字段 | prompt 字段 |
|---|---|---|---|
| `character` | `variants` | `selected_variant_id` | `prompt` |
| `character_headshot` | `headshot_variants` | `headshot_selected_variant_id` | `headshot_prompt` |

### 辅助函数

新增 `_get_variant_fields(asset_type: str) -> tuple[str, str, str]`：

```python
def _get_variant_fields(asset_type: str) -> tuple:
    if asset_type == "character_headshot":
        return "headshot_variants", "headshot_selected_variant_id", "headshot_prompt"
    return "variants", "selected_variant_id", "prompt"
```

### 受影响接口

1. **PATCH** `update_asset_description`：按 `asset_type` 写入对应 variants/prompt 字段
2. **POST** `regenerate_comic_asset`：`character_headshot` 分支调用 `_append_variant_hs`（或复用 `_append_variant` 传字段名）
3. **POST** `select_asset_variant`：按 `asset_type` 操作对应 selected_variant_id
4. **DELETE** `delete_asset_variant`：按 `asset_type` 操作对应 variants
5. **POST** `favorite_asset_variant`：按 `asset_type` 操作对应 variants

### `_append_variant` 扩展

新增 `variant_key` / `selected_key` 参数，默认值保持向后兼容：

```python
def _append_variant(item, url, is_uploaded_source=False,
                    variant_key="variants", selected_key="selected_variant_id"):
```

---

## 前端路由规则

### `selectedUrl(asset, assetType)`

扩展为按 `assetType` 读取对应字段：

```javascript
function selectedUrl(asset, assetType) {
    const isHeadshot = assetType === 'character_headshot';
    const variants = (isHeadshot ? asset.headshot_variants : asset.variants) || [];
    const sid = isHeadshot ? asset.headshot_selected_variant_id : asset.selected_variant_id;
    ...
}
```

### `renderVariantStrip(asset, assetType)`

按 `assetType` 读取对应 variants 数组和 selected_variant_id。

### `openAssetModal`

角色资产初始化时用 `selectedUrl(asset, 'character')` 读全身图 URL。

### `syncAssetModalCharTabs`

切换 tab 时用 `selectedUrl(asset, state.modalCharType)` 刷新图片。

### `saveAssetModalDescription`

`state.modalAssetType` 已是 `character` 或 `character_headshot`（由 tab 决定），PATCH 请求路径已正确，后端按 `asset_type` 路由写入对应字段即可。

---

## 兼容性

- 旧角色数据无 `headshot_variants` → 迁移函数补充默认值，前端读取时用 `|| []` 兜底
- `_append_variant` 新增参数有默认值，所有现有调用点无需修改
- 场景/道具不受影响（不走 `character_headshot` 路径）
