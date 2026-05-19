# 角色双面板数据隔离 — 实施计划

## 实施顺序

### 阶段 A：后端（main.py）

- [ ] A1. 新增辅助函数 `_get_variant_fields(asset_type)` → 返回 `(variant_key, selected_key, prompt_key)`
- [ ] A2. `_append_variant` 新增 `variant_key` / `selected_key` 参数（默认值保持兼容）
- [ ] A3. `_migrate_project_assets`：角色补充 `headshot_variants` / `headshot_selected_variant_id` / `headshot_prompt` 默认值
- [ ] A4. PATCH `update_asset_description`：按 `_get_variant_fields` 路由写入对应字段
- [ ] A5. `regenerate_comic_asset`：`character_headshot` 分支调用 `_append_variant` 时传入 headshot 字段名
- [ ] A6. `select_asset_variant`：按 `_get_variant_fields` 操作对应字段
- [ ] A7. `delete_asset_variant`：按 `_get_variant_fields` 操作对应字段
- [ ] A8. `favorite_asset_variant`：按 `_get_variant_fields` 操作对应字段

### 阶段 B：前端（project.html）

- [ ] B1. `selectedUrl(asset, assetType)` 扩展：按 `assetType` 读取对应字段（新增参数，默认 `'character'` 保持兼容）
- [ ] B2. `renderVariantStrip(asset, assetType)`：按 `assetType` 读取对应 variants 数组和 selected_variant_id
- [ ] B3. `openAssetModal`：角色资产用 `selectedUrl(asset, 'character')` 初始化图片
- [ ] B4. `syncAssetModalCharTabs`：用 `selectedUrl(asset, state.modalCharType)` 刷新图片
- [ ] B5. `saveAssetModalDescription`：`state.modalAssetType` 已正确（`character` 或 `character_headshot`），无需改动

### 阶段 C：验证

- [ ] C1. `python3 -m py_compile main.py`
- [ ] C2. 启动服务器，打开角色弹窗
- [ ] C3. Full Body 重生成 → Headshot tab 变体条不变
- [ ] C4. Headshot 重生成 → Full Body tab 变体条不变
- [ ] C5. 切换 tab → 变体条和图片正确切换
- [ ] C6. 旧数据（无 headshot_variants）打开弹窗不报错
- [ ] C7. 场景/道具弹窗正常

---

## 关键文件

| 文件 | 修改位置 |
|------|---------|
| `main.py` | `_append_variant`（592）、`_migrate_project_assets`（574）、PATCH接口（3125）、`regenerate_comic_asset`（3391）、select/delete/favorite（3488-3553） |
| `static/project.html` | `selectedUrl`（1144）、`renderVariantStrip`（1111）、`openAssetModal`（1154）、`syncAssetModalCharTabs`（988） |

---

## 风险点

1. `_append_variant` 新增参数后，所有现有调用点需确认不受影响（默认值兜底）
2. `renderVariantStrip` 变体条点击事件（select/delete/favorite）需要知道当前 `assetType`，确认事件委托能拿到正确值
3. 前端 `selectedUrl` 被多处调用，扩展参数时需检查所有调用点

---

## 回滚点

- 所有后端改动向后兼容（新增字段有默认值，旧字段不删除）
- 前端 `selectedUrl` 新增参数有默认值，旧调用点不受影响
