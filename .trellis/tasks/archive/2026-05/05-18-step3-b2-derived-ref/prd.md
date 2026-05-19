# B2 派生层级自动 Full Body ref

## Goal

生成 Three View 或 Headshot 时，自动把角色的 Full Body selected variant 作为参考图传给图片 API，无需用户操作，确保派生视角与主图外观一致。

## 依赖

**前置任务**：`05-18-step3-b1-use-ref` 必须已合并并通过测试（ref_images 传递链路已验证）。

## 执行流程

1. **实现**：由主 Claude 落代码（main.py only）
2. **Review**：由 Codex 审查代码变更
3. **修改**：主 Claude 根据 review 意见修改
4. **测试**：语法检查通过，人工验证 three_view/headshot 生成请求携带 full body url
5. **通过后**：进入下一任务 `05-18-step3-b3-reverse-gen`

## Requirements

### 后端（main.py only）

- 在 `regenerate_comic_asset` 的 ref_images 构建区域（B1 之后），追加 B2 逻辑：
  - 仅对 `asset_type in ("character_three_view", "character_headshot")` 触发
  - 查找 `item.get("selected_variant_id")` 对应的 full body variant url
  - 若无 selected，取 `item.get("variants", [])` 最后一个的 url
  - 若找到 full body url：`ref_images = [{"url": fb_url, "name": item.get("name", "")}]`（覆盖 B1 的结果）
  - 若无 full body：保持 B1 的 ref_images（可能为 None）
- B2 优先级高于 B1：有 full body 时始终用 full body，不用 current ref

## 关键代码位置

- ref_images 构建区域：B1 实现后的 main.py:3526 附近
- `_get_variant_fields`：main.py:580（full body 对应 `variants` / `selected_variant_id`）

## Acceptance Criteria

- [ ] `uv run python3 -m py_compile main.py` 通过
- [ ] 生成 character_three_view 时，若有 full body selected variant，请求携带其 url 作为参考图
- [ ] 生成 character_headshot 时，同上
- [ ] 无 full body 时，降级为 B1 的行为（use_current_as_ref 开关仍有效）
- [ ] 生成 character（full body）时，B2 逻辑不触发

## Out of Scope

- B3 反向生成（上传图 → full body，单独任务）
- 前端 UI 改动
