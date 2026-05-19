# Seedance 节点高级模式（首尾帧 / 多图参考）

## Goal

为画布 Seedance 节点增加 Seedance 2.0 的三种高级模式支持：
- `single_image`：单图生视频（当前已有）
- `first_last_frame`：首帧 + 尾帧生视频
- `multi_image`：主图 + 多张参考图/参考视频

## 已确认事实

- 后端 `CanvasVideoRequest.images` 支持 `AIReference(url, name, role)`
- APIMart 分支处理 `image_with_roles`：role 可为 `first_frame` / `last_frame` / `reference_image`
- `CanvasVideoRequest.videos` 支持参考视频 URL 列表（最多 3 个）
- 当前 `runSeedanceNode` 把所有上游图片作为无 role 的 images 传入
- 画布节点通过左侧 in 端口接收连线，`orderedSources` 收集上游图片 refs

## Requirements

### 1. 模式选择 UI

在 `renderSeedanceBody` 中 prompt 下方增加模式切换（三个按钮或 select）：
- 单图模式（默认）：行为不变
- 首尾帧模式：需要恰好 2 张输入图片
- 多图参考模式：第 1 张为主图，其余为参考图（最多 9 张）；可附加参考视频

节点数据新增字段：`seedanceMode: 'single_image' | 'first_last_frame' | 'multi_image'`

### 2. 输入图片 role 分配

根据 `seedanceMode` 和输入图片顺序自动分配 role：

| 模式 | 图片 1 role | 图片 2 role | 图片 3+ role |
|------|------------|------------|-------------|
| single_image | （无 role，走 image_urls） | — | — |
| first_last_frame | `first_frame` | `last_frame` | — |
| multi_image | `first_frame` | `reference_image` | `reference_image` |

### 3. 参考视频支持

多图参考模式下，节点数据新增 `referenceVideos: string[]`（最多 3 个 URL）。
UI 提供简单的 URL 输入或从 asset 节点连线获取视频。
传入后端 `videos` 字段。

### 4. 执行逻辑修改（runSeedanceNode）

- 根据 `node.seedanceMode` 构造 `images` 数组时带上 role
- `first_last_frame` 模式校验：输入图片必须恰好 2 张
- `multi_image` 模式：第 1 张 role=first_frame，其余 role=reference_image
- 如果 `node.referenceVideos` 有值，传入 `videos` 字段

### 5. 模式切换时的 UI 提示

- 切换到 first_last_frame 时，若输入不足 2 张，显示提示文字
- 切换到 multi_image 时，显示"第 1 张为主图，其余为参考"的说明

## Acceptance Criteria

- [ ] 节点 UI 出现模式切换控件，默认 single_image
- [ ] 切换模式后 `node.seedanceMode` 持久化
- [ ] first_last_frame 模式：2 张输入图 → 请求 images 带 first_frame/last_frame role
- [ ] first_last_frame 模式：输入不足 2 张时生成按钮禁用或弹出提示
- [ ] multi_image 模式：多张输入 → 第 1 张 first_frame，其余 reference_image
- [ ] multi_image 模式：referenceVideos 传入 videos 字段
- [ ] single_image 模式行为与当前完全一致（无回归）

## Out of Scope

- 参考音频输入（低优先级，后续做）
- Prompt token 引用语法（`[图片1]`）
- 镜头/运动控制 select
- Seed / negativePrompt / promptExtend
