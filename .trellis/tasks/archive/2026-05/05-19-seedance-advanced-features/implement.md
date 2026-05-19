# implement.md：Seedance 高级模式

## 执行顺序

### Step 1：节点数据扩展

- `addSeedanceNode()` 新增字段：`seedanceMode: 'single_image'`, `referenceVideos: []`
- 位置：`static/canvas.html` ~行 2540

### Step 2：模式切换 UI

- `renderSeedanceBody()` 中 prompt 下方添加三按钮切换
- 按钮组：单图 / 首尾帧 / 多图参考
- 切换时更新 `node.seedanceMode` + scheduleSave()
- 位置：~行 5556（prompt textarea 之后）

### Step 3：参考视频 URL 输入

- 仅在 multi_image 模式下显示
- 简单文本输入框，逗号分隔或多行
- 绑定到 `node.referenceVideos` 数组
- 位置：renderSeedanceBody 内，模式按钮之后

### Step 4：模式提示文字

- first_last_frame：显示"需要连接 2 张图片（首帧 + 尾帧）"
- multi_image：显示"第 1 张为主图，其余为参考图"
- 根据实际输入数量动态显示警告

### Step 5：runSeedanceNode 修改

- 根据 `node.seedanceMode` 构造 images 数组
- single_image：保持现有逻辑（无 role）
- first_last_frame：refs[0] role=first_frame, refs[1] role=last_frame
- multi_image：refs[0] role=first_frame, refs[1..n] role=reference_image
- 如果 node.referenceVideos 有值，加入 videos 字段
- 位置：~行 6282

### Step 6：输入校验

- first_last_frame 模式：refs.length !== 2 时 alert 并 return
- multi_image 模式：refs.length < 1 时 alert 并 return

## 验证命令

```bash
uv run uvicorn main:app --port 3000 --reload

# 浏览器检查：
# 1. 创建 seedance 节点 → 模式切换按钮可见
# 2. 切换模式 → 刷新后模式保持
# 3. first_last_frame 模式 + 2 张图 → 请求 body 含 image_with_roles
# 4. multi_image 模式 + 3 张图 → 请求 body 含 image_with_roles
# 5. single_image 模式 → 行为不变
```
