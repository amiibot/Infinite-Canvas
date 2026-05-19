# variant-ops Implement

## Checklist

### 后端 main.py

- [ ] 1. `_get_variant_fields`（行574）：新增 `character_three_view` 分支
- [ ] 2. `_migrate_project_assets`（行580-586）：character 循环新增 three_view 字段初始化
- [ ] 3. `select_asset_variant`（行3494）：白名单 + collection 映射加入 `character_three_view`
- [ ] 4. `delete_asset_variant`（行3517）：同上
- [ ] 5. `favorite_asset_variant`（行3540）：同上

### 前端 project.html

- [ ] 6. `selectedUrl`（行1261-1270）：新增 three_view 分支
- [ ] 7. `renderVariantStrip`（行1227-1238）：新增 three_view 分支

### 验证

- [ ] 8. `python3 -m py_compile main.py`
- [ ] 9. 启动服务器，打开 Step3，确认现有 character/scene/prop 弹窗无回归
- [ ] 10. 通过 API 手动测试 three_view select/delete/favorite

## 执行顺序

后端 1→2→3→4→5 → 前端 6→7 → 验证 8→9→10
