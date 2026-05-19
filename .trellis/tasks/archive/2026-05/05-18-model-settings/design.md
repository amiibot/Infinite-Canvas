# Design: per-project 模型配置

## 数据模型变更

### `art_direction` 字段扩展

```python
# ComicArtDirectionSave 新增字段
class ComicArtDirectionSave(BaseModel):
    style_prompt: str = ""
    style_negative_prompt: str = ""
    style_name: str = ""
    image_model: str = ""                          # 新增：文生图模型，空字符串表示用全局默认
    aspect_ratio: dict = {}                        # 新增：{"character":"1:1","scene":"16:9","prop":"1:1"}
```

存储结构（project["art_direction"]）：
```json
{
  "style_prompt": "...",
  "style_negative_prompt": "...",
  "style_name": "...",
  "image_model": "gpt-image-2",
  "aspect_ratio": {"character": "1:1", "scene": "16:9", "prop": "1:1"}
}
```

### 迁移

旧项目 `art_direction` 没有 `image_model` / `aspect_ratio` 字段，读取时用 `.get()` + 默认值即可，无需 `_migrate_project_assets`（art_direction 不在资产迁移路径上）。

## 后端改动

### 1. `ComicArtDirectionSave` 扩展（main.py L535）

新增 `image_model: str = ""` 和 `aspect_ratio: dict = {}`。

### 2. 三处生成接口的 model 读取逻辑

统一改为：
```python
art_direction = project_snapshot.get("art_direction") or {}
project_model = art_direction.get("image_model", "").strip()
image_models = provider.get("image_models") or [IMAGE_MODEL]
model = selected_model(project_model or image_models[0], IMAGE_MODEL)
```

涉及位置：
- `_bg_generate_assets` L3442
- `_bg_regenerate_asset` L3562
- `_bg_render_frame` L4088

## 前端改动

### 1. `loadProject` 并行加载 config（project.html）

```js
async function loadProject() {
    const [project, config] = await Promise.all([
        api(`/api/comic/projects/${projectId}`),
        api('/api/config'),
    ]);
    state.imageModels = config.image_models || [];
    hydrateProject(project);
    render();
}
```

### 2. `hydrateProject` 恢复新字段

```js
state.selectedImageModel = project.art_direction.image_model || '';
state.aspectRatio = project.art_direction.aspect_ratio || { character: '1:1', scene: '16:9', prop: '1:1' };
```

### 3. `state` 初始值

```js
imageModels: [],
selectedImageModel: '',
```

### 4. `renderStep2` 新增模型下拉

在风格提示词表单上方插入：
```html
<label>
  <div class="section-title">图像模型</div>
  <select id="imageModelSelect">
    <option value="">全局默认</option>
    ${state.imageModels.map(m => `<option value="${escapeHtml(m)}" ${state.selectedImageModel===m?'selected':''}>${escapeHtml(m)}</option>`).join('')}
  </select>
</label>
```

### 5. `handleApplyStyle` 读取新字段

```js
const imageModelField = document.getElementById('imageModelSelect');
state.selectedImageModel = imageModelField?.value || '';
// aspect_ratio 已通过事件委托实时更新到 state.aspectRatio

await api(`/api/comic/projects/${projectId}/art_direction`, {
    method: 'POST',
    body: JSON.stringify({
        style_prompt: ...,
        style_negative_prompt: ...,
        style_name: ...,
        image_model: state.selectedImageModel,
        aspect_ratio: state.aspectRatio,
    })
});
```

## 兼容性

- 旧项目加载：`art_direction.image_model` 为 undefined → `hydrateProject` fallback 到 `''` → 生成时用全局默认，行为不变
- 旧项目加载：`art_direction.aspect_ratio` 为 undefined → fallback 到 `{character:'1:1',scene:'16:9',prop:'1:1'}`，行为不变
