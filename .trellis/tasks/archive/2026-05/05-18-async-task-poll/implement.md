# Implement：异步任务系统

## 实施顺序

### 后端（main.py）

- [x] 1. 在全局变量区新增 `_ASYNC_TASKS: dict = {}`
- [x] 2. 新增 `_create_task(task_id, **meta)` 和 `_update_task(task_id, **kwargs)` 工具函数
- [x] 3. 新增 `GET /api/tasks/{task_id}` 轮询接口
- [x] 3b. 新增 `POST /api/tasks/{task_id}/cancel` 取消接口
- [x] 4. 提取 `generate_comic_assets` 逻辑为 `_bg_generate_assets(task_id, pid, body)`，接口改为立即返回 `{_task_id}`，后台函数 catch `CancelledError` 写 `cancelled`
- [x] 5. 提取 `regenerate_comic_asset` 逻辑为 `_bg_regenerate_asset(task_id, pid, body)`，同上
- [x] 6. 提取 `render_comic_frame` 逻辑为 `_bg_render_frame(task_id, pid, fid)`，同上

### 前端（static/project.html）

- [x] 7. 新增 `pollTask(taskId, opts)` 通用轮询工具函数，`status='cancelled'` 时静默停止
- [x] 8. 改造 `handleGenerateAssets()`：POST → 拿 `_task_id` → pollTask → onProgress 更新状态栏 → onComplete hydrateProject；生成中显示"取消"按钮
- [x] 9. 改造 `handleRegenAsset()`：同上，onComplete 额外刷新弹窗图片和变体条；资产卡片显示"取消"按钮
- [x] 10. 改造 `renderFrame(fid)`：同上，onComplete 调用 `renderStep4()`；渲染按钮变为"取消"

## 验证命令

```bash
# 启动服务
uv run --with-requirements requirements.txt python main.py &

# 验证轮询接口存在
curl -s http://localhost:3000/api/tasks/nonexistent | python3 -m json.tool

# 验证 generate_assets 立即返回
curl -s -X POST http://localhost:3000/api/comic/projects/TEST_PID/generate_assets \
  -H "Content-Type: application/json" -d '{}' | python3 -m json.tool
# 应看到 {"_task_id": "..."}，不阻塞
```

## 风险点

- `_bg_*` 函数内部的异常必须 catch 并写入 `_update_task(task_id, status='failed', error=str(e))`，否则前端永远轮询不到结果
- `asyncio.create_task` 创建的任务如果抛出未捕获异常，会在事件循环中静默失败，必须在 bg 函数顶层加 try/except
- `_ASYNC_TASKS` 无锁，asyncio 单线程模型下安全（不需要加锁）
