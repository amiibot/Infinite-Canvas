# Implement Plan — sync_descriptions

## 文件变更清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `main.py` | Edit | 新增 Pydantic model、后台任务函数、路由 |
| `static/project.html` | Edit | renderStep1 增加按钮、新增 state 字段和 handler |

## 后端实现步骤

### Step B1：新增 Pydantic Model

在 `main.py` 的 `ComicParseRequest` 附近（约第 640 行）插入：

```python
class ComicSyncDescriptionsRequest(BaseModel):
    force: bool = False
    provider_id: str = ""
    model: str = ""
```

### Step B2：新增后台任务函数

在 `_bg_generate_assets` 函数之前（约第 3439 行）插入 `_bg_sync_descriptions`。

函数结构（分块写入，约 80 行）：

**块 1（约 40 行）**：函数头 + 实体收集 + LLM 调用循环（characters + scenes）

```python
async def _bg_sync_descriptions(task_id: str, pid: str, body):
    try:
        _update_task(task_id, status="running")
        _, project_snapshot = get_comic_project_or_404(pid)
        original_text = (project_snapshot.get("original_text") or "")[:500]
        provider_id = body.provider_id or get_primary_provider_id(load_api_providers())

        def needs_update(item):
            return body.force or not (item.get("description") or "").strip()

        chars_todo = [c for c in project_snapshot.get("characters", []) if needs_update(c)]
        scenes_todo = [s for s in project_snapshot.get("scenes", []) if needs_update(s)]
        props_todo  = [p for p in project_snapshot.get("props", [])   if needs_update(p)]
        total = len(chars_todo) + len(scenes_todo) + len(props_todo)
        _update_task(task_id, total=total)

        failures = []
        desc_updates = {}  # {id: new_description}
        done = 0

        for char in chars_todo:
            name = char.get("name", "")
            age = char.get("age", "")
            gender = char.get("gender", "")
            clothing = char.get("clothing", "")
            try:
                result = await call_chat_completion(
                    messages=[
                        {"role": "system", "content": "You are a visual novel character designer. Write concise visual descriptions."},
                        {"role": "user", "content": (
                            f"Character name: {name}\n"
                            f"Age: {age}, Gender: {gender}, Clothing: {clothing}\n"
                            f"Story context: {original_text}\n\n"
                            "Write a 1-2 sentence visual description for image generation. Return only the description."
                        )},
                    ],
                    provider_id=provider_id,
                    model=body.model,
                )
                desc_updates[char["id"]] = result.strip()
            except Exception as exc:
                failures.append(f"character '{name}': {exc}")
                print(f"sync_descriptions character failed: {exc}")
            done += 1
            _update_task(task_id, progress=done)
```

**块 2（约 40 行）**：scenes + props 循环 + 写回 + 任务完成

```python
        for scene in scenes_todo:
            name = scene.get("name", "")
            try:
                result = await call_chat_completion(
                    messages=[
                        {"role": "system", "content": "You are a visual novel background artist. Write concise scene descriptions."},
                        {"role": "user", "content": (
                            f"Scene name: {name}\n"
                            f"Story context: {original_text}\n\n"
                            "Write a 1-2 sentence visual description for image generation. Return only the description."
                        )},
                    ],
                    provider_id=provider_id,
                    model=body.model,
                )
                desc_updates[scene["id"]] = result.strip()
            except Exception as exc:
                failures.append(f"scene '{name}': {exc}")
                print(f"sync_descriptions scene failed: {exc}")
            done += 1
            _update_task(task_id, progress=done)

        for prop in props_todo:
            name = prop.get("name", "")
            try:
                result = await call_chat_completion(
                    messages=[
                        {"role": "system", "content": "You are a visual novel prop designer. Write concise prop descriptions."},
                        {"role": "user", "content": (
                            f"Prop name: {name}\n"
                            f"Story context: {original_text}\n\n"
                            "Write a 1-2 sentence visual description for image generation. Return only the description."
                        )},
                    ],
                    provider_id=provider_id,
                    model=body.model,
                )
                desc_updates[prop["id"]] = result.strip()
            except Exception as exc:
                failures.append(f"prop '{name}': {exc}")
                print(f"sync_descriptions prop failed: {exc}")
            done += 1
            _update_task(task_id, progress=done)

        if total > 0 and len(failures) == total:
            _update_task(task_id, status="failed", error=f"All descriptions failed: {failures[0]}")
            return

        result_holder = {}
        def _write_back(projects):
            project = next((p for p in projects if p["id"] == pid), None)
            if not project:
                return projects
            for collection in ("characters", "scenes", "props"):
                for item in project.get(collection, []):
                    if item["id"] in desc_updates:
                        item["description"] = desc_updates[item["id"]]
            result_holder["project"] = project
            return projects
        atomic_update_comic_projects(_write_back)
        _update_task(task_id, status="completed",
                     result={"failures": failures, "updated": len(desc_updates)})

    except asyncio.CancelledError:
        _update_task(task_id, status="cancelled")
        raise
    except Exception as exc:
        _update_task(task_id, status="failed", error=str(exc))
        print(f"_bg_sync_descriptions error: {exc}")
```

