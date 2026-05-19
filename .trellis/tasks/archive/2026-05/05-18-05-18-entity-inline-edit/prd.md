# Step1 实体内联编辑 UI

## Goal

Step1 侧边栏为每个角色/场景/道具增加内联编辑入口，让用户可以直接修改已有实体的字段，而不必删除后重建。同时扩展后端 PATCH 接口支持 name/age/gender/clothing 字段。

## Background

当前 `renderEntitySection`（`project.html:1924`）只渲染 chip（名称 + 删除按钮），无编辑入口。后端 `PATCH /api/comic/projects/{pid}/assets/{type}/{id}`（`main.py:3219`）只支持 description/prompt/image_url，不支持 name/age/gender/clothing。

## Requirements

### 前端

1. 每个实体 chip 增加编辑按钮（铅笔图标），点击后在 chip 下方展开内联编辑表单。
2. 编辑表单字段：
   - 角色（characters）：name、description、age、gender、clothing
   - 场景（scenes）：name、description
   - 道具（props）：name、description
3. 表单预填当前值。
4. 点击"保存"调用 PATCH 接口，成功后收起表单并刷新侧边栏。
5. 点击"取消"直接收起表单，不发请求。
6. 同一时间只允许一个实体处于编辑状态（打开新的编辑表单时关闭其他）。
7. 编辑表单与添加表单互斥（打开编辑时关闭添加表单，反之亦然）。

### 后端

扩展 `AssetDescriptionUpdate`（`main.py:542`）增加字段：
- `name: Optional[str] = None`
- `age: Optional[str] = None`
- `gender: Optional[str] = None`
- `clothing: Optional[str] = None`

在 `update_asset_description`（`main.py:3219`）的 `_update` 函数中写回这些字段（仅 characters 支持 age/gender/clothing）。

## Acceptance Criteria

- [ ] 点击角色 chip 的编辑按钮，展开包含 name/description/age/gender/clothing 的内联表单，字段预填当前值
- [ ] 点击场景/道具 chip 的编辑按钮，展开包含 name/description 的内联表单
- [ ] 保存后实体名称/描述在侧边栏即时更新
- [ ] 取消后表单收起，数据不变
- [ ] 同时只有一个编辑表单展开
- [ ] 后端 PATCH 接口接受并写回 name/age/gender/clothing 字段
- [ ] 不影响现有的添加表单、删除、折叠/展开功能

## Notes

- 编辑表单用与添加表单相同的 `.inline-form` 样式，保持视觉一致。
- state 中用 `state.editingEntity = { kind, id }` 跟踪当前编辑状态，null 表示无编辑中。
- PATCH 路径复用现有 `/api/comic/projects/{pid}/assets/{type}/{id}`，asset_type 对 characters 传 `"character"`。
