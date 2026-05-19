# UI合规修复 P1+P2（#10-#19,#24）

## Goal

完成 UI 审查中剩余 P1（#10-13）和 P2（#14-19）问题，以及 P3 中的 #24。

## Requirements

| # | 问题 | 文件 |
|---|------|------|
| 10 | `<select>` 缺少 `aria-label` | 5 文件 |
| 11 | mousemove getBoundingClientRect 布局抖动 | history-bulk-manager.js |
| 12 | Modal 缺少 `overscroll-behavior: contain` | canvas.html |
| 13 | `#board` 缺少 `touch-action: manipulation` | canvas.html |
| 14 | `<img>` 缺少 `width`/`height` | 全部，~40 处 |
| 15 | 下方图片缺少 `loading="lazy"` | JS 渲染列表 |
| 16 | i18n.js `"..."` → `"…"` | 1 文件，~50 处 |
| 17 | HTML 中 `"..."` → `"…"` | 5 文件 |
| 18 | theme.css `:focus-visible` 全局样式 | 已完成（上轮） |
| 19 | theme.css `color-scheme: light` | 1 文件 |
| 24 | project.html 缺少 `</head>` | 1 文件 |

## Acceptance Criteria

- [ ] 所有 `<select>` 有 aria-label
- [ ] getBoundingClientRect 不在 forEach 内逐项调用
- [ ] canvas.html modal 有 overscroll-behavior:contain
- [ ] #board 有 touch-action:manipulation
- [ ] 静态 `<img>` 有 width/height
- [ ] JS 渲染列表图有 loading="lazy"
- [ ] i18n.js 无 `...`（全部替换为 `…`）
- [ ] HTML 中无 `...`（UI 文本）
- [ ] theme.css 有 color-scheme:light
- [ ] project.html head 标签正确闭合
