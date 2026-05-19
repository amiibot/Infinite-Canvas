# PRD：Step3 资产生成 Prompt 质量与一致性提升

## Goal

将 luna-studio 中已验证的 prompt 拼接逻辑和 I2I 参考图机制移植到 luna-canvas，提升角色/场景/道具资产的生成质量和跨视角一致性。

## 已确认事实（代码验证）

- `regenerate_comic_asset`（main.py:3429）：`generate_ai_image` 第5参数 `reference_images` 始终传 `None`，I2I 从未启用
- `generate_comic_assets`（main.py:3349）：批量生成同上，全部 T2I
- `ComicAssetRegenRequest`（main.py:545）：已有 `size` 字段，但无 `use_current_as_ref` 字段
- `_get_variant_fields`（main.py:580）：返回 `(variant_key, selected_key, prompt_key)`，prompt_key 已存在于数据结构
- 前端弹窗（project.html:1238）：读取 `asset[promptKey]` 作为 textarea 默认值，但后端生成后从不写回 prompt_key
- `selectedUrl()`（project.html:1409）：已能正确获取各类型的 selected variant url
- `STRICTLY MAINTAIN` 前缀（main.py:3521）：已在 prompt 里加，但没有把对应图片传给 API
- style 拼接（main.py:3458-3505）：`f"{base} {style_prompt}"` 无重复检测
- negative prompt（main.py:3437）：`body.negative_prompt or art_neg`，二选一，不合并

## Requirements

### A. Prompt 质量提升（无架构变动）

**A1. Base Prompt 写回**
- 生成完成后，把不含 style 的 `base` 写回 `item[prompt_key]`（在 `_write` 回调里执行）
- 前端弹窗下次打开时自动显示上次的描述 prompt（已有读取逻辑，无需改前端）

**A2. Style 重复检测**
- 拼接前：`if style_prompt and style_prompt not in base`，才追加 style
- 防止用户手动输入了 style 关键词后被重复追加

**A3. Negative Prompt 合并**
- `effective_neg = ", ".join(filter(None, [body.negative_prompt, art_neg]))`
- 用户自定义 neg 与 art direction neg 同时生效，而非二选一

### B. I2I 参考图传递（核心一致性）

**B1. 单个重生成时可选传入当前图作参考**
- `ComicAssetRegenRequest` 加 `use_current_as_ref: bool = False`
- 后端：若 `use_current_as_ref=True`，读取 `selected_variant` 的 url，构造 `[{"url": url}]` 传给 `generate_ai_image`
- 前端弹窗：加"参考当前图"复选框（默认**关**），仅在有已生成图时显示

**B2. 角色派生层级：Three View / Headshot 自动使用 Full Body 作参考**
- `character_three_view` / `character_headshot` 生成时：
  1. 查找 `character.selected_variant_id` 对应的 full body url
  2. 若存在，自动作为 ref（始终启用，无需用户开关）
  3. 若不存在，降级为纯 T2I（当前行为）
- 同时保留 `STRICTLY MAINTAIN` prompt 前缀

**B3. 反向生成：上传图 → 生成 Full Body**
- 生成 Full Body（`asset_type="character"`）时，检测是否有 `is_uploaded_source=True` 的三视图或头像 variant
- 若有，自动作为 ref 传入（优先级：三视图 > 头像）
- 无需用户操作，自动触发

### C. 批量生成传参

**C1. `generate_comic_assets` 也走 I2I 逻辑**
- 批量生成时，若角色已有 full body selected variant，headshot 生成自动带 ref
- 场景/道具无参考图逻辑，保持 T2I

## Acceptance Criteria

- [ ] 重生成后，弹窗再次打开时 prompt textarea 显示上次的描述（不含 style）
- [ ] style_prompt 不会被重复拼接进 prompt（style 已在 base 中时不再追加）
- [ ] 用户在弹窗填写 negative prompt 时，与 art direction neg 合并生效
- [ ] 勾选"参考当前图"后，重生成请求携带当前 selected variant url 作为参考图
- [ ] character_three_view / character_headshot 生成时，若有 full body selected variant，自动使用作参考
- [ ] 上传三视图/头像后，生成 full body 时自动使用上传图作参考
- [ ] 批量生成时，角色 headshot 若有 full body selected variant 则自动带 ref

## Out of Scope

- 异步 task 轮询系统
- Art Direction AI 推荐系统（Step2）
- Series 级资产继承
- base_character_id 变体链接

## Open Questions

（已无阻塞性问题，用户确认 A1-C1 全部实现）
