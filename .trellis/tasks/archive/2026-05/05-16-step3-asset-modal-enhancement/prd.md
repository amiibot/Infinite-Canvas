# Step3 资产弹窗增强：从 luna-studio 回迁核心功能

## Goal

将 luna-studio 中 CharacterWorkbench 的核心功能回迁到 luna-canvas 的资产弹窗，提升 Step3 的资产编辑体验。本次不包含视频/Motion Ref 相关功能（属于 Step5 范畴）。

## 背景

luna-canvas 现有资产弹窗（`openAssetModal`）已有基础骨架：描述编辑、风格应用、负面提示词、批量生成、变体条。但与 luna-studio 相比缺少：
1. 每个资产独立的提示词编辑器（prompt 字段未持久化）
2. Art Direction 风格折叠预览（只有复选框，看不到实际内容）
3. 逆向生成提示词注入（上传图片后自动加 STRICTLY MAINTAIN 前缀）
4. 角色弹窗的 Full Body / Headshot 双面板布局（现在是单图 + char_type tab）

## 确认事实（来自代码探索）

- 后端 PATCH `/api/comic/projects/{pid}/assets/{type}/{id}` 目前只保存 `description`，不保存 `prompt`
- 角色资产有 `full_body_image_url` 和 `headshot_image_url` 两个独立图片字段
- 变体系统已完整实现（variants 数组 + selected_variant_id）
- `state.modalCharType` 已有 `character` / `character_headshot` 切换逻辑
- 风格预览区 `assetModalStylePreview` 已存在但内容为空（只显示 monospace 文本）
- 变体的 `is_uploaded_source` 字段在 luna-canvas 后端尚未实现

## Requirements

### R1：每资产提示词编辑器
- 弹窗右侧新增"生成提示词"编辑区（textarea），可编辑当前资产的 prompt
- 提示词随"重新生成"按钮一起使用（传给后端）
- 提示词持久化：点击保存或生成时，通过 PATCH 接口保存到资产的 `prompt` 字段
- 后端 PATCH 接口扩展支持 `prompt` 字段（characters/scenes/props 均支持）

### R2：Art Direction 风格折叠预览
- 将现有"应用风格方向"区域改为可展开/折叠的面板
- 展开后显示：当前风格的 positive prompt 和 negative prompt 实际内容
- 折叠时只显示复选框 + "已应用风格方向"摘要文字
- 无风格时隐藏整个区域

### R3：逆向生成提示词注入
- 检测资产是否有 `is_uploaded_source=true` 的变体
- 若有，打开弹窗时在提示词前自动注入：`STRICTLY MAINTAIN the SAME character appearance, face, hairstyle, skin tone, and clothing as the reference image. `
- 仅在提示词为空或为默认值时注入（不覆盖用户已编辑的提示词）
- 后端变体结构新增 `is_uploaded_source` 字段（上传时标记）

### R4：角色弹窗双面板布局
- 角色弹窗左侧改为 Full Body / Headshot 两个标签页切换
- 每个标签页独立显示对应图片、变体条、提示词编辑器
- 切换标签页时更新 `state.modalCharType`，生成按钮对应当前标签页类型
- 场景/道具弹窗保持单面板不变

## Acceptance Criteria

- [ ] 打开任意资产弹窗，右侧有可编辑的提示词 textarea，内容来自资产的 `prompt` 字段
- [ ] 修改提示词后点击"重新生成"，新提示词被传给后端并持久化
- [ ] 风格区域可折叠，展开后显示实际 prompt 文本
- [ ] 上传图片后重新打开弹窗，提示词自动包含 STRICTLY MAINTAIN 前缀
- [ ] 角色弹窗左侧有 Full Body / Headshot 两个标签，切换后图片和变体条对应更新
- [ ] 场景/道具弹窗不受影响，保持原有布局
- [ ] Python 语法检查通过，JS 无报错

## Out of Scope

- Three-View 三视图面板（需要新后端字段，下一轮处理）
- Motion Ref / 视频生成（Step5 范畴）
- 资产库跨项目导入
- 提示词模板库

## Open Questions

无
