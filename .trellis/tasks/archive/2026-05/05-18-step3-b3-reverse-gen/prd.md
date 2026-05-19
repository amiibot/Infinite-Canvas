# B3 反向生成：上传图作 Full Body ref

## Goal

用户上传三视图或头像后，生成 Full Body 时自动把上传图作为参考图，实现"从上传图反推全身图"的反向生成流程，无需用户手动操作。

## 依赖

**前置任务**：`05-18-step3-b2-derived-ref` 必须已合并并通过测试（ref_images 传递链路已完整验证）。

## 执行流程

1. **实现**：由主 Claude 落代码（main.py only）
2. **Review**：由 Codex 审查代码变更
3. **修改**：主 Claude 根据 review 意见修改
4. **测试**：语法检查通过，人工验证上传图后生成 full body 时请求携带上传图 url
5. **通过后**：进入下一任务 `05-18-step3-c1-batch-ref`

## Requirements

### 后端（main.py only）

- 在 `regenerate_comic_asset` 的 ref_images 构建区域，B2 之后追加 B3 逻辑：
  - 仅对 `asset_type == "character"` 且 `ref_images is None` 时触发
  - 按优先级检测上传图：
    1. `item.get("three_view_variants", [])` 中 `is_uploaded_source=True` 的 variant
    2. `item.get("headshot_variants", [])` 中 `is_uploaded_source=True` 的 variant
  - 若找到：`ref_images = [{"url": uploaded["url"], "name": item.get("name", "")}]`
  - 若未找到：`ref_images = None`（保持当前行为）
- B3 仅在 `ref_images is None` 时触发，不覆盖 B1（用户主动勾选的 use_current_as_ref）

## 关键代码位置

- ref_images 构建区域：B2 实现后的 main.py 对应位置
- `is_uploaded_source` 字段：`_make_variant()` main.py:564 已支持

## Acceptance Criteria

- [ ] `uv run python3 -m py_compile main.py` 通过
- [ ] 角色有上传的三视图时，生成 full body 请求携带三视图 url 作为参考图
- [ ] 无三视图但有上传头像时，生成 full body 请求携带头像 url 作为参考图
- [ ] 无任何上传图时，行为与改动前完全一致
- [ ] 用户勾选了 use_current_as_ref（B1）时，B3 不覆盖（B1 优先）
- [ ] 生成 three_view / headshot 时，B3 逻辑不触发

## Out of Scope

- 前端上传入口改动（已有上传逻辑，is_uploaded_source 已被正确标记）
- C1 批量生成（单独任务）
