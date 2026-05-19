# 角色工作台列上传按钮

## Goal

为角色工作台每列（Full Body / Three View / Headshot）添加独立上传按钮，允许用户直接在工作台上传参考图。

## Requirements

### 前端
- 每列 header 区域添加上传图标按钮（📎 或 upload icon）
- 点击触发文件选择器（accept image/*）
- 上传后调用现有 `upload_asset_image` 接口，标记 `is_uploaded_source: true`
- 上传成功后刷新该列图片和变体条
- 上传中显示 loading 状态

### 后端
- 复用现有 `/comic-projects/{id}/assets/{asset_id}/upload` 接口
- 确保 `asset_type` 支持 `character_three_view`（已在之前任务中完成）

## 约束
- luna-studio 中上传后会触发 "reverse generation" 提示（STRICTLY MAINTAIN 前缀），luna-canvas 已有此逻辑，只需确保工作台上传也能正确触发

## Acceptance Criteria

- [ ] 每列 header 有上传按钮
- [ ] 点击上传按钮可选择图片文件
- [ ] 上传成功后变体条出现新变体（带 is_uploaded_source 标记）
- [ ] 上传中按钮显示 loading
- [ ] `python3 -m py_compile main.py` 通过
