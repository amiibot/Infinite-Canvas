# Step4 多参考图分镜生成

## Goal

导演工坊 Step4 分镜渲染时，自动收集当前帧关联的角色/场景/道具资产图作为 reference_images 传给图片生成 API，使出图与项目资产风格一致。

## User Value

当前分镜出图与已有角色/场景完全无关联，用户需要反复手动调整 prompt 才能让分镜图中的角色看起来像项目中设定的角色。加入参考图后，生成的分镜图会自动参考已有资产的视觉风格。

## Confirmed Facts

- `_bg_render_frame` (main.py:5170) 当前传 `reference_images=None`
- frame 数据结构已有 `character_ids`、`scene_id`、`prop_ids` 关联字段
- **但当前这些字段从未被填充**：`analyze_storyboard` 生成帧时全部为空，手动添加也无选择器
- 角色有三种 variant 池：`variants`(全身)、`three_view_variants`、`headshot_variants`
- 场景/道具有 `variants` + `selected_variant_id`
- `generate_ai_image` 已支持 `reference_images` 参数（`List[{"url": str, "name": str}]`）
- GPT-Image-2 实际支持最多 16 张参考图（我们代码中 `[:4]` 是过时限制）；apimart 最多 14 张
- 项目中已有成熟的"取最佳 URL"模式（selected_variant_id → fallback 最后一个）
- MVP 以 GPT-Image-2 为目标
- **luna-studio 做法**：LLM 生成分镜时输出 character_ids/scene_id/prop_ids（用临时 ID 映射），渲染时按帧级关联收集参考图；优先级 three_view > full_body > headshot

## Requirements

- [ ] 改造 `analyze_storyboard` LLM prompt，输出帧级实体关联（character_ref_names/scene_ref_name/prop_ref_names）
- [ ] 后端按名字模糊匹配（双向 `in`）映射为真实 ID，写入 frame 的 character_ids/scene_id/prop_ids
- [ ] 渲染分镜时自动从帧关联的资产中收集参考图
- [ ] 角色参考图优先级：three_view > full_body > headshot
- [ ] 无关联资产或资产无图时，graceful fallback（不传参考图，行为与当前一致）
- [ ] 修复 `generate_ai_image` 中 `[:4]` → `[:16]`（GPT-Image-2 实际支持 16 张）

## Acceptance Criteria

1. AI 生成分镜后，帧的 character_ids/scene_id/prop_ids 被正确填充（名字匹配到的实体）
2. 帧关联了有图的角色时，渲染调用 generate_ai_image 的 reference_images 包含该角色图
3. 帧关联了有图的场景时，reference_images 包含该场景图
4. 帧无关联资产时，reference_images 为 None，行为不变
5. 总参考图数不超过 16 张（GPT-Image-2 上限）
6. Python 语法检查通过，服务启动无报错

## Out of Scope

- 前端 UI 变更（不需要用户手动选择参考图）
- 支持用户自定义参考图优先级
- 针对 apimart 14 张限制的扩展优化（后续可做）

## Open Questions

（全部已解决）

## 决策记录

1. **GPT-Image-2 参考图上限是 16 张**，代码中 `[:4]` 是过时限制，本次一并修复为 `[:16]`
2. **配额分配**：由于 16 张上限远超实际需求（通常 5-6 张），不需要复杂分配策略，直接全部传入
3. **附带修复**：`generate_ai_image` 中 `image_refs[:4]` → `[:16]`，所有参考图场景受益
4. **采用方案 A**：改造 `analyze_storyboard` 让 LLM 输出帧级实体关联（参考 luna-studio 模式），而非项目级 fallback
5. **名字匹配策略**：双向 `in` 模糊匹配（`ref_name in entity.name or entity.name in ref_name`），比 luna-studio 单向匹配更宽容
6. **Graceful degradation**：项目无实体时 LLM 照常生成分镜，关联字段为空，渲染时不传参考图
