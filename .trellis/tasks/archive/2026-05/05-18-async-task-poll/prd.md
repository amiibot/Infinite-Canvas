# PRD：异步任务系统 — 生成接口后台化 + 前端轮询

## Goal

将 luna-canvas 中耗时的 AI 生成接口改为异步模式：接口立即返回 `_task_id`，前端通过轮询接口获取进度和结果，消除同步阻塞导致的超时和 UI 卡死问题。

## 确认事实（代码已验证）

- **目标接口（3 个）**：
  - `POST /api/comic/projects/{pid}/generate_assets`：串行生成所有角色/场景/道具，最慢（N 张图）
  - `POST /api/comic/projects/{pid}/regenerate_asset`：单资产批量生成，batch_size 1~4
  - `POST /api/comic/projects/{pid}/frames/{fid}/render`：单帧渲染
- **现有异步基础设施**：`BackgroundTasks` 未在 main.py 中使用；FastAPI 已引入，可直接用
- **现有轮询模式**：`video_tasks` 已有 `pollTimer + setInterval` 轮询模式（`project.html:2191`），可复用
- **luna-studio 参考**：`pipeline.asset_generation_tasks` 是内存 dict，`BackgroundTasks.add_task` 触发后台执行，`GET /tasks/{task_id}` 返回状态
- **任务状态字段**：`{task_id, status: pending|running|completed|failed, progress: 0~100, result, error}`

## Requirements

### 后端

1. 新增全局内存任务表 `_ASYNC_TASKS: dict[str, dict]`，存储任务状态
2. 新增工具函数：
   - `create_task(task_id, meta)` → 写入 pending 状态
   - `update_task(task_id, **kwargs)` → 更新状态/进度/结果
3. 新增轮询接口：`GET /api/tasks/{task_id}` → 返回任务状态；completed 时附带 `project` 字段
4. 改造 `generate_comic_assets`：立即返回 `{_task_id}`，后台执行生成逻辑，每完成一张更新 progress
5. 改造 `regenerate_comic_asset`：同上，batch_size 张图完成后 completed
6. 改造 `render_comic_frame`：同上，单张图完成后 completed
7. 任务完成/失败后保留在内存中（重启丢失可接受，不做 TTL 清理）

### 前端（project.html）

8. `handleGenerateAssets()`：POST 后拿到 `_task_id`，启动轮询，显示进度（"生成中 3/8..."）
9. 资产弹窗重新生成：同上，轮询完成后刷新弹窗图片
10. Step4 帧渲染按钮：同上，轮询完成后刷新帧图片
11. 轮询间隔：2 秒；超时：5 分钟后停止并显示错误
12. 每个任务独立管理轮询句柄（不复用全局 pollTimer，避免冲突）
13. 生成中显示"取消"按钮，点击后调用 `POST /api/tasks/{task_id}/cancel`，前端静默停止轮询并恢复按钮

## Acceptance Criteria

- [ ] `GET /api/tasks/{task_id}` 接口存在，返回 `{task_id, status, progress, result?, error?}`
- [ ] `generate_assets` POST 立即返回（< 500ms），不阻塞
- [ ] `regenerate_asset` POST 立即返回，不阻塞
- [ ] `render` POST 立即返回，不阻塞
- [ ] 前端批量生成时显示进度（如 "3/8 完成"）
- [ ] 前端单资产重生成时显示 spinner，完成后自动刷新图片
- [ ] 前端帧渲染时显示 spinner，完成后自动刷新帧图片
- [ ] 任务失败时前端显示错误信息
- [ ] 生成中显示"取消"按钮，点击后任务停止，前端恢复正常状态（不显示错误）

## Out of Scope

- 持久化任务（重启后任务丢失可接受）
- 任务队列限流（并发控制）
- WebSocket 推送（轮询已足够）
- Step5 视频生成异步化（单独任务）
