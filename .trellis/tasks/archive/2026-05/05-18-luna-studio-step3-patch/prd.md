# PRD: luna-studio Step3 资产生成补齐 A1/A2/B1

## 目标

将 luna-canvas 已验证的 Step3 三项改进移植到 luna-studio，使两个项目的资产生成行为一致。

## 背景

通过代码对比（luna-canvas main.py vs luna-studio src/apps/comic_gen/），确认 luna-studio 缺失以下三点：

---

## 功能点

### A1 — 生成后 prompt 写回 item

**现状（luna-studio）**
- Character：`full_body_prompt`、`three_view_prompt`、`headshot_prompt` 已在 `assets.py` 写回 ✅
- Scene：`Scene` 模型无 `prompt` 字段，`generate_scene` 不写回 ❌
- Prop：`Prop` 模型无 `prompt` 字段，`generate_prop` 不写回 ❌

**目标**
- `models.py`：给 `Scene` 和 `Prop` 各加一个 `prompt: Optional[str] = None` 字段
- `assets.py`：`generate_scene` / `generate_prop` 加 `prompt: str = None` 参数；有用户传入时用用户 prompt 作 base，否则自动构造；生成前写回 `scene.prompt = base_prompt`（不含 style suffix）
- `pipeline.py`：`generate_asset` 调用 `generate_scene`/`generate_prop` 时透传 `prompt` 参数

**验收**：生成场景/道具后，重新打开弹窗，prompt 输入框显示上次生成的 prompt；用户自定义 prompt 优先于自动构造。

---

### A2 — style 去重追加（scene/prop）

**现状（luna-studio）**
- Character：`assets.py` 已有 `style_suffix not in base_prompt` 去重 ✅
- Scene：`assets.py:397` 直接拼接 `f"...{positive_prompt}"` 无去重 ❌
- Prop：`assets.py:445` 同上 ❌

**目标**
- `generate_scene`：构造 `base_prompt` 后，追加 style 时加去重判断：
  `generation_prompt = f"{base_prompt}, {positive_prompt}" if positive_prompt and positive_prompt not in base_prompt else base_prompt`
- `generate_prop`：同上

**验收**：当 style prompt 已包含在 base_prompt 中时，不重复追加。

---

### B1 — "参考当前图"复选框

**现状（luna-studio）**
- `GenerateAssetRequest`（`api.py:103`）无 `use_current_as_ref` 字段
- `pipeline.generate_asset`（`pipeline.py:185`）无此参数
- `assets.py` 的 `generate_character/scene/prop` 无此参数
- `api.ts:210` 的 `generateAsset` 函数无此参数
- `CharacterWorkbench.tsx` 和 `ConsistencyVault.tsx` 无对应 checkbox UI

**目标**
- `api.py`：`GenerateAssetRequest` 加 `use_current_as_ref: bool = False`
- `pipeline.py`：`generate_asset` 和 `create_asset_generation_task` 透传该参数
- `assets.py`：`generate_character`、`generate_scene`、`generate_prop` 接收 `use_current_as_ref`；当为 True 时，取当前 selected variant 的 URL 作为 `ref_image_path`（仅当有图时生效）
- `api.ts`：`generateAsset` 函数签名加 `useCurrentAsRef?: boolean`，请求体加 `use_current_as_ref`
- `CharacterWorkbench.tsx`：在每个 tab（full_body/three_view/headshot）的生成按钮区域各自加 checkbox，有图时显示，无图时隐藏；状态传入 `handleGenerateClick`
- `ConsistencyVault.tsx`：scene/prop 的 modal 区域同样加 checkbox

---

## 范围外

- B2/B3/C1（两个项目已等价，不需要改动）
- luna-canvas 本身不改动
- 前端构建/部署流程

## 验收标准

- [ ] 用户传入自定义 prompt 时，scene/prop 使用用户 prompt 作 base（不被自动模板覆盖）
- [ ] `Scene.prompt` 和 `Prop.prompt` 字段存在于 models.py
- [ ] 生成场景/道具后，API 返回的 project 数据中对应 item 有 `prompt` 字段
- [ ] scene/prop 的 style 不重复追加（positive_prompt already in base_prompt 时不追加）
- [ ] `GenerateAssetRequest` 包含 `use_current_as_ref: bool = False`
- [ ] 前端 checkbox 在有图时可见、无图时隐藏
- [ ] 勾选后重新生成，请求体中 `use_current_as_ref: true`，后端使用当前图作为 ref

## 已确认决策

- B1 checkbox：CharacterWorkbench 中 full_body/three_view/headshot **各自独立** state（`useCurrentAsRefFullBody`、`useCurrentAsRefThreeView`、`useCurrentAsRefHeadshot`），切换 tab 不互相影响。
- scene/prop 的 `generate_scene`/`generate_prop` 加 `prompt` 参数，支持用户自定义 prompt 覆盖自动模板。
