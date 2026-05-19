# Implement: luna-studio Step3 资产生成补齐 A1/A2/B1

## 执行顺序

后端先行（无前端依赖），前端最后。

---

## 后端

### Step 1 — models.py：加 Scene.prompt / Prop.prompt 字段

文件：`other/luna-studio/src/apps/comic_gen/models.py`

- `Scene`（~line 188）：在 `image_asset` 字段后加 `prompt: Optional[str] = Field(None, description="Prompt used for last generation (without style suffix)")`
- `Prop`（~line 206）：同上

### Step 2 — assets.py：generate_scene 重构（A1 + A2 + prompt 参数）

文件：`other/luna-studio/src/apps/comic_gen/assets.py`，~line 386

1. 函数签名加 `prompt: str = None` 和 `use_current_as_ref: bool = False`
2. 内部逻辑：用户传入 `prompt` 时用作 `base_prompt`，否则自动构造
3. 写回 `scene.prompt = base_prompt`（A1）
4. 构造 `generation_prompt` 时加去重判断（A2）
5. B1 取 ref 逻辑：`use_current_as_ref=True` 时取 `scene.image_asset.selected_id` 对应 variant 的本地路径

### Step 3 — assets.py：generate_prop 重构（A1 + A2 + prompt 参数）

同 Step 2，~line 434。自动构造 base_prompt 为 `"Prop Design: {prop.name}. {prop.description}. Isolated on white background, high quality, detailed."`

### Step 4 — assets.py：generate_character 加 use_current_as_ref 参数（B1 后端）

文件：`other/luna-studio/src/apps/comic_gen/assets.py`

- 函数签名加 `use_current_as_ref: bool = False`
- 在现有 ref_image_path 逻辑前插入 B1 取 ref 逻辑
- 按 generation_type 选对应 asset：
  - `full_body` → `character.full_body_asset`
  - `three_view` → `character.three_view_asset`
  - `headshot` → `character.headshot_asset`

### Step 5 — pipeline.py：透传 prompt + use_current_as_ref（中间层）

文件：`other/luna-studio/src/apps/comic_gen/pipeline.py`

- `generate_asset`（~line 185）：签名加 `use_current_as_ref: bool = False`；调用 `generate_scene`/`generate_prop` 时加 `prompt=prompt, use_current_as_ref=use_current_as_ref`；调用 `generate_character` 时加 `use_current_as_ref=use_current_as_ref`
- `create_asset_generation_task`（~line 295）：签名加 `use_current_as_ref: bool = False`，存入 `task["params"]["use_current_as_ref"]`
- `process_asset_generation_task`（~line 355）：从 params 取出 `use_current_as_ref`，传给 `generate_asset`
- `_process_series_asset_task`（~line 421）：scene/prop 调用处加 `prompt=prompt`（series 路径已有 `prompt` 变量）；`use_current_as_ref` 不透传

### Step 6 — api.py：GenerateAssetRequest 加字段，端点透传（B1 入口）

文件：`other/luna-studio/src/apps/comic_gen/api.py`

- `GenerateAssetRequest`（~line 103）加 `use_current_as_ref: bool = False`
- `generate_series_asset` 端点（~line 454）调用 `pipeline.create_asset_generation_task` 时透传 `use_current_as_ref=request.use_current_as_ref`
- 找到 project 的 generate asset 端点（`/projects/{script_id}/assets/generate`），同样透传

---

## 前端

### Step 7 — api.ts：generateAsset 加参数

文件：`other/luna-studio/frontend/src/lib/api.ts`，~line 210

- 函数签名末尾加 `useCurrentAsRef: boolean = false`
- 请求体加 `use_current_as_ref: useCurrentAsRef`

### Step 8 — CharacterWorkbench.tsx：B1 checkbox UI

文件：`other/luna-studio/frontend/src/components/modules/CharacterWorkbench.tsx`

- 加三个独立 state：`useCurrentAsRefFullBody`、`useCurrentAsRefThreeView`、`useCurrentAsRefHeadshot`
- `handleGenerateClick`（~line 246）按 type 选对应 state，传给 `onGenerate`
- `CharacterWorkbenchProps.onGenerate` 类型加 `useCurrentAsRef?: boolean`
- 每个 tab 的生成按钮区域加 checkbox（有图时显示，`hasImage` = 对应 asset 的 `selected_id` 非空）

### Step 9 — ConsistencyVault.tsx：B1 checkbox UI + handleGenerate 透传

文件：`other/luna-studio/frontend/src/components/modules/ConsistencyVault.tsx`

- `handleGenerate`（~line 72）签名加 `useCurrentAsRef: boolean = false`，传给 `api.generateAsset`
- `onGenerate` 调用处（~line 459）透传 `useCurrentAsRef`
- scene/prop 的 `CharacterDetailModal` 加 checkbox state + UI（有图时显示）

---

## 验证命令

```bash
cd /home/ami/project/luna-canvas/other/luna-studio

# 后端语法检查
python -c "from src.apps.comic_gen.models import Scene, Prop; s=Scene(id='x',name='x',description='x'); print('Scene.prompt:', s.prompt); p=Prop(id='y',name='y',description='y'); print('Prop.prompt:', p.prompt)"
python -c "from src.apps.comic_gen.assets import AssetGenerator; print('assets ok')"
python -c "from src.apps.comic_gen.pipeline import Pipeline; print('pipeline ok')"
python -c "from src.apps.comic_gen.api import app; print('api ok')"

# 前端类型检查
cd frontend && npx tsc --noEmit 2>&1 | head -30
```

---

## 风险点

- `_process_series_asset_task` 的 scene/prop 路径补入 `prompt` 透传，但 `use_current_as_ref` 不透传（series 路径无此需求，默认 False 行为不变）
- `generate_character` 的 B1 ref 逻辑需在现有 `ref_image_path` 赋值逻辑之前插入，确保优先级正确（用户显式勾选 > 自动 full_body ref）
