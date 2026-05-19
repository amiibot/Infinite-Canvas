# Step3 资产弹窗工作台化重做

## Goal

将 luna-canvas Step3 资产弹窗从当前简易左右分栏升级为 luna-studio 同等体验的工作台模式。角色使用 3 列并排工作台，场景/道具使用 50/50 左右分栏弹窗。

## 背景

- 原版参考：luna-studio `CharacterWorkbench`（角色）+ `CharacterDetailModal`（场景/道具）
- 当前状态：统一的 65/35 左右分栏 + tab 切换，缺少多列视图、变体管理、锁定依赖等功能
- 文件：`static/project.html`（前端）、`main.py`（后端）

## 子任务结构

| Child | Slug | 内容 |
|-------|------|------|
| 1 | `05-17-variant-ops` | 共享变体操作（选择/删除/收藏）+ 后端 API |
| 2 | `05-17-char-workbench` | 角色 3 列工作台（Full Body + Three View + Headshot） |
| 3 | `05-17-scene-prop-modal` | 场景/道具 50/50 弹窗（Image/Video tab） |
| 4 | `05-17-motion-placeholder` | Motion 预留按钮（disabled 状态，不可使用） |

## 执行顺序

1. variant-ops（后端 + 共享 JS）→ 其他子任务依赖此基础
2. char-workbench（角色布局重做）
3. scene-prop-modal（场景/道具布局重做）
4. motion-placeholder（在已完成的布局上添加预留按钮）

## 全局约束

- 所有修改在 `static/project.html` + `main.py` 内完成（vanilla HTML/JS，无框架）
- 后端新增 three_view 数据结构支持
- Motion 按钮仅预留 UI，点击提示"即将推出"，不实现功能
- 变体收藏/删除需后端 API 支持

## Acceptance Criteria（Parent 级）

- [ ] 角色资产打开后显示 3 列工作台（Full Body / Three View / Headshot）
- [ ] 场景/道具资产打开后显示 50/50 左右分栏弹窗
- [ ] 变体可选择、删除、收藏（所有资产类型）
- [ ] Motion 按钮存在但 disabled，点击提示"即将推出"
- [ ] 无回归：现有资产 CRUD、重生成、描述编辑正常工作
- [ ] 刷新后状态持久化正常

## 待形成书面记录（后续考虑）

- 锁定依赖：Headshot/Three View 依赖 Full Body 先生成才解锁
- 批量生成（×1 / ×4）
- 反向生成 UI 提示（Upload Detected 覆盖层）

## Out of Scope

- Motion 实际功能（视频生成、音频上传）
- 从画布节点打开工作台
- 双向同步
