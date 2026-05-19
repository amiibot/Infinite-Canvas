# 角色双面板数据隔离：Full Body / Headshot 独立存储

## Goal

让角色弹窗的 Full Body 和 Headshot 两个 tab 拥有完全独立的变体列表、选中状态和生成提示词，消除当前共用同一 `variants` 数组导致的相互干扰。

## 背景

当前角色资产只有一个 `variants` 数组和一个 `selected_variant_id`，`character` 和 `character_headshot` 两种生成类型共用这些字段。这导致：
- 重生成全身图会覆盖头像的选中状态（反之亦然）
- 删除/收藏变体会影响另一侧
- `prompt` 字段也是共用的，无法为两种类型分别保存提示词

## 确认事实（来自代码探索）

- 角色数据结构（main.py 行 3198-3208）：`variants`, `selected_variant_id`, `asset_type: "full_body"`, `prompt`（新增）
- `_migrate_project_assets`（行 574-581）：已迁移 `full_body_image_url` 和 `headshot_image_url` 两个旧字段到同一 `variants` 数组
- PATCH 接口（行 3125-3150）：`character_headshot` 已映射到 `characters` 集合，但写入同一 `variants`
- `regenerate_comic_asset`：`character` 和 `character_headshot` 分支已分开，生成不同尺寸/前缀，但都写入同一 `variants`
- 变体操作接口（select/delete/favorite）：`character_headshot` 已支持，但操作同一 `variants`
- 前端 `renderVariantStrip(asset, assetType)`：已接受 `assetType` 参数，但渲染的是同一 `asset.variants`

## Requirements

### R1：后端数据模型扩展
- 角色新增字段：`headshot_variants: []`、`headshot_selected_variant_id: null`、`headshot_prompt: ""`
- 全身图继续使用现有 `variants` / `selected_variant_id` / `prompt`
- 迁移：`_migrate_project_assets` 中为旧角色数据补充这三个新字段的默认值

### R2：后端接口路由
- PATCH 接口：`character_headshot` 类型写入 `headshot_variants` / `headshot_prompt`
- `regenerate_comic_asset`：`character_headshot` 分支写入 `headshot_variants` / `headshot_selected_variant_id`
- 变体操作接口（select/delete/favorite）：`character_headshot` 操作 `headshot_variants`
- 辅助函数：新增 `_headshot_selected_url(item)` 或扩展 `_selected_url` 支持 headshot

### R3：前端适配
- `renderVariantStrip(asset, assetType)`：按 `assetType` 读取对应变体数组
- `openAssetModal`：角色资产按当前 tab 读取对应图片 URL
- `syncAssetModalCharTabs`：切换 tab 时刷新对应变体条和图片
- `saveAssetModalDescription`：`character_headshot` tab 时保存到 `headshot_prompt`
- 上传处理：已在上一轮修复，按 tab 路由到正确字段

## Acceptance Criteria

- [ ] Full Body tab 重生成不影响 Headshot tab 的变体列表（反之亦然）
- [ ] 切换 tab 后变体条显示对应类型的变体
- [ ] 删除/收藏 Full Body 变体不影响 Headshot 变体（反之亦然）
- [ ] 两个 tab 可以分别保存独立的生成提示词
- [ ] 旧数据（无 headshot_variants 字段）打开弹窗不报错，自动补充默认值
- [ ] 场景/道具弹窗不受影响
- [ ] Python 语法检查通过，JS 无报错

## Out of Scope

- Three-View 三视图（需要更多新字段，下一轮）
- 跨项目资产导入

## Open Questions

无
