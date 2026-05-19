# Motion预留(按钮+disabled)

## Goal

在角色工作台和场景/道具弹窗中添加 Motion 模式的 UI 入口，但功能不可用，点击提示"即将推出"。

## Requirements

### 角色工作台

- 每列标题栏添加 Static / Motion 切换按钮组
- Static 默认选中高亮
- 点击 Motion → toast/alert 提示"Motion 功能即将推出"
- Motion 按钮样式为 disabled 态（半透明 + cursor-not-allowed）

### 场景/道具弹窗

- 左侧 Video tab 按钮存在
- 点击 Video tab → toast/alert 提示"Motion 功能即将推出"
- Video tab 样式为 disabled 态

## Acceptance Criteria

- [ ] 角色工作台每列有 Static/Motion 按钮组
- [ ] 点击 Motion 按钮显示"即将推出"提示，不切换状态
- [ ] 场景/道具弹窗有 Video tab 但不可切换
- [ ] 所有 disabled 按钮视觉上明确不可用

## 依赖

- `05-17-char-workbench`（角色工作台布局）
- `05-17-scene-prop-modal`（场景/道具弹窗布局）
