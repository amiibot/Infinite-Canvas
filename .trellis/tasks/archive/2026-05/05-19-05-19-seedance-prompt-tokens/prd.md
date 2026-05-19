# Seedance Prompt 引用 token 解析

## Goal

在 Seedance 节点的 prompt 中支持 `[图片1]` `[视频2]` 语法，用户可以在文本中引用连线输入的媒体资源，生成时自动解析并映射到实际 URL。配套提供素材预览条，让用户直观看到编号对应关系并一键插入 token。

## Context

Studio 的 VideoCreator 已有完整实现（`seedanceReferenceTokens.ts`）。Canvas 场景的区别：

- Studio 通过 UI 面板手动上传/选择参考媒体，有明确的编号
- Canvas 通过连线（connections）提供输入，需要从连线源节点推导编号
- Canvas 没有独立的"参考图片/视频"面板，所有输入来自连线 + referenceVideos 字段

## Requirements

### 1. Token 语法

- 正则：`/\[(图片|视频)(\d+)\]/g`
- `[图片N]` → 第 N 个图片输入的 URL
- `[视频N]` → 第 N 个视频输入的 URL
- 本版不支持 `[音频N]`（后端无 audio_urls 字段）

### 2. Inventory 构建（Canvas 适配）

从 Seedance 节点的连线输入构建 token inventory：

- **images**: 所有连线输入的图片（image 节点、asset 图片、output 节点图片），按连线创建时间排序编号
- **videos**: `node.referenceVideos` 数组 + 连线输入的视频 asset，按顺序编号

编号策略：按连线创建时间（connections 数组顺序）稳定排序，UI 上显示编号标签。

### 3. 素材预览条（Media Preview Row）

在 prompt textarea 上方显示横向滚动的缩略图条：

- 列出所有连线输入的图片和视频，带编号标签（如"图片1"、"视频2"）
- 图片显示缩略图，视频显示 video 标签或占位图标
- 点击缩略图 → 在 prompt textarea 光标位置插入对应 token `[图片1]`
- 无连线输入时不显示预览条
- 连线变化时自动更新预览条

### 4. 解析时机

在 `runSeedanceNode` 发起请求前：
1. 构建 inventory
2. 调用 token 解析（移植 Studio 的 `parseSeedanceReferenceTokens` 逻辑为纯 JS）
3. 用 `cleanPrompt` 替代原始 prompt 发送
4. `matchedImageUrls` 合并到 images 字段（带 role=reference_image）
5. `matchedVideoUrls` 合并到 videos 字段

### 5. 与 mode 的交互

- **所有模式**都支持 token 解析（后端 image_with_roles 在所有模式都生效）
- `single_image` 模式：token 引用的图片作为额外 reference_image 追加
- `first_last_frame` 模式：连线的前 2 张图仍作为首尾帧，token 引用的额外图片作为 reference_image
- `multi_image` 模式：token 引用的图片追加到 reference_image 列表

### 6. 无效 token 处理

- 无效 token（如 `[图片99]` 无对应输入）不阻塞生成
- 从 prompt 中静默移除无效 token
- 控制台 warn 提示

## Acceptance Criteria

- [ ] Seedance 节点 prompt 上方出现素材预览条，显示所有连线输入的图片/视频缩略图+编号
- [ ] 点击预览条中的缩略图，在 prompt 光标位置插入 `[图片N]` 或 `[视频N]`
- [ ] prompt 中写 `[图片1]` 时，生成请求的 images 包含第 1 个连线图片的 URL（带 role）
- [ ] prompt 中写 `[视频1]` 时，生成请求的 videos 包含对应视频 URL
- [ ] 无效 token 不阻塞生成，仅从 prompt 中移除
- [ ] token 解析后 prompt 中的 token 文本被清除，不发送给模型
- [ ] 无 token 时行为与当前完全一致（向后兼容）
- [ ] 连线变化时预览条自动更新
- [ ] Seedance 节点有独立 `in-audio` 端口，asset 节点有 `out-audio` 端口
- [ ] `[音频N]` token 解析后 reference_audio_urls 包含对应音频 URL
- [ ] 无效 token 时显示 toast 提示用户
- [ ] 模式切换时清除 referenceVideos 避免残留

## Out of Scope

- prompt 中 token 文本高亮
- 自动补全弹窗

## Reference

- Studio token 解析：`other/luna-studio/frontend/src/lib/seedanceReferenceTokens.ts`
- Studio 预览条：`other/luna-studio/frontend/src/components/modules/VideoCreator.tsx:1192`
- Studio inventory 构建：`VideoCreator.tsx:471` (`buildPromptTokenInventory`)
- Canvas 节点：`static/canvas.html` → `renderSeedanceBody()` / `runSeedanceNode()`
- 后端 Seedance 请求构造：`main.py:2638` (`image_with_roles` / `video_urls`)
