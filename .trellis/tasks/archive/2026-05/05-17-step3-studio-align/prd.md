# Step3 资产功能对齐 luna-studio

## Goal

将 luna-studio Step3 资产管理的核心功能回迁到 luna-canvas，实现功能对齐。本任务为父任务，拆分为独立子任务分别实施。

## Scope

对齐以下 6 项功能差距（按优先级排列）：

| # | 功能 | 子任务 |
|---|------|--------|
| 1 | 批量生成 (batch_size x1-x4) | batch-gen |
| 2 | 生成中 loading 状态 + 进度文案 | batch-gen (同上) |
| 3 | 角色工作台列上传按钮 | cw-upload |
| 4 | 资产锁定 (lock) | asset-lock |
| 5 | 异步任务轮询 | async-poll |
| 6 | 模型选择 | model-select |

## 实施分组

- **Group A** (本轮): #1 + #2 batch-gen, #3 cw-upload, #4 asset-lock
- **Group B** (后续): #5 async-poll, #6 model-select（架构改动大，延后）

## Cross-child Acceptance Criteria

- [ ] 所有子任务独立通过 `python3 -m py_compile main.py`
- [ ] 各功能互不干扰，无回归
- [ ] 角色工作台 3 列 + 场景/道具弹窗均正常工作

## Handoff / Deferred

本轮完成 Group A（batch-gen, cw-upload, asset-lock）。
Group B（async-poll, model-select）延后，将在本任务完成后创建独立 follow-up 任务。
