# UI 合规修复 P0+P1（#1-#9）

## Goal

修复 Web Interface Guidelines 审查中 P0（#1-#7）和 P1（#8-#9）问题，提升项目可访问性和动效安全性。

## Background

2026-05-20 UI 审查发现全局性问题，详见 `docs/ui-audit-2026-05-20.md`。

## Requirements

### P0 — 必须修复

| # | 问题 | 文件 |
|---|------|------|
| 1 | `login.html` 缺少 `<meta name="viewport">` | login.html |
| 2 | `login.html` label/name/autocomplete 修复 | login.html |
| 3 | 图标按钮缺少 `aria-label` | 全部 HTML (~60处) |
| 4 | `<div onclick>` → `<button>` | 10 文件 (~40处) |
| 5 | 表单控件缺少 `<label>` 或 `aria-label` | 12 文件 (~30处) |
| 6 | `outline:none` 配合 `:focus-visible` 替代 | 12 文件 (~40处) |
| 7 | JS 文件无键盘支持 | history-bulk-manager.js, image-preview.js |

### P1 — 应该修复

| # | 问题 | 文件 |
|---|------|------|
| 8 | 无 `prefers-reduced-motion` 全局规则 | theme.css |
| 9 | `transition: all` → 具体属性列表 | 13 文件 (~70处) |

## Acceptance Criteria

- [ ] login.html 有 viewport meta、真实 `<label>`、正确 autocomplete、name 属性
- [ ] 所有图标按钮有 `aria-label`
- [ ] 所有 `<div onclick>` 改为 `<button>`（保留样式不变）
- [ ] 所有表单控件有关联 label 或 aria-label
- [ ] 所有 `outline:none` 处有 `:focus-visible` 替代样式
- [ ] history-bulk-manager.js 支持 Enter/Space 键盘选择
- [ ] image-preview.js 支持键盘缩放（+/-/0）
- [ ] theme.css 有 `@media (prefers-reduced-motion: reduce)` 规则
- [ ] 无 `transition: all`（全部替换为具体属性）
- [ ] 服务启动无报错

## Out of Scope

- P2/P3 问题（img width/height、loading=lazy、ellipsis、theme-color 等）
- 功能变更或视觉重设计
- 新增测试框架

## Implementation Strategy

按文件分批修改。优先 #1-2（login.html），再按文件批量处理 #3-6，最后 #7-9。
