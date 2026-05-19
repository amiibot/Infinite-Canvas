# B1 单个重生成 use_current_as_ref

## Goal

在重生成弹窗中加入"参考当前图"开关，勾选后把当前 selected variant 作为参考图传给图片 API，提升重生成结果与原图的外观一致性。

## 依赖

**前置任务**：`05-18-step3-prompt-quality`（A1+A2+A3）必须已合并并通过测试。

## 执行流程

1. **实现**：由主 Claude 落代码（main.py + project.html）
2. **Review**：由 Codex 审查代码变更
3. **修改**：主 Claude 根据 review 意见修改
4. **测试**：语法检查通过，人工在浏览器验证复选框显示/隐藏逻辑及请求参数
5. **通过后**：进入下一任务 `05-18-step3-b2-derived-ref`

## Requirements

### 后端（main.py）

- `ComicAssetRegenRequest` 加 `use_current_as_ref: bool = False`
- `regenerate_comic_asset` 中，若 `use_current_as_ref=True`：
  - 用 `_get_variant_fields(asset_type)` 获取 `(vk, sk, _)`
  - 读取 `item.get(sk)` 对应的 variant url；若无 selected，取最后一个 variant 的 url
  - 构造 `ref_images = [{"url": ref_url, "name": item.get("name", "")}]`
  - 传给 `generate_ai_image(..., ref_images, ...)`
- 若 `use_current_as_ref=False` 或无可用 url，`ref_images = None`（保持当前行为）

### 前端（project.html）

- 弹窗生成按钮区域加复选框：
  ```html
  <label id="useRefLabel"><input type="checkbox" id="useCurrentAsRef"> 参考当前图</label>
  ```
- 打开弹窗时：若 `selectedUrl(asset, charType)` 有值，显示该复选框；否则隐藏
- 提交重生成请求时，body 加 `use_current_as_ref: document.getElementById('useCurrentAsRef').checked`
- 复选框默认**不勾选**

## 关键代码位置

- `ComicAssetRegenRequest`：main.py:545
- `regenerate_comic_asset` 生成循环前：main.py:3526
- 弹窗打开逻辑：project.html:1421（`openAssetModal`）
- 弹窗提交逻辑：搜索 `regenerate_asset` 的 fetch 调用

## Acceptance Criteria

- [ ] `uv run python3 -m py_compile main.py` 通过
- [ ] 弹窗打开时，有已生成图则显示"参考当前图"复选框，无图则隐藏
- [ ] 复选框默认不勾选
- [ ] 勾选后提交，请求 body 包含 `use_current_as_ref: true`
- [ ] 后端收到 `use_current_as_ref=true` 时，`generate_ai_image` 的 `reference_images` 参数非 None
- [ ] 未勾选时行为与改动前完全一致

## Out of Scope

- B2 的派生层级自动 ref（单独任务）
- B3 的反向生成（单独任务）
