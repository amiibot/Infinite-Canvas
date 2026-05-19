# 资产锁定 asset-lock

## Goal

为 Step3 资产添加锁定功能。锁定后禁止重新生成和删除操作，防止误操作覆盖满意的结果。

## Requirements

### 后端
- 资产数据新增 `locked: bool` 字段（默认 false）
- `_migrate_project_assets` 中 `setdefault("locked", False)`
- 新增 `POST /comic-projects/{id}/assets/{asset_id}/toggle-lock` 接口
  - 请求体：`{ "asset_type": "character" | "scene" | "prop" }`
  - 切换 `locked` 状态并保存
- `regenerate_comic_asset` 接口：若资产 `locked=True`，返回 403
- `delete_variant` 接口：若资产 `locked=True`，返回 403

### 前端 — 角色工作台
- luna-studio 的 lock 含义：Three View / Headshot 列在没有 Full Body 图片时显示锁定遮罩（"Generate Master Asset first"）
- luna-canvas 对齐：当角色无 full_body_image_url 且无上传图时，Three View 和 Headshot 列显示锁定遮罩
- 这是**条件锁定**（依赖 Full Body 是否存在），不是用户手动锁

### 前端 — 用户手动锁定
- 变体条区域添加锁定/解锁图标按钮
- 锁定后：生成按钮 disabled，删除按钮隐藏，显示锁图标
- 解锁后恢复正常

## Acceptance Criteria

- [ ] 后端 toggle-lock 接口正常切换 locked 状态
- [ ] 锁定资产调用 regenerate 返回 403
- [ ] 锁定资产调用 delete variant 返回 403
- [ ] 角色工作台：无 Full Body 时 Three View/Headshot 列显示遮罩
- [ ] 手动锁定后 UI 正确禁用生成/删除
- [ ] `_migrate_project_assets` 包含 `setdefault("locked", False)`
- [ ] `python3 -m py_compile main.py` 通过
