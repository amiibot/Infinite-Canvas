# Implement：Step3 资产生成 Prompt 质量与一致性提升

## 子任务拆分

| 子任务 | 文件 | 行数估计 |
|--------|------|---------|
| A1 Base Prompt 写回 | main.py | ~5 |
| A2 Style 重复检测 | main.py | ~10 |
| A3 Negative Prompt 合并 | main.py | ~3 |
| B1 use_current_as_ref | main.py + project.html | ~25 + ~20 |
| B2 派生层级自动 ref | main.py | ~20 |
| B3 反向生成 | main.py | ~15 |
| C1 批量生成 I2I | main.py | ~15 |

## 实现顺序

### Step 1 — A1 + A2 + A3（main.py only）

**A3**：修改 main.py:3437
```python
# 旧
negative_prompt = body.negative_prompt or art_direction.get("style_negative_prompt", "")
# 新
art_neg = art_direction.get("style_negative_prompt", "")
negative_prompt = ", ".join(filter(None, [body.negative_prompt, art_neg]))
```

**A2**：重构各 asset_type 分支的 style 拼接。当前 style 内联在 f-string 里，需提取出来：
```python
# 各分支只写 base（不含 style）
base = f"Full body character design of {item.get('name','')}. {item.get('description','')}. Clean white background, high quality."
# 统一在分支结束后追加 style
if body.apply_style and style_prompt and style_prompt not in base:
    base = f"{base} {style_prompt}"
```

**A1**：在 `_write` 回调里，找到 entry 后写回 prompt_key：
```python
vk, sk, pk = _get_variant_fields(asset_type)
for entry in project.get(collection, []):
    if entry["id"] == asset_id:
        entry[pk] = base_prompt  # base_prompt 需从外层传入（闭包）
        for url in new_urls:
            _append_variant(entry, url, variant_key=vk, selected_key=sk)
```
注意：`base_prompt` 是不含 style 的 base，需在外层保存一份。

### Step 2 — B1（main.py + project.html）

**main.py**：
1. `ComicAssetRegenRequest` 加 `use_current_as_ref: bool = False`
2. 在 `prompt = ...` 构建完成后，构建 ref_images：
```python
ref_images = None
if body.use_current_as_ref:
    vk, sk, _ = _get_variant_fields(asset_type)
    sid = item.get(sk)
    ref_url = next((v["url"] for v in item.get(vk, []) if v["id"] == sid), None)
    if not ref_url and item.get(vk):
        ref_url = item[vk][-1]["url"]
    if ref_url:
        ref_images = [{"url": ref_url, "name": item.get("name", "")}]
```
3. `generate_ai_image(prompt, size, "auto", model, ref_images, provider["id"])`

**project.html**：
在弹窗的生成按钮区域加复选框：
```html
<label id="useRefLabel" style="display:none">
  <input type="checkbox" id="useCurrentAsRef"> 参考当前图
</label>
```
- 打开弹窗时：若 `selectedUrl(asset, charType)` 有值，显示该复选框
- 提交时：body 加 `use_current_as_ref: document.getElementById('useCurrentAsRef').checked`

### Step 3 — B2（main.py only）

在 B1 的 ref_images 构建逻辑之后，追加 B2 逻辑（B2 优先级高于 B1）：
```python
# B2: 派生层级自动使用 full body 作参考（覆盖 B1）
if asset_type in ("character_three_view", "character_headshot"):
    fb_sid = item.get("selected_variant_id")
    fb_url = next((v["url"] for v in item.get("variants", []) if v["id"] == fb_sid), None)
    if not fb_url and item.get("variants"):
        fb_url = item["variants"][-1]["url"]
    if fb_url:
        ref_images = [{"url": fb_url, "name": item.get("name", "")}]
```

### Step 4 — B3（main.py only）

在 B2 之后，仅对 `asset_type == "character"` 且 `ref_images is None` 时触发：
```python
if asset_type == "character" and not ref_images:
    # 优先三视图上传图，其次头像上传图
    for vk_check in ("three_view_variants", "headshot_variants"):
        uploaded = next((v for v in item.get(vk_check, []) if v.get("is_uploaded_source")), None)
        if uploaded:
            ref_images = [{"url": uploaded["url"], "name": item.get("name", "")}]
            break
```

### Step 5 — C1（main.py only）

在 `generate_comic_assets` 的 character headshot 生成处（目前只生成 headshot，约 main.py:3359-3370）：
```python
for char in project_snapshot.get("characters", []):
    # ... build prompt ...
    # C1: 若有 full body，作为参考图
    fb_sid = char.get("selected_variant_id")
    fb_url = next((v["url"] for v in char.get("variants", []) if v["id"] == fb_sid), None)
    if not fb_url and char.get("variants"):
        fb_url = char["variants"][-1]["url"]
    char_ref = [{"url": fb_url}] if fb_url else None
    image_data, _ = await generate_ai_image(build_prompt(base), ..., char_ref, provider["id"])
```

## 验证命令

```bash
uv run python3 -m py_compile main.py && echo "syntax OK"
```

## 风险点

- variant url 格式：`save_ai_image_to_output` 返回 `/output/xxx.png`，`output_file_from_url` 可处理，无风险
- B1/B2 同时触发时 B2 覆盖 B1，符合预期（full body 是更权威的参考）
- B3 仅在 `ref_images is None` 时触发，不会覆盖 B1
- `generate_ai_image` 的 `reference_images` 参数接受 dict list，无需转 AIReference 对象
