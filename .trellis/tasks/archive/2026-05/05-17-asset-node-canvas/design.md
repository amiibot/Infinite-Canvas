# Design: asset 节点集成到无限画布

## 架构概述

在 canvas.html 现有节点系统中新增 `asset` 类型，复用 image 节点的渲染模式（图片 + 标题），通过 toolbar/createMenu 入口 + 弹窗选择器放置节点。

## 数据结构

```js
// asset 节点（存储在 nodes[] 中，随画布 PUT 持久化）
{
  id: uid('asset'),
  type: 'asset',
  x: Number, y: Number,
  projectId: String,    // comic project id
  assetType: String,    // 'character' | 'scene' | 'prop'
  assetId: String,      // 资产 id
  url: String,          // 落地时快照的图片 URL
  name: String,         // 资产名称
  assetKind: String     // 显示用标签：'角色'/'场景'/'道具'
}
```

## 渲染

- `defaultNodeSize('asset')` → `{w: 220, h: 0}`（高度自适应）
- `renderNode` 新增 `asset` 分支：
  - node-head: badge（assetKind）+ name
  - node-body: `<img>` 渲染 url，无图时占位符（同 image 节点模式）
- 样式复用 `.node-image` 类，badge 用小标签样式

## 连线

- `canOutput` 列表加入 `'asset'`
- `generatorSources` 新增 `asset` 分支：
  ```js
  if(n.type === 'asset' && n.url) return {id:n.id, type:'asset', label:n.name, refs:[{url:n.url, name:n.name}], prompt:''};
  ```
- 不加入 `canInput`（asset 节点不接收输入）

## 放置入口

### 选择器弹窗（`#assetPickerModal`）

- 触发：toolbar "资产" 按钮 / createMenu "资产卡片" 选项
- 流程：加载 `GET /api/comic/projects` → 用户选 project → 加载详情 → 展示资产列表（缩略图 + 名称）→ 点击确认 → 创建节点落到视口中心
- 关闭：点击遮罩 / 取消按钮 / ESC

### toolbar

- 在现有按钮列后新增：`<button class="tool-btn" onclick="showAssetPicker()">资产</button>`

### createMenu

- 在菜单项列表中新增 `{label: '资产卡片', action: showAssetPicker}`

## 后端

无新接口。复用：
- `GET /api/comic/projects` — 项目列表
- `GET /api/comic/projects/{pid}` — 项目详情（含 characters/scenes/props）

## 兼容性

- 旧画布数据无 asset 节点，无需迁移
- 新节点通过现有 PUT /api/canvases 持久化，无 schema 变更
- 未识别的 node.type 在旧版前端会被 fallback 渲染（`defaultNodeSize` 返回默认尺寸），不会崩溃

## 边界情况

- 资产图片 URL 失效（项目删除/资产重生成）：显示占位符，不阻塞画布加载
- 无 comic projects 时：选择器显示空状态提示
- 资产无图片（未生成）：节点显示占位符
