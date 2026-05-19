# PRD：Step2 AI 风格推荐 + 用户自定义预设

## Goal

提升 Step2 风格方向的易用性：
1. **AI 推荐**：一键让 LLM 根据剧本内容推荐 3 种视觉风格，用户可直接选用
2. **用户自定义预设**：将当前手动填写的 style_prompt 保存为命名预设，下次可复用

## 确认事实（代码已验证）

- 剧本内容存储在 `project["original_text"]`（`main.py:3151`）
- `call_chat_completion` 已有（`main.py:1011`），可直接复用
- 8 个内置预设已在前端 `project.html:1186` 硬编码，无需后端接口
- luna-studio 参考实现：`llm.py:518 analyze_script_for_styles`，system prompt 可直接参考
- `localStorage` 在 project.html 中未使用，可用于存储用户自定义预设
- Step2 当前 UI：预设卡片网格 + style_prompt textarea + negative_prompt textarea + "应用风格"按钮

## Requirements

### 后端

1. 新增 `POST /api/comic/projects/{pid}/art_direction/analyze`
   - 读取 `project["original_text"]` 作为剧本内容
   - 调用 `call_chat_completion` 推荐 3 种风格
   - 返回 `{"recommendations": [{name, description, reason, positive_prompt, negative_prompt}]}`
   - 剧本为空时返回 400

### 前端

2. Step2 新增"AI 推荐"按钮（在"应用风格"按钮旁）
   - 点击后调用接口，显示 loading 状态
   - 返回 3 张推荐卡片，显示在内置预设下方的独立折叠区（默认展开），样式与内置预设卡片一致
   - 内置预设始终可见，推荐卡片与内置卡片共用同一选中逻辑
   - 推荐结果仅本次会话有效（不持久化），刷新后折叠区消失

3. Step2 新增"保存为预设"按钮
   - 点击后弹出输入框填写预设名称
   - 将当前 style_prompt + negative_prompt 保存到 `localStorage`（key: `luna_custom_presets`）
   - 自定义预设显示在内置预设卡片之后，带删除按钮（✕）
   - 最多保存 10 个自定义预设

## Acceptance Criteria

- [ ] `POST /api/comic/projects/{pid}/art_direction/analyze` 存在，返回 3 条推荐
- [ ] 剧本为空时接口返回 400
- [ ] Step2 有"AI 推荐"按钮，点击后显示推荐卡片
- [ ] 推荐卡片可点击选中，填充 style_prompt / negative_prompt
- [ ] Step2 有"保存为预设"按钮，保存后立即显示在预设列表
- [ ] 自定义预设有删除按钮，删除后从 localStorage 移除
- [ ] 刷新页面后自定义预设仍存在

## Out of Scope

- 自定义预设同步到后端/跨设备
- 推荐结果持久化
- 预设排序/编辑
