# implement.md：Seedance 2.0 专用节点

## 执行顺序

### Step 1：addSeedanceNode() 数据函数
- 位置：`static/canvas.html`，`addVideoNode` 之后（~行 2538）
- 创建节点数据，type='seedance'，含 model/duration/aspectRatio/resolution/prompt/generateAudio/watermark

### Step 2：工具栏按钮
- 位置：行 771（`addVideoNode` 按钮之后）
- 添加 `<button onclick="addSeedanceNode()">` + film 图标 + "Seedance" 文字

### Step 3：端口注册 + 渲染分发
- `canInput` 数组（行 4456）加入 `'seedance'`
- `canOutput` 数组（行 4457）加入 `'seedance'`
- 渲染分发（行 4435 区域）加入 `if(node.type === 'seedance') body.appendChild(renderSeedanceBody(node));`

### Step 4：renderSeedanceBody(node) UI 函数
- 位置：`renderVideoBody` 之后（~行 5600）
- 内容：prompt textarea + 模型 select + 比例/分辨率/时长 + toggles + 生成按钮
- 绑定所有控件的 oninput/onchange → 更新 node 属性 + scheduleSave()
- 生成按钮 onclick → runCanvasGenerate(node.id)

### Step 5：执行系统注册
- `canvasRunTypes()` 返回数组（行 6534）加入 `'seedance'`
- `runCascadeNodeByType()` 加入 `if(node.type === 'seedance') return runSeedanceNode(node.id, runOpts);`

### Step 6：runSeedanceNode() 执行函数
- 位置：`runVideoNode` 之后（~行 6179）
- 从连线上游收集图片 refs（复用 `generatorSources` / `orderedSources`）
- 使用 `node.prompt` 作为 prompt（不聚合上游）
- 找到支持 seedance 的 provider（videoApiProviders 中含 seedance 模型的，或 fallback 第一个）
- POST `/api/canvas-video`，model=node.model，duration/aspectRatio/resolution/generateAudio/watermark
- 输出处理：`result.videos` → appendOutputImages + mergeGeneratedOutputs

### Step 7：refreshRunNodes 刷新支持
- 检查 `refreshRunNodes` 中是否有按 type 过滤的逻辑，若有则加入 seedance

## 验证命令

```bash
# 启动服务
uv run uvicorn main:app --port 3000 --reload

# 浏览器打开画布，检查：
# 1. 工具栏有 Seedance 按钮
# 2. 创建节点后 UI 正常
# 3. 连接图片节点 → 填 prompt → 生成
```
