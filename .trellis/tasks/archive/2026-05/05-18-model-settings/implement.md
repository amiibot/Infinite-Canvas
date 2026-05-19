# Implement: per-project 模型配置

## 实现顺序

### 后端

- [ ] 1. `ComicArtDirectionSave` 新增 `image_model: str = ""` 和 `aspect_ratio: dict = {}`（main.py L535）
- [ ] 2. `_bg_generate_assets` L3442：model 读取改为优先用 `art_direction.image_model`
- [ ] 3. `_bg_regenerate_asset` L3562：同上
- [ ] 4. `_bg_render_frame` L4088：同上
- [ ] 5. `python3 -m py_compile main.py` 通过

### 前端

- [ ] 6. `state` 初始值新增 `imageModels: []` 和 `selectedImageModel: ''`
- [ ] 7. `loadProject` 改为并行请求 `/api/config`，将 `image_models` 存入 `state.imageModels`
- [ ] 8. `hydrateProject` 从 `art_direction` 恢复 `selectedImageModel` 和 `aspectRatio`
- [ ] 9. `renderStep2` 在风格提示词上方插入模型下拉（`#imageModelSelect`）
- [ ] 10. `handleApplyStyle` 读取 `#imageModelSelect` 值，POST 时带上 `image_model` 和 `aspect_ratio`
- [ ] 11. JS 语法检查通过

## 验证命令

```bash
python3 -m py_compile main.py && echo "OK"

python3 -c "
import re; html = open('static/project.html').read()
scripts = re.findall(r'<script[^>]*>(.*?)</script>', html, re.DOTALL)
open('/tmp/check_js.js', 'w').write('\n'.join(scripts))
" && node --check /tmp/check_js.js && echo "JS OK"
```

## 风险点

- `loadProject` 改为 `Promise.all` 后，`/api/config` 失败会导致整个页面加载失败 → 用 `Promise.allSettled` 或单独 try/catch 保护 config 请求
- `aspect_ratio` 存为 dict，后端不做字段校验，前端传什么就存什么 → 可接受，前端只有固定下拉选项

## 回滚点

- 后端：`ComicArtDirectionSave` 字段新增是向后兼容的，旧数据不受影响
- 前端：`hydrateProject` 的 fallback 保证旧项目行为不变
