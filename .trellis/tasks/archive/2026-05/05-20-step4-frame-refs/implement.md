# Step4 多参考图分镜生成 - 实现计划

## 实现顺序

### Step 1: 修复 generate_ai_image 参考图上限

**文件**: `main.py`
**改动**: 4 处 `[:4]` → `[:16]`
- 行 1976: `image_refs[:4]` (gpt2 generations)
- 行 1985: `image_refs[:4]` (gpt2 edits multipart)
- 行 2015: `image_refs[:4]` (gpt2 edits json)
- 行 2048: `refs[:4]` (modelscope fallback)

### Step 2: 新增参考图收集辅助函数

**文件**: `main.py`（放在 `_bg_render_frame` 前面）
**新增函数**:
- `_get_char_best_url(char)` — 角色最佳图 URL（three_view > full_body > headshot）
- `_get_asset_best_url(item)` — 场景/道具最佳图 URL
- `_collect_frame_refs(project, frame)` — 组装 reference_images 列表

### Step 3: 修改 _bg_render_frame 传入参考图

**文件**: `main.py:5170`
**改动**: 调用 `_collect_frame_refs` 获取 refs，替换 `generate_ai_image` 的 `None` 参数

### Step 4: 改造 analyze_storyboard LLM prompt

**文件**: `main.py:5109`
**改动**:
1. 从 project 中提取 characters/scenes/props 名称列表，构建 entities 上下文
2. 改写 system_prompt：提供实体列表，要求输出 `character_ref_names`/`scene_ref_name`/`prop_ref_names`
3. 解析 LLM 输出时，按名字模糊匹配（双向 `in`）映射为真实 ID
4. 写入 frame 的 `character_ids`/`scene_id`/`prop_ids`

### Step 5: 更新 prd.md

标记 requirements 和 acceptance criteria 完成状态。

## 验证

```bash
# 语法检查
python3 -c "import ast; ast.parse(open('main.py').read())"

# 启动服务确认无报错
python3 main.py &
sleep 3
curl -s http://localhost:3000/docs | head -5
kill %1

# 功能验证（手动）
# 1. 创建项目 → Step1 添加角色/场景
# 2. Step3 生成角色资产图
# 3. Step4 点击"AI 生成分镜"→ 确认帧数据中 character_ids 被填充
# 4. 点击"生成图片"渲染分镜 → 确认 API 调用包含 reference_images
```

## 风险点和回滚

- **LLM prompt 改动**：如果新 prompt 导致 LLM 输出格式异常，旧字段（action_description/dialogue/camera_movement/image_prompt）的解析逻辑不变，新字段（character_ref_names 等）解析失败只是 ID 为空，不影响核心功能
- **参考图上限修复**：从 4→16 是放宽限制，不会破坏现有功能
- **回滚**：所有改动在 main.py 单文件，git revert 即可
