# PRD：Step3 资产弹窗工作台改造

## 目标

将 `static/project.html` 中的资产弹窗从"展示卡片"风格改造为"工作台"风格，
参考 luna-studio `CharacterWorkbench` 的布局，提升资产编辑体验。

## 背景

当前弹窗为 55/45 左右分栏，左侧图片偏小，右侧描述文本占据大量空间，
操作按钮层级不够突出。原版 CharacterWorkbench 是更合适的工作台范式。

## 已确认事实（代码探查）

### 当前 HTML 结构（行 852–925）
```
.asset-modal-overlay
  .asset-modal                          (max-width:900px, max-height:88vh)
    .asset-modal-left  (width:55%)      ← 图片区
    .variant-strip                      ← 变体缩略图条（当前在 left 外、right 外）
    .asset-modal-right (width:45%)      ← 参数区
      .asset-modal-header               ← badge + title + close
      .asset-modal-body                 ← 描述 / 风格 / 提示词 / 高级 / char-tabs
      .asset-modal-footer               ← Regenerate + Done
```

### 当前 CSS 关键值
- `.asset-modal-left`: width:55%, min-height:420px
- `.asset-modal-right`: width:45%
- `.asset-modal`: max-width:900px

### 变体条位置问题
`#variantStrip` 当前在 `.asset-modal-left` 和 `.asset-modal-right` 之间，
不在任何一侧内部，视觉上悬空。

### 参数区内容顺序（当前）
1. 描述（可编辑）
2. 风格设置（折叠面板）
3. 生成提示词 textarea
4. 高级设置（负向提示词，折叠）
5. 角色 tab（Full Body / Headshot）

### JS 关键函数
- `openAssetModal(assetType, assetId)` — 填充所有字段
- `refreshAssetModalImage(url, alt)` — 更新左侧图片
- `renderVariantStrip(asset, type)` — 渲染变体条
- `syncAssetModalCharTabs()` — 同步角色 tab 状态

## 需求

### N1：扩大左侧预览区
- 左侧宽度从 55% → 65%
- 右侧宽度从 45% → 35%
- 弹窗最大宽度从 900px → 1040px（保持 88vh 高度限制）
- 左侧图片填满整个面板（object-fit: contain）

### N2：变体条移入左侧底部
- `#variantStrip` 从当前位置移入 `.asset-modal-left` 内部，固定在底部
- 左侧改为 flex column，图片 flex:1，变体条 flex-shrink:0

### N3：右侧参数区重排（工作台顺序）
从上到下：
1. **Header**：badge + title + close（保持不变）
2. **角色 tab**（Full Body / Headshot）— 移到顶部，仅角色资产显示
3. **生成提示词** textarea（主要输入）
4. **描述**（折叠或简化，次要）
5. **风格设置**（折叠面板，保持现有逻辑）
6. **高级设置**（负向提示词，折叠，保持现有逻辑）
7. **Footer**：Regenerate（主按钮）+ Done

### N4：主按钮视觉强化
- Regenerate 按钮高度从 38px → 42px
- 字体加粗，视觉权重高于 Done

## 不在本轮范围

- 后端 API 变更
- 新增字段（prompt、is_uploaded_source 等，属于另一个任务）
- 三面板布局（luna-studio 的 Full Body / Three View / Headshot 三列）
- 动画/主题变更

## 验收标准

- [ ] 弹窗打开后，左侧图片区宽度明显大于右侧参数区（65/35）
- [ ] 变体缩略图条显示在左侧图片下方，不再悬空
- [ ] 角色资产弹窗：右侧顶部显示 Full Body / Headshot tab
- [ ] 生成提示词 textarea 在描述之前显示
- [ ] Regenerate 按钮视觉权重最高（高度 42px，加粗）
- [ ] 场景/道具弹窗无回归（无角色 tab，其余功能正常）
- [ ] 深色/浅色主题均正常
