# 批量生成 batch_size + loading 状态

## Goal

为 Step3 资产重新生成功能添加批量生成支持（x1/x2/x3/x4）和生成中 loading 状态反馈。

## Requirements

### 后端
- `ComicAssetRegenRequest` 新增 `batch_size: int = 1` 字段（范围 1-4）
- `regenerate_comic_asset` 接口循环 `batch_size` 次生成，每次调用 `_append_variant`
- 返回所有新增变体的 URL 列表

### 前端 — 角色工作台
- 每列 footer 区域添加 `x1 x2 x3 x4` 按钮组（默认选中 x1）
- 点击生成时传入选中的 batch_size
- 生成中：按钮 disabled + 显示 "正在生成 N 张..." 文案 + spinner

### 前端 — 场景/道具弹窗
- 弹窗底部生成按钮旁添加同样的 x1-x4 选择器
- 生成中同样显示 loading 状态

## Acceptance Criteria

- [ ] 后端 `batch_size=3` 时生成 3 张变体并全部追加到 variants 列表
- [ ] 前端 x1-x4 按钮切换正常，选中态高亮
- [ ] 生成中显示 "正在生成 N 张..." + 按钮不可点击
- [ ] 生成完成后变体条自动刷新显示新变体
- [ ] `python3 -m py_compile main.py` 通过
