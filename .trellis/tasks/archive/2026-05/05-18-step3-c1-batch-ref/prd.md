# C1 批量生成 headshot 自动带 Full Body ref

## Goal

`generate_comic_assets`（批量生成全部资产）时，若角色已有 Full Body selected variant，headshot 生成自动携带 full body 作为参考图，与 B2 的单个重生成行为保持一致。

## 依赖

**前置任务**：`05-18-step3-b3-reverse-gen` 必须已合并并通过测试（整条 ref 传递链路已完整验证）。

## 执行流程

1. **实现**：由主 Claude 落代码（main.py only）
2. **Review**：由 Codex 审查代码变更
3. **修改**：主 Claude 根据 review 意见修改
4. **测试**：语法检查通过，人工验证批量生成时 headshot 请求携带 full body url
5. **通过后**：整个 `05-18-step3-prompt-consistency` 父任务完成，归档所有子任务

## Requirements

### 后端（main.py only）

- 修改 `generate_comic_assets`（main.py:3349）中 character headshot 生成循环：
  - 当前：`generate_ai_image(build_prompt(base), char_size, "auto", model, None, provider["id"])`
  - 改为：生成前查找 `char.get("selected_variant_id")` 对应的 full body variant url
  - 若无 selected，取 `char.get("variants", [])` 最后一个的 url
  - 若找到：`char_ref = [{"url": fb_url}]`；否则 `char_ref = None`
  - `generate_ai_image(build_prompt(base), char_size, "auto", model, char_ref, provider["id"])`
- 场景和道具生成不变（无参考图逻辑）

## 关键代码位置

- `generate_comic_assets`：main.py:3349
- character headshot 生成循环：main.py:3359-3370
- `generate_ai_image` 签名：main.py:1335

## Acceptance Criteria

- [ ] `uv run python3 -m py_compile main.py` 通过
- [ ] 批量生成时，若角色有 full body selected variant，headshot 生成请求携带其 url 作为参考图
- [ ] 无 full body 时，headshot 生成行为与改动前完全一致（纯 T2I）
- [ ] 场景和道具生成行为不变
- [ ] 批量生成失败处理逻辑不变（单个失败继续，最后汇总）

## Out of Scope

- 批量生成的异步化（单独的 Phase C 任务）
- 场景/道具的参考图逻辑
