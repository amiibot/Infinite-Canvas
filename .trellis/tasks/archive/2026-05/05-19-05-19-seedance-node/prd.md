# PRD：Seedance 2.0 专用节点（无限画布）

## Goal

在无限画布中新增 `seedance` 节点类型，作为 Seedance 2.0 图生视频（i2v）的专用快捷入口。
相比通用 video 节点，参数精简、UI 聚焦，只暴露 Seedance 2.0 相关选项。

## 已确认事实

- 后端 `/api/canvas-video` 已原生支持 Seedance 模型（APIMart 分支，`main.py:2639`）
- Seedance 2.0 模型：`doubao-seedance-2-0-260128`（质量）/ `doubao-seedance-2-0-fast-260128`（快速）
- 现有 video 节点的 `runVideoNode` 调用 `/api/canvas-video`，输出 `result.videos`
- 节点端口系统：`canInput` / `canOutput` 数组控制连线权限
- `runCascadeNodeByType` 按 type 分发执行函数
- `canvasRunTypes()` 返回可执行节点类型列表

## Requirements

### 1. 节点数据结构

```js
{
  id: uid('sdn'),
  type: 'seedance',
  x, y,
  model: 'doubao-seedance-2-0-260128',
  duration: 5,
  aspectRatio: '16:9',
  resolution: '',
  prompt: '',
  generateAudio: false,
  watermark: false,
  inputs: [],
  running: false
}
```

### 2. 节点 UI（renderSeedanceBody）

- prompt textarea（直接内嵌，不依赖外部 prompt 节点）
- 模型 select：质量版 / 快速版 两个选项
- 一行三列：比例 select / 分辨率 select / 时长 input
- 可选 toggle：生成音频 / 水印
- 生成按钮

### 3. 端口

- 左侧 `in` 端口：接收图片输入（来自 image/asset 节点）
- 右侧 `out` 端口：输出视频（连接到 output 节点）

### 4. 执行逻辑（runSeedanceNode）

- 从连线的上游节点收集图片 refs
- 使用节点内嵌 prompt（不从上游 prompt 节点聚合）
- 调用 `/api/canvas-video`，传入 Seedance 模型和参数
- 输出处理与 video 节点相同（`result.videos`）

### 5. 工具栏入口

- toolbar 新增 Seedance 按钮，点击调用 `addSeedanceNode()`

## Acceptance Criteria

- [ ] 工具栏出现 Seedance 按钮，点击后在画布创建节点
- [ ] 节点显示 prompt textarea + 模型/比例/分辨率/时长控件
- [ ] 左侧 in 端口可接收 image/asset 节点连线
- [ ] 右侧 out 端口可连接到 output 节点
- [ ] 填写 prompt 后点击生成，请求发送到 `/api/canvas-video`
- [ ] 生成完成后视频出现在 output 节点
- [ ] 节点持久化：刷新后节点和参数仍存在

## Out of Scope（本期）

- 多图输入（首帧+尾帧）
- 从上游 prompt 节点聚合文本
- 级联执行（cascade）中的 seedance 节点
