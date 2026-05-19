# batch-gen 实施计划

## 现状
- 后端 `batch_size` 字段和循环已实现（main.py:550, 3407, 3481）
- `handleRegenAsset` 已支持 `opts.batchSize` 参数（project.html:2301）
- 角色工作台 `data-cw-gen` 按钮已传 `batch_size: 1`（project.html:1472）
- 缺：前端 x1-x4 选择器 UI + loading 文案

## 实施步骤

### 1. 角色工作台 footer 加 batch 选择器
- 每列 `.cw-col-footer` 内，生成按钮前加 4 个 radio 按钮 `x1 x2 x3 x4`
- 用 `data-cw-batch` 属性标记，默认 x1 选中

### 2. 工作台生成逻辑读取 batch_size
- `charWorkbench` click handler 中读取当前列的 batch 选择值
- 传入 fetch body

### 3. 工作台 loading 状态增强
- 生成中显示 "正在生成 N 张..." 替代 "⏳..."
- 生成完成恢复

### 4. 场景/道具弹窗加 batch 选择器
- `assetModalRegenBtn` 旁加 x1-x4 按钮组
- `handleRegenAsset` 调用处读取选中值

### 5. 弹窗 loading 状态
- 弹窗生成按钮生成中显示 "正在生成 N 张..."

## 验证
```bash
python3 -m py_compile main.py && echo "OK"
```
