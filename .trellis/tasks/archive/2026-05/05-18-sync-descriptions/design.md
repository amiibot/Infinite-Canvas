# Design — sync_descriptions

## 接口设计

### Request

```
POST /api/comic/projects/{pid}/sync_descriptions
Content-Type: application/json

{
  "force": false,       // 是否覆盖已有 description
  "provider_id": "",    // 空字符串 = 使用主 provider
  "model": ""           // 空字符串 = 使用 provider 默认 chat model
}
```

### Response（立即返回）

```json
{ "_task_id": "uuid" }
```

### Pydantic Model

```python
class ComicSyncDescriptionsRequest(BaseModel):
    force: bool = False
    provider_id: str = ""
    model: str = ""
```

## 后台任务设计

### 函数签名

```python
async def _bg_sync_descriptions(task_id: str, pid: str, body: ComicSyncDescriptionsRequest):
```

### 执行流程

```
1. _update_task(task_id, status="running")
2. 读取项目快照（get_comic_project_or_404）
3. 收集需要处理的实体：
   - force=False → 过滤 description 为空/None 的
   - force=True  → 全部实体
4. total = len(chars_todo) + len(scenes_todo) + len(props_todo)
   _update_task(task_id, total=total)
5. 逐个调用 LLM，生成 description
6. 收集 {id: new_description} 映射
7. atomic_update_comic_projects 写回（只更新 description 字段）
8. _update_task(task_id, status="completed", result={...})
```

### LLM Prompt 设计

**角色 (character)**：
```
System: You are a visual novel character designer. Write concise visual descriptions.
User: Character name: {name}
Age: {age}, Gender: {gender}, Clothing: {clothing}
Original story context: {original_text[:500]}

Write a 1-2 sentence visual description for this character suitable for image generation.
Return only the description text, no extra formatting.
```

**场景 (scene)**：
```
System: You are a visual novel background artist. Write concise scene descriptions.
User: Scene name: {name}
Original story context: {original_text[:500]}

Write a 1-2 sentence visual description for this scene suitable for image generation.
Return only the description text, no extra formatting.
```

**道具 (prop)**：
```
System: You are a visual novel prop designer. Write concise prop descriptions.
User: Prop name: {name}
Original story context: {original_text[:500]}

Write a 1-2 sentence visual description for this prop suitable for image generation.
Return only the description text, no extra formatting.
```

### 写回策略

使用 `atomic_update_comic_projects`，在回调函数中：
- 只更新 `item["description"]`，不触碰其他字段
- 用 id 匹配，避免顺序依赖

### 错误处理

- 单个实体失败：`failures.append(f"{kind} '{name}': {exc}")`，继续处理下一个
- 全部失败：`_update_task(task_id, status="failed", error=failures[0])`
- 部分失败：`_update_task(task_id, status="completed", result={"failures": failures})`
- `asyncio.CancelledError`：re-raise（让框架处理取消）
- 顶层 `except Exception`：`_update_task(task_id, status="failed", error=str(exc))`

## 前端设计

### 按钮位置

在 `renderStep1()` 的按钮行（`display:flex` 容器）中，parseBtn 左侧或右侧增加：

```html
<button id="syncDescBtn" class="ghost-btn" type="button"
  ${state.syncingDesc ? 'disabled' : ''}
  ${hasEntities ? '' : 'style="display:none"'}>
  ${state.syncingDesc ? '<span class="spinner"></span>' : ''}
  ${tr('同步描述', 'Sync Descriptions')}
</button>
```

显示条件：`project.characters.length + project.scenes.length + project.props.length > 0`

### 状态字段（新增到 state）

```js
syncingDesc: false,
syncDescTaskId: null,
```

### handleSyncDescriptions 函数

```
1. state.syncingDesc = true; render()
2. POST /api/comic/projects/{pid}/sync_descriptions
3. 拿到 _task_id，存入 state.syncDescTaskId
4. 轮询 /api/tasks/{task_id}（每 2s 一次）
5. status === "completed" → 刷新项目数据，setStatus 成功提示
6. status === "failed" → setStatus 错误提示
7. finally: state.syncingDesc = false; render()
```

### 事件委托

在现有 `document.addEventListener('click', ...)` 中增加：

```js
if (event.target.id === 'syncDescBtn') {
    handleSyncDescriptions();
}
```

## 数据流

```
前端 POST sync_descriptions
  → 后端立即返回 _task_id
  → 前端轮询 GET /api/tasks/{task_id}
  → 后台 _bg_sync_descriptions 逐个调用 LLM
  → atomic_update_comic_projects 写回 description
  → task.status = "completed"
  → 前端 GET /api/comic/projects/{pid} 刷新数据
  → renderSidePanel() 更新侧边栏
```

## 不需要新增的内容

- 不需要新的数据迁移（description 字段已存在于所有实体）
- 不需要新的 API provider 逻辑（复用 `get_primary_provider_id`）
- 不需要新的轮询基础设施（复用现有 `/api/tasks/{task_id}` + 取消接口）
