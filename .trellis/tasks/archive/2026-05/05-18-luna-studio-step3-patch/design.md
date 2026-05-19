# Design: luna-studio Step3 资产生成补齐 A1/A2/B1

## 数据流总览

```
前端 checkbox/prompt → api.ts:generateAsset → POST /assets/generate
  → api.py:GenerateAssetRequest → pipeline.create_asset_generation_task
  → task["params"] → pipeline.process_asset_generation_task
  → pipeline.generate_asset → assets.generate_scene/prop/character
  → models.Scene.prompt / models.Prop.prompt 写回
  → pipeline._save_data()
```

---

## A1 — prompt 写回

### models.py

`Scene`（line ~180）和 `Prop`（line ~197）各加一个字段：

```python
prompt: Optional[str] = Field(None, description="Prompt used for last generation (without style suffix)")
```

### assets.py — generate_scene（line ~386）

函数签名加 `prompt: str = None` 参数。内部逻辑改为：

```python
def generate_scene(self, scene, positive_prompt=None, negative_prompt="", batch_size=1, model_name=None, size=None, prompt=None, use_current_as_ref=False):
    ...
    if positive_prompt is None:
        positive_prompt = "cinematic lighting, movie still, 8k, highly detailed, realistic"

    # A1: 用户传入 prompt 优先，否则自动构造
    if prompt:
        base_prompt = prompt
    else:
        base_prompt = f"Scene Concept Art: {scene.name}. {scene.description}. High quality, detailed."

    scene.prompt = base_prompt  # A1 写回（不含 style suffix）

    # A2: style 去重追加
    generation_prompt = f"{base_prompt}, {positive_prompt}" if positive_prompt and positive_prompt not in base_prompt else base_prompt
```

### assets.py — generate_prop（line ~434）

同上，自动构造 base_prompt 为 `f"Prop Design: {prop.name}. {prop.description}. Isolated on white background, high quality, detailed."`，写回 `prop.prompt = base_prompt`。

---

## A2 — style 去重

与 A1 同步完成（见上方 `generation_prompt` 构造逻辑）。

Character 已有去重（`style_suffix not in base_prompt`），scene/prop 补齐后三类一致。

---

## B1 — use_current_as_ref 全链路

### api.py — GenerateAssetRequest

```python
use_current_as_ref: bool = False
```

### api.py — 端点调用（line ~462）

在 `pipeline.create_asset_generation_task(...)` 调用中追加 `use_current_as_ref=request.use_current_as_ref`。

### pipeline.py — generate_asset（line ~281）

调用 `generate_scene`/`generate_prop` 时加 `prompt=prompt` 透传：

```python
self.asset_generator.generate_scene(target_asset, effective_positive_prompt, effective_negative_prompt,
    batch_size=batch_size, model_name=t2i_model, size=effective_size, prompt=prompt,
    use_current_as_ref=use_current_as_ref)
self.asset_generator.generate_prop(target_asset, effective_positive_prompt, effective_negative_prompt,
    batch_size=batch_size, model_name=t2i_model, size=effective_size, prompt=prompt,
    use_current_as_ref=use_current_as_ref)
```

`_process_series_asset_task`（line ~421）的 series 路径同样透传 `prompt`（series 路径已有 `prompt` 变量，只需加到调用处）。

### pipeline.py — create_asset_generation_task 签名

加参数 `use_current_as_ref: bool = False`，存入 `task["params"]["use_current_as_ref"]`。

### pipeline.py — process_asset_generation_task

从 `params` 取出 `use_current_as_ref`，传给 `self.generate_asset(...)`。

### pipeline.py — generate_asset 签名

加参数 `use_current_as_ref: bool = False`，在调用 `asset_generator.generate_character/scene/prop` 时透传。

### assets.py — generate_character/scene/prop

各函数加参数 `use_current_as_ref: bool = False`。

**取 ref 逻辑（在现有 ref_image_path 逻辑之前插入，优先级最高）：**

```python
# B1: 用户勾选"参考当前图"
if use_current_as_ref and not ref_image_path:
    asset_obj = character  # 或 scene / prop
    image_asset = getattr(asset_obj, 'image_asset', None)  # scene/prop
    # character 需按 generation_type 选对应 asset
    if image_asset and image_asset.selected_id:
        sel = next((v for v in image_asset.variants if v.id == image_asset.selected_id), None)
        if sel:
            local_path = os.path.join("output", sel.url)
            if os.path.exists(local_path):
                ref_image_path = local_path
```

Character 的三个 tab 各自对应不同 asset：
- `full_body` → `character.full_body_asset`
- `three_view` → `character.three_view_asset`
- `headshot` → `character.headshot_asset`

### api.ts — generateAsset

函数签名末尾加 `useCurrentAsRef: boolean = false`，请求体加 `use_current_as_ref: useCurrentAsRef`。

### CharacterWorkbench.tsx

**state（各自独立）：**
```tsx
const [useCurrentAsRefFullBody, setUseCurrentAsRefFullBody] = useState(false);
const [useCurrentAsRefThreeView, setUseCurrentAsRefThreeView] = useState(false);
const [useCurrentAsRefHeadshot, setUseCurrentAsRefHeadshot] = useState(false);
```

**handleGenerateClick 修改：**
```tsx
const handleGenerateClick = (type: "full_body" | "three_view" | "headshot", batchSize: number) => {
    let prompt = "";
    let useCurrentAsRef = false;
    if (type === "full_body") { prompt = fullBodyPrompt; useCurrentAsRef = useCurrentAsRefFullBody; }
    else if (type === "three_view") { prompt = threeViewPrompt; useCurrentAsRef = useCurrentAsRefThreeView; }
    else if (type === "headshot") { prompt = headshotPrompt; useCurrentAsRef = useCurrentAsRefHeadshot; }
    onGenerate(type, prompt, applyStyle, negativePrompt, batchSize, useCurrentAsRef);
};
```

**onGenerate prop 类型更新：**
```tsx
onGenerate: (type: string, prompt: string, applyStyle: boolean, negativePrompt: string, batchSize: number, useCurrentAsRef?: boolean) => void;
```

**checkbox UI（每个 tab 的生成按钮区域，有图时显示）：**
```tsx
{hasImage && (
  <label className="flex items-center gap-2 text-sm text-text-secondary cursor-pointer select-none">
    <input
      type="checkbox"
      checked={useCurrentAsRefXxx}
      onChange={(e) => setUseCurrentAsRefXxx(e.target.checked)}
      className="rounded border-gray-600 bg-gray-700 text-primary focus:ring-primary"
    />
    Use current image as reference
  </label>
)}
```

`hasImage` 判断：对应 asset 的 `selected_id` 非空且 variants 非空。

### ConsistencyVault.tsx — handleGenerate

函数签名加 `useCurrentAsRef: boolean = false`，传给 `api.generateAsset`。

scene/prop 的 modal（`CharacterDetailModal`）加同样的 checkbox，`onGenerate` prop 透传 `useCurrentAsRef`。

---

## 兼容性

- `use_current_as_ref` 默认 `False`，不影响现有调用
- `Scene.prompt` / `Prop.prompt` 默认 `None`，Pydantic 向后兼容
- `generate_series_asset` 路径（`_process_series_asset_task`）的 scene/prop 调用补入 `prompt` 透传（series 路径已有 `prompt` 变量）；`use_current_as_ref` 暂不透传到 series 路径
