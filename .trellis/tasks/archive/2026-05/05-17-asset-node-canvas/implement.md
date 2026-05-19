# Implement: asset 节点集成到无限画布

## 执行顺序

### 1. 节点基础设施（canvas.html）

- [ ] `defaultNodeSize` 加入 `'asset'` → `{w:220, h:0}`
- [ ] `canOutput` 列表加入 `'asset'`
- [ ] `renderNode` 新增 `case 'asset'` 分支（head: badge+name, body: img/placeholder）
- [ ] `generatorSources` 新增 `asset` 分支返回 refs

**验证**：手动在 console 创建 asset 节点，确认渲染正常、可连线到 generator

### 2. 选择器弹窗 UI

- [ ] HTML: `#assetPickerModal` 弹窗结构（project 下拉 + 资产网格 + 确认/取消）
- [ ] CSS: 弹窗样式（复用现有 modal 模式）
- [ ] JS: `showAssetPicker()` — 打开弹窗，加载项目列表
- [ ] JS: 选择 project 后加载详情，渲染资产缩略图列表（分 character/scene/prop 三组）
- [ ] JS: 点击资产 → 高亮选中；确认 → 创建节点落到视口中心 → 关闭弹窗
- [ ] JS: 关闭逻辑（遮罩点击 / 取消 / ESC）

**验证**：弹窗打开 → 选项目 → 选资产 → 节点出现在画布

### 3. 入口接入

- [ ] toolbar 新增"资产"按钮，onclick=showAssetPicker()
- [ ] createMenu 新增"资产卡片"选项

**验证**：toolbar 按钮和右键菜单均能触发选择器

### 4. 持久化验证

- [ ] 创建 asset 节点 → 触发 scheduleSave → 刷新页面 → 节点仍在
- [ ] asset 节点图片正常显示

### 5. 边界情况

- [ ] 无图资产：显示占位符
- [ ] 无项目时：选择器空状态
- [ ] 图片 URL 失效：img onerror 显示占位符

## 回滚点

所有改动在 canvas.html 单文件内，回滚 = git checkout canvas.html。

## 验收命令

```bash
# 服务启动
lsof -ti:3000 | xargs kill -9 2>/dev/null; uv run python main.py &
# 浏览器打开画布页面手动验证
```
