# 场景/道具50-50弹窗

## Goal

将场景和道具资产弹窗改为 50/50 左右分栏布局，左侧变体管理（Image/Video tab），右侧描述编辑 + 风格控制 + 生成。

## 参考

luna-studio `CharacterDetailModal`：`max-w-5xl h-[85vh]`，左 50% VariantSelector，右 50% 控制面板。

## Requirements

### 布局

- 弹窗尺寸：`max-w-5xl h-[85vh]`
- 左侧（50%）：
  - Image/Video tab 切换（Video tab 为 Motion 预留，初始 disabled）
  - Image tab：当前选中变体大图 + 变体网格（复用 variant-ops 组件）
- 右侧（50%）：
  - Header：资产名 + 关闭按钮
  - 描述编辑（textarea + Edit/Save 切换）
  - Style Settings：Apply Style 开关 + 风格预览
  - Advanced Settings（折叠）：负面提示词
  - Footer：生成按钮

### 生成

- 场景调用 `regenerate_comic_asset` with `asset_type=scene`
- 道具调用 `regenerate_comic_asset` with `asset_type=prop`

## Acceptance Criteria

- [ ] 场景资产打开后显示 50/50 左右分栏
- [ ] 道具资产打开后显示 50/50 左右分栏
- [ ] 左侧 Image tab 显示变体网格 + 当前大图
- [ ] 左侧 Video tab 存在但 disabled（Motion 预留）
- [ ] 右侧描述可编辑保存
- [ ] 右侧 Apply Style 开关正常工作
- [ ] 生成后变体网格刷新

## 依赖

- `05-17-variant-ops`（变体网格组件）
