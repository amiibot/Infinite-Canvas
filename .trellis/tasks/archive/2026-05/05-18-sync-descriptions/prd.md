# sync_descriptions — LLM 批量补全描述

## Goal

Step1 解析剧本后，角色/场景/道具的 description 字段可能为空或过于简短。
新增 `POST /api/comic/projects/{pid}/sync_descriptions` 接口，
由 LLM 批量补全/更新这些 description 字段；
Step1 页面增加"同步描述"按钮，触发异步任务并轮询进度。

## Background

- 现有 `/parse` 接口已能提取 name，但 description 质量参差不齐。
- 用户手动填写 description 费时，LLM 可根据 name + 原始剧本文本自动补全。
- 批量处理（每个实体一次 LLM 调用）耗时可能超 30s，必须走异步任务模式。

## Requirements

### 功能需求

1. 新增接口 `POST /api/comic/projects/{pid}/sync_descriptions`
   - 请求体：`{ "force": false, "provider_id": "", "model": "" }`
   - `force=false`（默认）：只补全 description 为空字符串或 None 的实体
   - `force=true`：覆盖所有实体的 description（全量更新）
   - 立即返回 `{ "_task_id": "..." }`，后台异步执行

2. 后台任务 `_bg_sync_descriptions`
   - 遍历 characters、scenes、props（各自独立处理）
   - 每个实体单独调用 `call_chat_completion`，生成 1-3 句英文描述
   - 写回时用 `atomic_update_comic_projects`，只更新 description 字段
   - 通过 `_update_task(task_id, progress=done, total=total)` 上报进度

3. Step1 前端新增"同步描述"按钮
   - 位置：parseBtn 旁边（同一行按钮区）
   - 仅在 `project.characters.length + project.scenes.length > 0` 时显示
   - 点击后调用接口，轮询 `/api/tasks/{task_id}` 直到 completed/failed
   - 轮询期间按钮显示 spinner + 禁用，完成后刷新项目数据并提示

### 非功能需求

- 单个实体 LLM 调用失败不中断整体任务，记录到 failures 列表
- 全部失败时任务状态置为 failed
- 部分失败时任务状态为 completed，result 中包含 failures 列表

## Acceptance Criteria

- [ ] `POST /api/comic/projects/{pid}/sync_descriptions` 返回 `_task_id`
- [ ] `GET /api/tasks/{task_id}` 能查到进度（progress/total）
- [ ] force=false 时，已有 description 的实体不被覆盖
- [ ] force=true 时，所有实体 description 均被更新
- [ ] 全部失败时 task.status = "failed"
- [ ] 部分失败时 task.status = "completed"，result.failures 非空
- [ ] Step1 页面有"同步描述"按钮，轮询完成后侧边栏刷新
- [ ] python3 -m py_compile main.py 通过
- [ ] JS 语法检查通过

## Out of Scope

- props 的 description 同步（可选，实现时一并处理，不单独 AC）
- 流式输出
- 每个实体的 description 字段长度限制（由 LLM prompt 控制）
