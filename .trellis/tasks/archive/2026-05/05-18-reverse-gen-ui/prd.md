# PRD: Reverse Generation 提示 UI

## 目标

上传参考图后，在资产弹窗左侧图片区叠加显示视觉提示，让用户明确知道"参考图已生效，生成将保持外观一致"。

## 现状（代码查证）

- `hasUploadedSource`：检查 `asset.variants` 中是否有 `is_uploaded_source: true` 的变体（`project.html:1474`）
- 当 `hasUploadedSource` 为 true 时，prompt 自动加 `STRICTLY MAINTAIN...` 前缀（`project.html:1476`）
- 但弹窗左侧图片区（`#assetModalImg`，`.asset-modal-left`）无任何视觉提示
- 上传的参考图 URL 可从 `asset.variants.find(v => v.is_uploaded_source)?.url` 取得

## 需求

1. 当 `hasUploadedSource` 为 true 时，在 `#assetModalImg` 容器上叠加：
   - 底部渐变遮罩（CSS `::after` 或绝对定位 div）
   - 提示卡片：图标 + "参考图已上传 / Upload Detected" 文字
   - 参考图缩略图（小图，右下角或提示卡片内）
2. 纯 CSS + JS，不改后端
3. 叠加层不遮挡图片主体，不影响点击放大（lightbox）

## 实现要点

- `openAssetModal` 中已有 `hasUploadedSource` 变量，在此处控制叠加层显示/隐藏
- 叠加层 HTML 可静态写在 `#assetModalImg` 内，通过 `hidden` class 切换
- 缩略图 src 从 `asset.variants.find(v => v.is_uploaded_source)?.url` 取得
- 样式写在现有 `<style>` 块中

## 验收标准

- [ ] 上传过参考图的资产，打开弹窗时左侧图片区显示"Upload Detected"提示
- [ ] 提示区包含参考图缩略图
- [ ] 未上传参考图的资产，提示不显示
- [ ] 不影响图片点击放大（lightbox）
- [ ] JS 语法检查通过

## 超出范围

- 后端改动
- Step3 卡片列表上的提示（只做弹窗内）
- 删除参考图功能
