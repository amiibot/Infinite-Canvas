# Step2 自定义预设输入框内联化

## Goal

将 Step2「保存为预设」按钮触发的 `window.prompt()` 弹框，改为页面内联展开的 input + 确认/取消按钮，提升交互一致性和样式可控性。

---

## Background

当前实现：`handleSavePreset()` 调用 `window.prompt()` 获取预设名称，浏览器原生弹框样式不可控，且在某些环境下会被拦截。

目标：点击「保存为预设」后，在按钮旁展开一个内联 input 区域，用户输入名称后点击确认完成保存，或点击取消收起。

---

## Scope

纯前端改动，仅修改 `static/project.html`。不涉及后端、API、localStorage 结构变更。

---

## Requirements

### 状态管理

- 在 `state` 对象中新增字段 `showPresetInput: false`，用于控制内联输入区域的显示/隐藏。
- `showPresetInput` 为 `true` 时渲染输入区域，为 `false` 时隐藏。

### UI 结构（renderStep2 内）

- 「保存为预设」按钮保持原位（`#savePresetBtn`）。
- 当 `state.showPresetInput === true` 时，在按钮行下方（或同行右侧）渲染内联区域，包含：
  - `<input type="text" id="presetNameInput" placeholder="预设名称 / Preset name">`
  - 确认按钮 `#confirmPresetBtn`（文字：✓ 确认 / Confirm）
  - 取消按钮 `#cancelPresetBtn`（文字：✕ 取消 / Cancel）
- 内联区域展开时，「保存为预设」按钮可保持可见（disabled 状态）或隐藏，二选一，保持视觉整洁即可。

### 交互逻辑

| 操作 | 行为 |
|------|------|
| 点击「保存为预设」 | 先校验 stylePrompt 非空（同现有逻辑）；校验通过后 `state.showPresetInput = true`，`render()` |
| 点击「确认」 | 读取 `#presetNameInput` 的值，调用现有 `saveCustomPreset` 核心逻辑（写 localStorage、`render()`），然后 `state.showPresetInput = false` |
| 点击「取消」 | `state.showPresetInput = false`，`render()` |
| 确认时名称为空 | `setStatus` 提示错误，不关闭输入框 |
| 展开后按 Enter | 触发确认逻辑（可选，nice-to-have） |

### 现有逻辑复用

`handleSavePreset()` 中 `window.prompt` 之后的保存逻辑（写 localStorage、`render()`）提取为独立函数 `doSavePreset(name)`，供确认按钮调用。`handleSavePreset()` 改为只做校验 + 展开输入框。

---

## Acceptance Criteria

- [ ] 点击「保存为预设」不再弹出 `window.prompt()`
- [ ] 点击后页面内出现 input 输入框和确认/取消按钮
- [ ] 输入名称后点击确认，预设正确写入 localStorage 并刷新预设列表
- [ ] 点击取消，输入框收起，无副作用
- [ ] 名称为空时点击确认，显示错误提示，输入框不关闭
- [ ] `state.showPresetInput` 控制展开/收起，不使用 DOM 直接操作
- [ ] 不改动 localStorage 数据结构（`luna_custom_presets` 格式不变）
- [ ] JS 语法检查通过（`node --check /tmp/check_js.js`）

---

## Out of Scope

- 后端改动
- 预设名称重复校验（当前无此逻辑，不新增）
- 动画/过渡效果（可选锦上添花，不作为 AC）
- 预设数量上限逻辑（现有 10 条上限保持不变）

---

## Implementation Notes

- 事件委托已在 `contentPanel` 的 click handler 中处理，新增 `#confirmPresetBtn` 和 `#cancelPresetBtn` 的判断分支即可。
- `renderStep2()` 使用模板字符串拼接 HTML，内联区域用条件表达式 `state.showPresetInput ? '...' : ''` 插入。
- 输入框展开后建议 `setTimeout(() => document.getElementById('presetNameInput')?.focus(), 0)` 自动聚焦。
- 验证命令（改动后必须执行）：
  ```bash
  python3 -c "
  import re; html = open('static/project.html').read()
  scripts = re.findall(r'<script[^>]*>(.*?)</script>', html, re.DOTALL)
  open('/tmp/check_js.js', 'w').write('\n'.join(scripts))
  " && node --check /tmp/check_js.js && echo "JS OK"
  ```
