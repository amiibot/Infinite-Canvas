# 共享变体操作(选择/删除/收藏)+后端

## Goal

为资产弹窗提供完整的变体管理 UI（选择、删除、收藏），对接已有后端 API，并新增 three_view 数据结构支持。

## 已有后端 API

- `POST /api/comic/projects/{pid}/assets/{asset_type}/{asset_id}/variants/{variant_id}/select`
- `DELETE /api/comic/projects/{pid}/assets/{asset_type}/{asset_id}/variants/{variant_id}`
- `POST /api/comic/projects/{pid}/assets/{asset_type}/{asset_id}/variants/{variant_id}/favorite`

## Requirements

### 后端

- 新增 `three_view` asset_type 支持（数据结构同 headshot：`three_view_variants`、`three_view_selected_variant_id`、`three_view_prompt`）
- `_get_variant_fields` 新增 three_view 分支：返回 `"three_view_variants", "three_view_selected_variant_id", "three_view_prompt"`
- `_migrate_asset_variants` 对 character 新增 three_view 字段初始化
- 确保 select/delete/favorite API 对 `character_three_view` 类型正常工作
- `regenerate_comic_asset` 支持 `asset_type=character_three_view`，prompt 逻辑同 luna-studio：
  - 若前端传 prompt 则使用前端值
  - 否则后端拼接：`"Character Reference Sheet for {name}. {description}. Three-view character design: Front view, Side view, and Back view..."`
  - 当存在 full_body 图或上传图时，前端注入 `STRICTLY MAINTAIN` 前缀

### 前端（共享 JS 函数）

- `selectVariant(assetType, assetId, variantId, subType?)` — 调用 select API，更新 UI
- `deleteVariant(assetType, assetId, variantId, subType?)` — 调用 delete API，确认弹窗，更新 UI
- `favoriteVariant(assetType, assetId, variantId, subType?)` — 调用 favorite API，切换心形图标
- 变体条/网格 UI 组件：每个变体缩略图 + 选中态 + 收藏图标 + 删除按钮（hover 显示）

## Acceptance Criteria

- [ ] three_view 数据结构在后端正确初始化和持久化
- [ ] 点击变体缩略图 → 选中，主图更新
- [ ] 点击删除 → 确认后移除变体，若删除当前选中则自动选下一个
- [ ] 点击收藏 → 心形切换，收藏的变体不被自动清理
- [ ] 所有操作对 character(full_body/headshot/three_view)、scene、prop 均有效

## 依赖

无（基础模块，其他 child 依赖此任务）
