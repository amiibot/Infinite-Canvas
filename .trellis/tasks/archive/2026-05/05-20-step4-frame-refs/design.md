# Step4 多参考图分镜生成 - 技术设计

## 架构概览

两个改动点：
1. **`analyze_storyboard`**：改造 LLM prompt，让其输出帧级实体关联（character_names/scene_name/prop_names），后端按名字匹配映射为 ID
2. **`_bg_render_frame`**：根据帧的 character_ids/scene_id/prop_ids 收集资产参考图，传给 generate_ai_image

附带修复：`generate_ai_image` 中 `image_refs[:4]` → `[:16]`

## 数据流

```
analyze_storyboard (LLM)
  → LLM 输出 character_ref_names / scene_ref_name / prop_ref_names
  → 后端按名字匹配 → 写入 frame.character_ids / scene_id / prop_ids

_bg_render_frame
  → 读取 frame.character_ids / scene_id / prop_ids
  → 从 project.characters/scenes/props 中找到对应资产
  → 按优先级提取最佳图片 URL（three_view > full_body > headshot）
  → 组装 reference_images 列表
  → 传给 generate_ai_image
```

## 详细设计

### 1. analyze_storyboard 改造

**Prompt 改造**：
- 在 system prompt 中提供项目已有实体列表（名字 + 简短描述）
- 要求 LLM 输出格式增加 `character_ref_names`、`scene_ref_name`、`prop_ref_names`
- 如果项目无实体，entities 部分为空列表，LLM 照常生成（graceful degradation）

**输出格式**（从 luna-studio 借鉴，简化版）：
```json
[{
  "action_description": "...",
  "dialogue": "...",
  "camera_movement": "...",
  "image_prompt": "...",
  "character_ref_names": ["角色A", "角色B"],
  "scene_ref_name": "场景名",
  "prop_ref_names": ["道具1"]
}]
```

**名字→ID 映射**（参考 luna-studio pipeline.py:907-923）：
- 角色：遍历 project.characters，`char.name == ref_name or ref_name in char.name`
- 场景：同理，fallback 到第一个场景
- 道具：同理

### 2. _bg_render_frame 参考图收集

新增辅助函数 `_collect_frame_refs(project, frame) -> list[dict]`：

```python
def _collect_frame_refs(project, frame):
    refs = []
    characters = project.get("characters", [])
    scenes = project.get("scenes", [])
    props = project.get("props", [])
    
    # 角色参考图（优先级：three_view > full_body > headshot）
    for cid in (frame.get("character_ids") or []):
        char = next((c for c in characters if c["id"] == cid), None)
        if not char:
            continue
        url = _get_char_best_url(char)
        if url:
            refs.append({"url": url, "name": char.get("name", "")})
    
    # 场景参考图
    sid = frame.get("scene_id")
    if sid:
        scene = next((s for s in scenes if s["id"] == sid), None)
        if scene:
            url = _get_asset_best_url(scene)
            if url:
                refs.append({"url": url, "name": scene.get("name", "")})
    
    # 道具参考图
    for pid in (frame.get("prop_ids") or []):
        prop = next((p for p in props if p["id"] == pid), None)
        if not prop:
            continue
        url = _get_asset_best_url(prop)
        if url:
            refs.append({"url": url, "name": prop.get("name", "")})
    
    return refs or None
```

**角色最佳图优先级**：
```python
def _get_char_best_url(char):
    for variants_key, sid_key in [
        ("three_view_variants", "three_view_selected_variant_id"),
        ("variants", "selected_variant_id"),
        ("headshot_variants", "headshot_selected_variant_id"),
    ]:
        sid = char.get(sid_key)
        url = next((v["url"] for v in char.get(variants_key, []) if v["id"] == sid), None) if sid else None
        if not url and char.get(variants_key):
            url = char[variants_key][-1]["url"]
        if url:
            return url
    return None
```

**场景/道具最佳图**（复用已有模式）：
```python
def _get_asset_best_url(item):
    sid = item.get("selected_variant_id")
    url = next((v["url"] for v in item.get("variants", []) if v["id"] == sid), None) if sid else None
    if not url and item.get("variants"):
        url = item["variants"][-1]["url"]
    return url
```

### 3. generate_ai_image 限制修复

3 处 `image_refs[:4]` → `image_refs[:16]`（行 1976, 1985, 2015）
1 处 `refs[:4]` → `refs[:16]`（行 2048）

## 兼容性

- 已有项目的帧 character_ids 为空 → 不传参考图，行为不变
- 重新执行 AI 生成分镜后，帧会获得关联 ID → 渲染时自动传参考图
- 手动添加的帧仍然没有关联（后续可加 UI），不影响渲染

## 风险

- LLM 输出的角色名可能与项目中的名字不完全匹配（如"小明" vs "张小明"）→ 用 `in` 模糊匹配缓解
- 参考图过多可能增加 API token 成本（GPT-Image-2 高保真处理每张输入图）→ 实际场景通常 3-6 张，可接受
