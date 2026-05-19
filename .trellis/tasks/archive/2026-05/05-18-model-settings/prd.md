# PRD: per-project 模型配置（model-settings）

## 目标

每个项目可独立配置文生图模型（image_model）和各资产类型宽高比（aspect_ratio），配置持久化到项目数据，刷新后不丢失。

## 用户价值

- 不同项目可以用不同模型（如一个项目用写实模型，另一个用动漫模型）
- aspect_ratio 设置不再因刷新丢失，减少重复操作

## 现状（代码查证）

- `aspectRatio` 只存在前端 `state`，页面刷新即丢失，从未写入项目数据
- 模型硬编码为全局 provider 的 `image_models[0]`，三处生成接口（L3442/L3562/L4088）均如此
- `art_direction` 已持久化到项目，但只有 `style_prompt` / `style_negative_prompt` / `style_name`

## 设计决策

- **model + aspect_ratio 合并存入 `art_direction`**，不新建独立接口（Step2 统一配置）
- **image_models 候选列表**：`loadProject` 时并行调用 `GET /api/config`，存入 `state.imageModels`
- **本次只做 t2i（文生图）模型**，i2i/i2v 作为可选扩展后续再加

## 需求

1. `art_direction` 新增字段：`image_model`（字符串）、`aspect_ratio`（对象，含 character/scene/prop）
2. Step2 新增模型下拉选择（从 `state.imageModels` 取候选列表）
3. Step2 的 aspect_ratio 选择保存时一并写入 `art_direction`
4. 三处生成接口优先读取 `art_direction.image_model`，为空时 fallback 到全局 `image_models[0]`
5. `hydrateProject` 从 `art_direction` 恢复 `state.aspectRatio` 和 `state.selectedImageModel`

## 验收标准

- [ ] Step2 显示模型下拉，候选来自全局 `image_models` 列表
- [ ] 点击"应用风格"后，`image_model` 和 `aspect_ratio` 持久化到项目
- [ ] 刷新页面后，模型选择和 aspect_ratio 恢复正确
- [ ] 生成资产时使用项目配置的 `image_model`（若已设置）
- [ ] 未设置时 fallback 到全局默认，行为与现在一致

## 超出范围

- i2i_model / i2v_model 配置
- 模型手动输入（只支持下拉选择已有列表）
- Series 级别模型配置
