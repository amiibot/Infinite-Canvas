# Implement：Step2 AI 风格推荐 + 用户自定义预设

## 实施顺序

### 后端（main.py）

- [ ] 1. 在 `POST /api/comic/projects/{pid}/art_direction`（约 3381 行）附近新增接口：
  ```python
  @app.post("/api/comic/projects/{pid}/art_direction/analyze")
  async def analyze_art_direction(pid: str):
  ```
  - 读取 project["original_text"]，为空返回 400
  - 构造 system_prompt + user_prompt，调用 `call_chat_completion`
  - 解析 JSON，返回 `{"recommendations": [...]}`
  - 顶层 try/except，失败返回 500

### 前端（static/project.html）

- [ ] 2. `state` 初始化（约 1161 行）新增：
  ```js
  aiRecommendations: [],
  analyzingStyle: false,
  ```

- [ ] 3. `renderStep2`（约 1897 行）改造：
  - 预设网格后追加自定义预设（从 localStorage 读取，带 `data-del-preset`）
  - 若 `state.aiRecommendations.length > 0`，追加 AI 推荐折叠区（标题 + 3 张卡片）
  - 按钮行新增 `#analyzeStyleBtn`（AI 推荐）和 `#savePresetBtn`（保存为预设）

- [ ] 4. 新增 `handleAnalyzeStyle()` 函数（在 `handleApplyStyle` 附近）

- [ ] 5. 新增 `handleSavePreset()` 函数

- [ ] 6. contentPanel click 委托（约 2781 行）新增：
  - `[data-del-preset]` 点击处理
  - `#analyzeStyleBtn` 点击 → `handleAnalyzeStyle()`
  - `#savePresetBtn` 点击 → `handleSavePreset()`

## 验证命令

```bash
# 语法检查
python3 -m py_compile main.py && echo "OK"

# 接口存在性
curl -s -X POST http://localhost:3000/api/comic/projects/TEST/art_direction/analyze | python3 -m json.tool
# 应返回 404（项目不存在）或 400（剧本为空）

# 有剧本的项目
curl -s -X POST http://localhost:3000/api/comic/projects/<real_pid>/art_direction/analyze | python3 -m json.tool
# 应返回 {"recommendations": [...]} 含 3 条
```

## 风险点

- `call_chat_completion` 返回的 JSON 可能包含 markdown 代码块，需 strip 后再 `json.loads`
- AI 推荐卡片 `id` 为 `ai_0/1/2`，`handleApplyStyle` 中 `presets.find` 找不到时走 `state.customStylePrompt` fallback，行为正确
- `prompt()` 在某些浏览器环境下被屏蔽，可接受（降级为无法保存，不崩溃）
