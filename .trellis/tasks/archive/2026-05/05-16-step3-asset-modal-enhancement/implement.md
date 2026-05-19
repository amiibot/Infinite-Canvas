# Step3 资产弹窗增强 — 实施计划

## 实施顺序

### 阶段 A：后端扩展（main.py）

- [ ] A1. `AssetDescriptionUpdate` 模型新增 `prompt: Optional[str] = None`
- [ ] A2. PATCH 接口：`asset_type` 校验加入 `character_headshot`；保存 `prompt` 字段
- [ ] A3. `_make_variant` 新增 `is_uploaded_source: bool = False` 参数
- [ ] A4. `_append_variant` 透传 `is_uploaded_source` 给 `_make_variant`
- [ ] A5. PATCH 接口上传 URL 时传 `is_uploaded_source=True`
- [ ] A6. `ComicAssetRegenRequest` 模型新增 `prompt: Optional[str] = None`
- [ ] A7. `regenerate_comic_asset` 接口：若 `body.prompt` 非空，用其替换 `base` 提示词（而非拼接 name+description）

### 阶段 B：前端 HTML 结构（project.html）

- [ ] B1. 在弹窗右侧新增"生成提示词"label + `assetModalPrompt` textarea
- [ ] B2. 将风格区域改为可折叠面板（保留 checkbox，新增展开/折叠箭头 + body div）
- [ ] B3. 在 body div 内添加正向/负向提示词预览区

### 阶段 C：前端 JS 逻辑（project.html）

- [ ] C1. DOM 引用：新增 `assetModalPrompt`、`artDirectionPanel`、`artDirectionBody`、`artDirectionArrow`
- [ ] C2. `state` 新增 `modalArtDirectionOpen: false`
- [ ] C3. `openAssetModal`：填充 prompt、检测 is_uploaded_source 注入前缀、填充 art direction 面板
- [ ] C4. `openAssetModal`：角色资产根据 `modalCharType` 选择正确图片 URL（full_body vs headshot）
- [ ] C5. `saveAssetModal`（原 `saveAssetModalDescription`）：同时保存 description + prompt
- [ ] C6. `handleRegenAsset`：传入 `prompt` 参数
- [ ] C7. `toggleArtDirection()` 函数：切换折叠状态
- [ ] C8. `syncAssetModalCharTabs`：切换 tab 时刷新图片和变体条

### 阶段 D：验证

- [ ] D1. Python 语法检查：`python3 -m py_compile main.py`
- [ ] D2. 启动服务器，打开 Step3，测试各资产弹窗
- [ ] D3. 验证 R1：提示词可编辑、保存、重新生成时传递
- [ ] D4. 验证 R2：风格面板可折叠，展开后显示实际内容
- [ ] D5. 验证 R3：上传图片后重新打开弹窗，提示词包含 STRICTLY MAINTAIN 前缀
- [ ] D6. 验证 R4：角色弹窗切换 Full Body / Headshot，图片和变体条正确更新
- [ ] D7. 验证场景/道具弹窗不受影响

---

## 关键文件

| 文件 | 修改范围 |
|------|---------|
| `main.py` | `AssetDescriptionUpdate`（~3110行）、PATCH接口（3123-3146）、`_make_variant`（556）、`_append_variant`（590-603）、`regenerate_asset`（~3400行） |
| `static/project.html` | 弹窗HTML（828-890）、DOM引用（~967）、`state`初始化（~928）、`openAssetModal`（1111-1153）、`saveAssetModalDescription`（1921-1934）、`handleRegenAsset`（1885-1919）、`syncAssetModalCharTabs`（~975） |

---

## 风险点

1. `regenerate_asset` 接口是否已接受 `prompt` 参数 → 实施前先确认（A6）
2. headshot 变体与 full_body 变体共用 variants 数组 → 本轮简化处理，不做分离
3. 折叠面板 HTML 改动涉及现有 checkbox 事件绑定 → 改动时保留 `id="assetModalApplyStyle"`

---

## 回滚点

- 后端改动均向后兼容（新增可选字段，不删除旧字段）
- 前端 HTML 改动前备注原始结构（git diff 可追溯）