### Step B3：新增路由

在 `parse_comic_project` 路由之后（约第 3282 行）插入：

```python
@app.post("/api/comic/projects/{pid}/sync_descriptions")
async def sync_comic_descriptions(pid: str, body: ComicSyncDescriptionsRequest = None):
    get_comic_project_or_404(pid)  # validate early
    if body is None:
        body = ComicSyncDescriptionsRequest()
    task_id = str(uuid.uuid4())
    _create_task(task_id)
    at = asyncio.create_task(_bg_sync_descriptions(task_id, pid, body))
    _update_task(task_id, _asyncio_task=at)
    return {"_task_id": task_id}
```

## 前端实现步骤

### Step F1：新增 state 字段

在 `state` 对象初始化处（找 `parsing: false` 附近）新增：

```js
syncingDesc: false,
syncDescTaskId: null,
```

### Step F2：修改 renderStep1 按钮区

将现有按钮行从：
```html
<button id="parseBtn" ...>
```
改为包含两个按钮：

```html
<div style="display:flex;gap:8px;align-items:center;">
  ${hasEntities ? `
    <button id="syncDescBtn" class="ghost-btn" type="button"
      ${state.syncingDesc ? 'disabled' : ''}>
      ${state.syncingDesc ? '<span class="spinner"></span>' : ''}
      ${tr('同步描述', 'Sync Descriptions')}
    </button>
  ` : ''}
  <button id="parseBtn" class="primary-btn" type="button"
    ${state.parsing ? 'disabled' : ''}>
    ${state.parsing ? '<span class="spinner"></span>' : ''}
    ${tr('AI 解析', 'AI Parse')}
  </button>
</div>
```

其中 `hasEntities` 计算：
```js
const project = state.project || { characters: [], scenes: [], props: [] };
const hasEntities = (project.characters.length + project.scenes.length + project.props.length) > 0;
```

### Step F3：新增 handleSyncDescriptions 函数

在 `handleParse` 函数之后插入（约 40 行）：

```js
async function handleSyncDescriptions() {
    state.syncingDesc = true;
    setStatus(tr('同步描述中…', 'Syncing descriptions…'));
    render();
    try {
        const resp = await api(`/api/comic/projects/${projectId}/sync_descriptions`, {
            method: 'POST',
            body: JSON.stringify({ force: false })
        });
        const taskId = resp._task_id;
        state.syncDescTaskId = taskId;
        // 轮询
        while (true) {
            await new Promise(r => setTimeout(r, 2000));
            const task = await api(`/api/tasks/${taskId}`);
            if (task.status === 'completed') {
                const project = await api(`/api/comic/projects/${projectId}`);
                hydrateProject(project);
                const failures = task.result?.failures || [];
                if (failures.length > 0) {
                    setStatus(tr(`描述同步完成，${failures.length} 项失败。`, `Sync done, ${failures.length} failed.`), 'warning');
                } else {
                    setStatus(tr('描述同步完成。', 'Descriptions synced.'), 'success');
                }
                break;
            }
            if (task.status === 'failed') {
                setStatus(task.error || tr('同步失败', 'Sync failed'), 'error');
                break;
            }
            if (task.status === 'cancelled') {
                setStatus(tr('同步已取消', 'Sync cancelled'), '');
                break;
            }
        }
    } catch (error) {
        setStatus(error.message || tr('同步失败', 'Sync failed'), 'error');
    } finally {
        state.syncingDesc = false;
        state.syncDescTaskId = null;
        render();
    }
}
```

### Step F4：事件委托

在 `if (event.target.id === 'parseBtn')` 块之后插入：

```js
if (event.target.id === 'syncDescBtn') {
    handleSyncDescriptions();
}
```

## 验证步骤

```bash
# 1. Python 语法检查
python3 -m py_compile /home/ami/project/luna-canvas/main.py && echo "OK"

# 2. JS 语法检查
python3 -c "
import re; html = open('/home/ami/project/luna-canvas/static/project.html').read()
scripts = re.findall(r'<script[^>]*>(.*?)</script>', html, re.DOTALL)
open('/tmp/check_js.js', 'w').write('\n'.join(scripts))
" && node --check /tmp/check_js.js && echo "JS OK"

# 3. 手动测试
# - 创建项目，解析剧本，确认 characters/scenes 有数据
# - 点击"同步描述"，观察 spinner 和进度
# - 检查侧边栏 description 是否更新
# - 测试 force=true 覆盖已有 description
```

## 注意事项

- `_bg_sync_descriptions` 中 `total=0` 时（所有实体已有描述且 force=false），
  直接 `_update_task(task_id, status="completed", result={"updated": 0, "failures": []})` 返回，
  不进入循环。
- `call_chat_completion` 的 `model=""` 时，`resolve_chat_provider` 会使用 provider 默认模型，
  需确认该函数对空 model 的处理（已有代码支持）。
- 写回时不使用 `get_comic_project_or_404`（会重新加载文件），直接在 `atomic_update_comic_projects`
  回调中操作，保证原子性。
