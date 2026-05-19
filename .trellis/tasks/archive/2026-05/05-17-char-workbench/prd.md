# 角色3列工作台

## Goal

将角色资产弹窗从当前左右分栏改为 3 列并排工作台布局（Full Body + Three View + Headshot），每列独立管理变体和生成。

## 参考

luna-studio `CharacterWorkbench`：3 列等宽面板，每列包含标题 + 图片区 + VariantSelector + 独立 prompt + 生成按钮。

## Requirements

### 布局

- 弹窗尺寸：`max-w-7xl h-[90vh]`
- Header：资产名 + "工作台" + 关闭按钮
- Body：3 列 flex 等分（`flex-1` 各列）
- 每列结构：
  - 标题栏（Full Body / Three View / Headshot）+ Static/Motion 切换预留
  - 图片区（当前选中变体大图 / 占位符）
  - 变体条（复用 variant-ops 的共享组件）
- Footer：负面提示词 textarea + Apply Style 开关 + Art Direction 折叠面板

### 每列独立 prompt

- Full Body：`asset.prompt || getInitialPrompt(asset)`
- Three View：`asset.three_view_prompt || ""`
- Headshot：`asset.headshot_prompt || ""`
- 生成时传对应 prompt 到后端

### 生成

- 每列独立"生成"按钮，调用 `regenerate_comic_asset` 并传 `asset_type` = `character_full_body` / `character_three_view` / `character_headshot`
- 生成中显示 loading 状态（该列 disabled）

### 数据流

- 打开弹窗 → 读取 character 数据填充 3 列
- 生成完成 → 刷新对应列的变体条和主图
- 关闭弹窗 → 无额外操作

## Acceptance Criteria

- [ ] 角色弹窗打开后显示 3 列并排布局
- [ ] 每列显示对应视图的当前图片和变体条
- [ ] 每列有独立 prompt textarea 和生成按钮
- [ ] 生成 Full Body → 只刷新第一列
- [ ] 生成 Three View → 只刷新第二列
- [ ] 生成 Headshot → 只刷新第三列
- [ ] Footer 负面提示词和 Art Direction 全局共享
- [ ] 无图时显示占位符文本

## 依赖

- `05-17-variant-ops`（变体条组件 + three_view 后端支持）
