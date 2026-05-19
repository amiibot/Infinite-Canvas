# A1+A2+A3 Prompt 质量基础修复

## Goal

修复 `regenerate_comic_asset` 中三个独立的 prompt 拼接缺陷，提升生成质量和用户体验。本任务是后续 B/C 系列任务的前置依赖。

## 依赖

无前置任务。

## 执行流程

1. **实现**：由主 Claude 落代码（main.py only）
2. **Review**：由 Codex 审查代码变更
3. **修改**：主 Claude 根据 review 意见修改
4. **测试**：`uv run python3 -m py_compile main.py` 通过，人工验证 prompt 拼接逻辑
5. **通过后**：进入下一任务 `05-18-step3-b1-use-ref`

## Requirements

### A1 — Base Prompt 写回

- `regenerate_comic_asset` 生成完成后，在 `_write` 回调中把不含 style 的 `base` 写回 `entry[prompt_key]`
- `prompt_key` 由 `_get_variant_fields(asset_type)` 第3个返回值决定
- 前端弹窗已有读取 `asset[promptKey]` 的逻辑（project.html:1238），无需改前端

### A2 — Style 重复检测

- 各 asset_type 分支中，style_prompt 从 f-string 内联提取出来，统一在分支结束后追加
- 追加条件：`body.apply_style and style_prompt and style_prompt not in base`
- 防止用户手动输入了 style 关键词后被重复追加

### A3 — Negative Prompt 合并

- 修改 main.py:3437 附近的 negative_prompt 赋值
- 旧：`body.negative_prompt or art_direction.get("style_negative_prompt", "")`（二选一）
- 新：`", ".join(filter(None, [body.negative_prompt, art_neg]))`（合并）

## 关键代码位置

- `regenerate_comic_asset`：main.py:3429
- negative_prompt 赋值：main.py:3437
- character full body 分支：main.py:3458
- character three_view 分支：main.py:3466
- character headshot 分支：main.py:3475
- scene 分支：main.py:3477
- prop 分支：main.py:3490
- `_write` 回调：main.py:3532

## Acceptance Criteria

- [ ] `uv run python3 -m py_compile main.py` 通过
- [ ] 重生成后，`item[prompt_key]` 被写入不含 style 的 base prompt
- [ ] 前端弹窗再次打开时，prompt textarea 显示上次的描述（不含 style）
- [ ] style_prompt 已在 base 中时，不会被重复追加
- [ ] 用户填写 negative prompt 时，与 art direction neg 合并（逗号分隔），而非二选一

## Out of Scope

- I2I 参考图（B 系列任务）
- 前端 UI 改动（除 A1 的弹窗默认值外，已有逻辑无需改）
