# asset-lock 实施计划

## 后端 (main.py)

### 1. Migration — `_migrate_project_assets` (~582行)
在 character/scene/prop 循环中各加 `setdefault("locked", False)`

### 2. Toggle Lock 接口
新增 `POST /api/comic/projects/{pid}/assets/{asset_type}/{asset_id}/toggle-lock`
- 找到资产，切换 `item["locked"]`，保存
- 返回更新后的 project

### 3. Regenerate 守卫 — `regenerate_comic_asset` (~3399行)
找到 item 后检查 `if item.get("locked"): raise HTTPException(403, "Asset is locked")`

### 4. Delete variant 守卫 — `delete_asset_variant` (~3529行)
找到 item 后检查 `if item.get("locked"): raise HTTPException(403, "Asset is locked")`

## 前端 (project.html)

### 5. 角色工作台条件锁
- 渲染每列时检查：若角色无 `full_body_image_url` 且无上传变体，Three View / Headshot 列显示遮罩
- 遮罩内容："先生成全身图" + 锁图标

### 6. 手动锁定按钮
- 变体条区域加 🔒/🔓 按钮
- 点击调用 toggle-lock 接口
- 锁定后：生成按钮 disabled，删除按钮隐藏

## 验证
```bash
python3 -m py_compile main.py && echo "OK"
```
- 锁定资产 → regenerate 返回 403
- 锁定资产 → delete variant 返回 403
- 角色无 full body → TV/HS 列遮罩显示
- 手动锁定/解锁切换正常
