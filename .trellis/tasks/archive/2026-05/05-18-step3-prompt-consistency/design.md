# Design：Step3 资产生成 Prompt 质量与一致性提升

## 架构边界

所有改动限于：
- `main.py`：后端 prompt 拼接逻辑、I2I ref 传递
- `static/project.html`：弹窗 UI（B1 的复选框）

不引入新路由、不改数据结构（仅加 `ComicAssetRegenRequest.use_current_as_ref` 字段）。

## 数据流

### A1 — Base Prompt 写回

```
regenerate_comic_asset()
  → 构建 base（不含 style）
  → 生成图片
  → _write() 回调中：entry[prompt_key] = base   ← 新增
```

`prompt_key` 由 `_get_variant_fields(asset_type)` 第3个返回值决定：
- character → `"prompt"`
- character_headshot → `"headshot_prompt"`
- character_three_view → `"three_view_prompt"`
- scene / prop → `"prompt"`

### A2 — Style 重复检测

```python
# 当前
base = f"...{style_prompt}..."

# 改后（在各 asset_type 分支末尾，拼接 style 之前）
if style_prompt and body.apply_style and style_prompt not in base:
    base = f"{base} {style_prompt}"
```

实际上当前 style 是内联在 f-string 里的，需要把 style 拼接提取出来统一处理。

### A3 — Negative Prompt 合并

```python
# 当前（main.py:3437）
negative_prompt = body.negative_prompt or art_direction.get("style_negative_prompt", "")

# 改后
art_neg = art_direction.get("style_negative_prompt", "")
negative_prompt = ", ".join(filter(None, [body.negative_prompt, art_neg]))
```

### B1 — use_current_as_ref

```
ComicAssetRegenRequest.use_current_as_ref: bool = False

regenerate_comic_asset()
  → 若 use_current_as_ref=True：
      vk, sk, _ = _get_variant_fields(asset_type)
      sid = item.get(sk)
      ref_url = next((v["url"] for v in item.get(vk,[]) if v["id"]==sid), None)
      if not ref_url and item.get(vk):
          ref_url = item[vk][-1]["url"]  # fallback to latest
      ref_images = [{"url": ref_url, "name": item.get("name","")}] if ref_url else None
  → generate_ai_image(prompt, size, "auto", model, ref_images, provider["id"])
```

前端：弹窗加复选框，仅在 `selectedUrl(asset, charType)` 有值时显示。

### B2 — 派生层级自动 ref

```
regenerate_comic_asset()，asset_type in (character_three_view, character_headshot)
  → 查找 full body ref：
      fb_sid = item.get("selected_variant_id")
      fb_url = next((v["url"] for v in item.get("variants",[]) if v["id"]==fb_sid), None)
      if not fb_url and item.get("variants"):
          fb_url = item["variants"][-1]["url"]
  → 若 fb_url 存在：ref_images = [{"url": fb_url, "name": item.get("name","")}]
  → 否则：ref_images = None（降级 T2I）
```

注意：B1 和 B2 可能同时触发（用户勾选 use_current_as_ref 且是 headshot）。优先级：B2（full body ref）> B1（current ref），因为 full body 是更权威的参考。

### B3 — 反向生成（上传图 → Full Body）

```
regenerate_comic_asset()，asset_type == "character"
  → 若 ref_images 尚未设置（B1 未触发）：
      检测 three_view_variants 中 is_uploaded_source=True 的 variant
      若无，检测 headshot_variants 中 is_uploaded_source=True 的 variant
      若找到，ref_images = [{"url": uploaded_url, "name": item.get("name","")}]
```

### C1 — 批量生成 I2I

```
generate_comic_assets()，处理 character headshot 时：
  → 查找 char.get("selected_variant_id") 对应的 full body url
  → 若存在：ref_images = [{"url": fb_url}]
  → generate_ai_image(prompt, size, "auto", model, ref_images, provider["id"])
```

## 兼容性

- `generate_ai_image` 已支持 `reference_images` 参数，无需改签名
- `AIReference` 是 Pydantic model，但 `generate_ai_image` 接受 dict list，需传 `[{"url":..., "name":...}]`
- 数据结构无变更，`prompt_key` 字段已存在于 migrate 逻辑中

## 风险点

- B1/B2/B3 的 ref url 是相对路径（如 `comic_char_xxx.png`），`generate_ai_image` 内部需要能处理相对路径。需确认 `output_file_from_url()` 的行为。
