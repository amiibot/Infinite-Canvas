# PRD: luna-canvas Codex 反馈修复

## 背景

Codex review 对 luna-canvas 工作区提出 1 个 P1 和 3 个 P2 问题。
本任务在实施前先由 Claude 自行审查一遍，确认问题属实后再修复。

---

## Claude 自审结论

### P1 — API/.env 密钥泄露（已确认）

`API/.env` 包含真实 key（`sk-dongtaiMAN-2026`、UUID 格式 key），
当前 `.gitignore` 未覆盖该路径。若提交则密钥暴露。

**结论：属实，需修复。**

---

### P2-A — headshot 迁移路由错误（已确认）

`_migrate_project_assets`（main.py:591）调用：
```python
_migrate_asset_variants(char, ["full_body_image_url", "headshot_image_url"])
```
`_migrate_asset_variants` 统一写入 `item["variants"]`（full_body 数组），
而 headshot 应写入 `char["headshot_variants"]`。
`three_view_image_url` 完全未迁移。

**结论：属实，旧项目 headshot/three_view tab 会显示错误或为空。**

---

### P2-B — 批量生成不检查 locked（已确认）

`generate_comic_assets`（main.py:3350-3433）在快照阶段和写回阶段
均未检查 `char/scene/prop["locked"]`，会覆盖用户锁定的资产。
其他单资产 PATCH 接口已有 locked 检查。

**结论：属实，需在快照阶段跳过 locked 资产。**

---

### P2-C — asset 图片 URL 未转义（已确认）

`static/project.html:2023`：
```js
${item.image ? `<img src="${item.image}" ...>` : ...}
```
`item.image` 来自 `selectedUrl()`，返回存储的 URL 字符串，
直接插入模板字符串，未经 `escapeHtml`。
若 URL 含 `"` 可逃出属性，注入事件处理器。
同文件其他地方（modal、variant 渲染）已使用 `escapeHtml`。

**结论：属实，需修复。**

---

## 需求

### F1 — 将 API/.env 加入 .gitignore
- 在根目录 `.gitignore` 追加 `API/.env`
- 不删除文件本身（本地运行需要）

### F2 — 修复 headshot/three_view 迁移路由
- `_migrate_project_assets` 中对 char 分别处理：
  - `full_body_image_url` → `variants` / `selected_variant_id`
  - `headshot_image_url` → `headshot_variants` / `headshot_selected_variant_id`
  - `three_view_image_url` → `three_view_variants` / `three_view_selected_variant_id`
- 扩展 `_migrate_asset_variants` 支持指定目标 key，或新增专用逻辑

### F3 — 批量生成跳过 locked 资产
- `generate_comic_assets` 快照阶段加 `if not char/scene/prop.get("locked")` 过滤

### F4 — 转义 asset 图片 URL
- `project.html:2023` 改为 `<img src="${escapeHtml(item.image)}" ...>`

---

## 验收标准

- [ ] `API/.env` 被 `.gitignore` 覆盖，不出现在可提交列表
- [ ] 旧项目含 `headshot_image_url` 加载后，headshot tab 显示正确图片
- [ ] 旧项目含 `three_view_image_url` 加载后，three_view tab 显示正确图片
- [ ] 锁定资产后执行"生成全部资产"，锁定资产不被更新
- [ ] `python3 -m py_compile main.py` 通过
- [ ] `item.image` 渲染处使用 `escapeHtml`

---

## 范围外

- 轮换已泄露的密钥（用户自行处理）
- luna-studio 同类问题（另立任务）
