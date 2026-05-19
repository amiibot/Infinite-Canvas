# Design：异步任务系统

## 架构

```
前端 POST /generate_assets
        ↓ 立即返回 {_task_id}
后端 asyncio.create_task(bg_generate_assets(task_id, ...))
        ↓ 后台执行，每完成一张调用 update_task(progress=n/total)
前端 setInterval → GET /api/tasks/{task_id}
        ↓ status=completed → hydrateProject(result.project)
        ↓ status=failed    → setStatus(error)
```

## 后端设计

### 全局任务表

```python
_ASYNC_TASKS: dict[str, dict] = {}

def _create_task(task_id: str, **meta) -> dict:
    task = {"task_id": task_id, "status": "pending", "progress": 0, "total": 0,
            "result": None, "error": None, **meta}
    _ASYNC_TASKS[task_id] = task
    return task

def _update_task(task_id: str, **kwargs):
    if task_id in _ASYNC_TASKS:
        _ASYNC_TASKS[task_id].update(kwargs)
```

### 轮询接口

```
GET /api/tasks/{task_id}
→ 200: {task_id, status, progress, total, result?, error?}
→ 404: task not found
```

`result` 字段在 completed 时包含 `{"project": <project_dict>}`，前端直接 `hydrateProject(task.result.project)`。

### 接口改造模式（3 个接口统一）

```python
@app.post("/api/comic/projects/{pid}/generate_assets")
async def generate_comic_assets(pid: str, body: ...):
    task_id = str(uuid.uuid4())
    _create_task(task_id)
    asyncio.create_task(_bg_generate_assets(task_id, pid, body))
    return {"_task_id": task_id}
```

后台函数 `_bg_generate_assets` 是原函数逻辑的提取，完成后写入 `result`，失败写入 `error`。

### progress 计算

- `generate_assets`：total = len(chars) + len(scenes) + len(props)，每完成一张 +1
- `regenerate_asset`：total = batch_size，每完成一张 +1
- `render_comic_frame`：total = 1，完成后 progress = 1

## 前端设计

### 通用轮询工具函数

```js
async function pollTask(taskId, { onProgress, onComplete, onError, intervalMs = 2000, timeoutMs = 300000 }) {
    const start = Date.now();
    return new Promise((resolve, reject) => {
        const timer = setInterval(async () => {
            if (Date.now() - start > timeoutMs) {
                clearInterval(timer);
                onError('生成超时');
                return;
            }
            try {
                const task = await api(`/api/tasks/${taskId}`);
                if (task.status === 'completed') {
                    clearInterval(timer);
                    onComplete(task.result);
                    resolve(task.result);
                } else if (task.status === 'failed') {
                    clearInterval(timer);
                    onError(task.error || '生成失败');
                    reject(new Error(task.error));
                } else {
                    onProgress?.(task.progress, task.total);
                }
            } catch (e) {
                clearInterval(timer);
                onError(e.message);
                reject(e);
            }
        }, intervalMs);
    });
}
```

### 各调用点改造

**handleGenerateAssets**：
```js
const { _task_id } = await api('.../generate_assets', { method: 'POST', body: ... });
await pollTask(_task_id, {
    onProgress: (n, t) => setStatus(`生成中 ${n}/${t}…`),
    onComplete: (r) => { hydrateProject(r.project); setStatus('资产生成完成。', 'success'); render(); },
    onError: (e) => setStatus(e, 'error'),
});
```

**handleRegenAsset**：同上，onComplete 后额外刷新弹窗图片和变体条。

**renderFrame**：同上，onComplete 后调用 `renderStep4()`。

## 取消机制

后端在 `_ASYNC_TASKS` 中额外存储 asyncio.Task 引用：

```python
_create_task(task_id, ...)
_ASYNC_TASKS[task_id]["_asyncio_task"] = asyncio.create_task(_bg_fn(...))
```

新增取消接口：
```
POST /api/tasks/{task_id}/cancel
→ 204: 取消请求已发送
→ 404: task not found
→ 409: task already completed/failed
```

后台函数顶层 catch `asyncio.CancelledError`，写入 `status='cancelled'`：
```python
async def _bg_generate_assets(task_id, ...):
    try:
        ...
    except asyncio.CancelledError:
        _update_task(task_id, status='cancelled')
    except Exception as e:
        _update_task(task_id, status='failed', error=str(e))
```

前端轮询收到 `status='cancelled'` 时停止轮询，恢复按钮状态，不显示错误。

前端取消按钮：
- `handleGenerateAssets`：Step3 生成中显示"取消"按钮，点击 `POST /api/tasks/{task_id}/cancel`
- `handleRegenAsset`：弹窗重生成按钮变为"取消"，点击同上
- `renderFrame`：渲染按钮变为"取消"，点击同上

## 兼容性

- 后端接口返回结构从 `project` 变为 `{_task_id}`，前端调用点全部同步改造，无向后兼容需求
- `_ASYNC_TASKS` 内存存储，重启丢失，前端超时后显示错误即可
- `asyncio.create_task` 需在 async 上下文中调用，FastAPI 路由函数本身是 async，满足条件
