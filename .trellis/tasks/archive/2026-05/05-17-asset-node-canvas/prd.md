# PRD：asset 节点集成到无限画布（MVP）

## Goal

在无限画布中新增 `asset` 节点类型，能渲染导演工坊的资产（角色/场景/道具）图片，并可连线到 generator 节点作为参考图。MVP 阶段通过 toolbar 和右键菜单的临时入口放置节点，抽屉/面板入口后续迭代。

## 已确认事实

- luna-canvas 统一后端（FastAPI port 3000），`static/canvas.html` 是无限画布，共享 `main.py`
- 画布节点以 `{id, type, x, y, ...}` 存储，通过 `PUT /api/canvases/{id}` 持久化
- toolbar（`.panel.toolbar`）和右键菜单（`#createMenu`）是两个现有的节点添加入口
- `generatorSources()` 决定哪些节点可作为 generator 输入；`canOutput` 列表控制连线权限
- 资产数据通过 `GET /api/comic/projects` 列表 + `GET /api/comic/projects/{pid}` 详情获取
- 资产图片在 `characters[].variants[].url`、`scenes[].variants[].url`、`props[].variants[].url`
- 选中变体通过 `selected_variant` 字段标记（需确认字段名）

## Requirements

### 1. 新节点类型 `asset`（canvas.html）

节点数据结构：
```js
{
  id: uid('asset'),
  type: 'asset',
  x, y,
  projectId: string,
  assetType: 'character' | 'scene' | 'prop',
  assetId: string,
  url: string,       // 落地时快照的图片 URL
  name: string,
  assetKind: string  // '角色' | '场景' | '道具'
}
```

渲染：
- `defaultNodeSize('asset')` → `{w: 220, h: 0}`
- node-head：assetKind badge + name
- node-body：图片（同 image 节点），无图时显示占位符
- 加入 `canOutput`，可连线到 generator
- `generatorSources()` 新增 `asset` 分支，返回 `{type:'asset', refs:[{url, name}], ...}`

### 2. 临时放置入口（MVP）

- toolbar 新增"资产"按钮，点击后弹出选择器弹窗
- 右键 `createMenu` 新增"资产卡片"选项，同样触发选择器弹窗
- 选择器弹窗：下拉选 project → 下拉/列表选资产 → 确认后落到画布中心

### 3. 后端

- 复用现有 `GET /api/comic/projects` 和 `GET /api/comic/projects/{pid}`，无需新增路由

## Acceptance Criteria

- [ ] canvas.html 中 asset 节点能正常渲染图片和名称
- [ ] asset 节点可连线到 generator，generator 能将其作为参考图（`generatorSources` 返回正确 refs）
- [ ] toolbar 和右键菜单均有"资产"入口，点击后弹出选择器
- [ ] 选择器能列出所有 comic projects 及其资产，选择后节点落到画布
- [ ] 节点持久化：刷新画布后 asset 节点仍存在且图片正常显示
- [ ] asset 节点无图时显示占位符，不报错

## Out of Scope（本期）

- 画布内资产抽屉/快速筛选面板
- asset 节点内重生成按钮
- 变体切换（headshot / full body）
- 从 project.html 发送到画布
- 双向同步

## Open Questions

无
