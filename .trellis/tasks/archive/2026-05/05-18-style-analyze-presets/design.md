# Design：Step2 AI 风格推荐 + 用户自定义预设

## 架构

```
前端 Step2
  ├── 内置预设卡片（8个，硬编码）
  ├── 自定义预设卡片（localStorage，带 ✕ 删除）
  ├── AI 推荐折叠区（session 内存，state.aiRecommendations）
  │     └── 3 张推荐卡片（点击逻辑与内置预设相同）
  ├── style_prompt / negative_prompt textarea
  └── 按钮行：[AI 推荐] [保存为预设] [应用风格]

后端
  └── POST /api/comic/projects/{pid}/art_direction/analyze
        → 读 project["original_text"]
        → call_chat_completion(system_prompt, user_prompt)
        → 返回 {recommendations: [{name, description, reason, positive_prompt, negative_prompt}]}
```

## 后端设计

### 接口

```
POST /api/comic/projects/{pid}/art_direction/analyze
→ 200: {"recommendations": [...]}
→ 400: 剧本内容为空
→ 404: 项目不存在
```

### LLM System Prompt（参考 luna-studio llm.py:527）

```
你是专业的电影美术指导。根据剧本内容推荐3种视觉风格。
每种风格包含：name（英文）、description（中文1-2句）、reason（中文≤50字）、
positive_prompt（英文SD提示词，只描述光影/色调/材质/氛围，≤50词）、
negative_prompt（英文，≤30词）。
返回严格 JSON：{"recommendations": [...]}
提示词只描述风格，不描述具体人物/场景/物品。
```

### 请求体

无需 body，直接从 project 读 `original_text`（截取前 2000 字符）。

## 前端设计

### state 新增字段

```js
aiRecommendations: [],      // [{name, description, reason, positive_prompt, negative_prompt}]
analyzingStyle: false,      // AI 推荐 loading 状态
```

### localStorage 结构

```js
// key: "luna_custom_presets"
[{id: "custom_<timestamp>", name: "我的风格", prompt: "...", negative: "..."}]
// 最多 10 条，超出时删除最旧的
```

### 预设 ID 规则

- 内置预设：`id` 为固定字符串（`ghibli` / `pixar` 等）
- 自定义预设：`id` 为 `custom_<timestamp>`
- AI 推荐：`id` 为 `ai_<index>`（0/1/2），session 内有效

### renderStep2 改动

1. 预设网格后追加自定义预设卡片（从 localStorage 读取）
2. 自定义预设卡片多一个 `data-del-preset` 属性，显示 ✕ 按钮
3. 若 `state.aiRecommendations.length > 0`，在自定义预设后追加 AI 推荐折叠区
4. 按钮行新增"AI 推荐"和"保存为预设"按钮

### 事件处理（复用现有 contentPanel click 委托）

- `[data-preset]` 点击：已有逻辑，AI 推荐卡片同样使用 `data-preset="ai_0"` 等
- `[data-del-preset]` 点击：从 localStorage 删除对应 id，`render()`
- `#analyzeStyleBtn` 点击：调用 `handleAnalyzeStyle()`
- `#savePresetBtn` 点击：调用 `handleSavePreset()`（inline prompt 输入名称）

### handleAnalyzeStyle()

```js
async function handleAnalyzeStyle() {
    state.analyzingStyle = true; render();
    try {
        const { recommendations } = await api(`/api/comic/projects/${projectId}/art_direction/analyze`, { method: 'POST' });
        state.aiRecommendations = recommendations.map((r, i) => ({
            id: `ai_${i}`, name: r.name, description: r.description,
            prompt: r.positive_prompt, negative: r.negative_prompt, reason: r.reason
        }));
    } catch(e) { setStatus(e.message, 'error'); }
    finally { state.analyzingStyle = false; render(); }
}
```

### handleSavePreset()

```js
function handleSavePreset() {
    const name = prompt(tr('预设名称', 'Preset name'));
    if (!name?.trim()) return;
    const stored = JSON.parse(localStorage.getItem('luna_custom_presets') || '[]');
    stored.push({ id: `custom_${Date.now()}`, name: name.trim(),
        prompt: state.customStylePrompt, negative: state.customNegativePrompt });
    if (stored.length > 10) stored.splice(0, stored.length - 10);
    localStorage.setItem('luna_custom_presets', JSON.stringify(stored));
    render();
}
```

## 兼容性

- `state.presetId` 已有，AI 推荐卡片选中后 `presetId = "ai_0"` 等，`handleApplyStyle` 中 `presets.find` 找不到时 fallback 到 `state.customStylePrompt`，行为正确无需改动
- 自定义预设选中后同样走 `state.customStylePrompt = preset.prompt`，与内置预设一致
