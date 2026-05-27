# Journal - ami (Part 1)

> AI development session journal
> Started: 2026-05-16

---



## Session 1: ConsistencyVault accessibility + bug 全量修复

**Date**: 2026-05-16
**Task**: ConsistencyVault accessibility + bug 全量修复
**Branch**: `main`

### Summary

基于 Codex review 对 ConsistencyVault.tsx 和 static/project.html 做全量修复：补 api/crudApi import（修复运行时崩溃）、setInterval 轮询加卸载清理、showStatus timer 竞态修复、videoPrompt stale state 修复、错误 banner 改 role=alert、Tab 组加 ARIA 语义、空状态加创建 CTA、AssetCard button 嵌套重构、hover-only 控件加 focus-within、modal 加 dialog 语义和焦点管理

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `7e972d5` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 2: Step3 资产弹窗工作台改造

**Date**: 2026-05-17
**Task**: Step3 资产弹窗工作台改造
**Branch**: `main`

### Summary

将资产弹窗改造为工作台风格：左侧 65/35 布局扩大预览区，变体条移入左侧底部，右侧参数区重排（角色 tab → 提示词 → 描述折叠 → 风格 → 高级），Regenerate 按钮视觉强化。同步修复 3 个 JS bug（syncCharTabs 时序、铅笔按钮折叠状态、headshot variant 路由）及 6 项 a11y 问题（aria-expanded 全路径同步、label 关联、prefers-reduced-motion、role=button 语义化）。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `5d8be51` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 3: 角色双面板数据隔离

**Date**: 2026-05-17
**Task**: 角色双面板数据隔离
**Branch**: `main`

### Summary

修复 Full Body / Headshot 共用 variants 数组的 bug。后端已有 _get_variant_fields 路由逻辑（headshot_variants/headshot_selected_variant_id/headshot_prompt），本次补充前端 getAssetCollection 缺少 character_headshot → characters 映射，修复 Headshot tab 下变体操作后弹窗无法刷新的问题。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `ab66039` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 4: Step3 资产弹窗增强：完成 trellis-check 并修复 AC4

**Date**: 2026-05-17
**Task**: Step3 资产弹窗增强：完成 trellis-check 并修复 AC4
**Branch**: `main`

### Summary

trellis-check 发现 AC4 缺口：openAssetModal 和 syncAssetModalCharTabs 未在前端注入 STRICTLY MAINTAIN 前缀（design.md 要求打开弹窗时自动注入，实现只在后端重生成时注入）。修复两处，检测 is_uploaded_source 变体后填入前缀。任务 05-16-step3-asset-modal-enhancement 全部 AC 通过，已归档。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `890d014` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 5: 异步任务系统：生成接口后台化 + 前端轮询 + 取消机制

**Date**: 2026-05-18
**Task**: 异步任务系统：生成接口后台化 + 前端轮询 + 取消机制
**Branch**: `main`

### Summary

将 generate_assets / regenerate_asset / render_frame 三个耗时接口改为异步模式：后端立即返回 _task_id，_bg_* 协程后台执行并更新 _ASYNC_TASKS；前端新增 pollTask 通用轮询工具（2s 间隔，12min 超时），三个调用点全部适配，生成中显示进度和取消按钮。Codex review 后修复：CharacterWorkbench 调用方适配异步响应、轮询超时从 5min 延长至 12min、.gitignore 补充 other/**/.env 规则。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `97a9641` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 6: Step2 AI 风格推荐 + 用户自定义预设

**Date**: 2026-05-18
**Task**: Step2 AI 风格推荐 + 用户自定义预设
**Branch**: `main`

### Summary

新增 POST /api/comic/projects/{pid}/art_direction/analyze，LLM 根据剧本推荐 3 种视觉风格并严格校验格式；Step2 新增 AI 推荐折叠区（details/summary）、用户自定义预设（localStorage 持久化，支持保存/删除），预设选中逻辑扩展至内置/自定义/AI 三类来源。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `0bf4335` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 7: Step2 per-project 模型配置 + aspect_ratio 持久化

**Date**: 2026-05-18
**Task**: Step2 per-project 模型配置 + aspect_ratio 持久化
**Branch**: `main`

### Summary

新增 ComicArtDirectionSave.image_model / aspect_ratio 字段；三处生成接口（generate_assets/regenerate_asset/render_frame）优先读项目 art_direction.image_model，空时 fallback 全局默认；loadProject 并行请求 /api/config（Promise.allSettled），config 失败不影响项目加载；hydrateProject 恢复 selectedImageModel / aspectRatio；Step2 新增模型下拉（#imageModelSelect）；handleApplyStyle POST 时带上 image_model 和 aspect_ratio。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `903080c` | (see git log) |
| `f9970b2` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 8: P2 三任务：sync_descriptions / reverse-gen-ui / preset-inline-input

**Date**: 2026-05-18
**Task**: P2 三任务：sync_descriptions / reverse-gen-ui / preset-inline-input
**Branch**: `main`

### Summary

实现三个 P2 功能：(1) sync_descriptions — POST /api/comic/projects/{pid}/sync_descriptions，LLM 批量补全角色/场景描述，后台异步 + 轮询；(2) reverse-gen-ui — 资产弹窗左侧图片区叠加 Upload Detected 徽章 + 参考图缩略图；(3) preset-inline-input — Step2 自定义预设保存改为页面内联输入框，移除 window.prompt。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `7f67833` | (see git log) |
| `5769dcf` | (see git log) |
| `885f58b` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 9: 网页测试 + fix(modal): refImageBadge 被 innerHTML 覆盖

**Date**: 2026-05-18
**Task**: 网页测试 + fix(modal): refImageBadge 被 innerHTML 覆盖
**Branch**: `main`

### Summary

对项目 9e8d0c72（缩小者）进行全流程网页测试：Step1 脚本解析、Step2 风格配置（预设内联输入框保存验证）、Step3 资产弹窗。发现并修复 #refImageBadge 被 refreshAssetModalImage 的 innerHTML 覆盖导致徽章消失的 bug；修复方案：将徽章节点移至 .asset-modal-left 层级，并加 position:relative，使其不受 #assetModalImg 的 innerHTML 重写影响。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `978dbf5` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 10: Step1 实体内联编辑 UI

**Date**: 2026-05-18
**Task**: Step1 实体内联编辑 UI
**Branch**: `main`

### Summary

为 Step1 侧边栏每个实体 chip 增加 ✎ 编辑按钮，展开内联表单（角色含 age/gender/clothing，场景/道具只含 name/description），预填当前值，保存调用 PATCH 接口。后端 AssetDescriptionUpdate 扩展 name/age/gender/clothing 字段。修复 Codex P2：编辑/添加表单互斥逻辑（点击添加时先重渲染侧边栏）。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `92219ad` | (see git log) |
| `aeb48c6` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 11: Seedance prompt token 解析 + 音频端口 + review 修复

**Date**: 2026-05-19
**Task**: Seedance prompt token 解析 + 音频端口 + review 修复
**Branch**: `feat/library-as-workspace-sibling`

### Summary

实现 Seedance 节点 prompt 中 [图片N]/[视频N]/[音频N] token 解析，新增独立 in-audio 端口和素材预览条，修复 Codex review 的 5 个问题（音频 fallback 移除、URL 去重、toast 提示、模式切换清理）

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `d4387bd` | (see git log) |
| `33428be` | (see git log) |
| `a6d0835` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 12: Step4 多参考图分镜生成

**Date**: 2026-05-20
**Task**: Step4 多参考图分镜生成
**Branch**: `feat/remove-step5-step6`

### Summary

实现分镜渲染自动收集帧级参考图：改造 analyze_storyboard LLM prompt 输出实体关联，后端按名字映射为 ID；新增 _collect_frame_refs 辅助函数；修复 generate_ai_image [:4]→[:16]

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `fee4bae` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 13: UI合规修复 P0+P1（#1-#9）

**Date**: 2026-05-20
**Task**: UI合规修复 P0+P1（#1-#9）
**Branch**: `feat/remove-step5-step6`

### Summary

修复 Web Interface Guidelines 审查中 P0-P1 问题：login.html viewport/label/autocomplete、全部图标按钮 aria-label、div onclick→button、表单控件 label、theme.css 全局 focus-visible + prefers-reduced-motion、transition:all→具体属性（18文件70+处）、JS 键盘支持（history-bulk-manager + image-preview）

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `0346ff1` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 14: UI 可访问性修复 P1+P2（#10-#19,#24）

**Date**: 2026-05-20
**Task**: UI 可访问性修复 P1+P2（#10-#19,#24）
**Branch**: `feat/remove-step5-step6`

### Summary

修复 select aria-label、getBoundingClientRect 缓存、modal overscroll-behavior、touch-action、img width/height+loading=lazy、省略号 Unicode 统一、color-scheme 声明、project.html head 闭合。新增前端 UI a11y spec。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `57ca525` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete


## Session 15: 角色工坊参考图反向生成收尾

**Date**: 2026-05-26
**Task**: 角色工坊参考图反向生成收尾
**Branch**: `fix/gpt-image-2-reference-edits`

### Summary

完成参考图反向生成链路修复、导演工坊文本展示修复与相关前端规范沉淀。

### Main Changes

(Add details)

### Git Commits

| Hash | Message |
|------|---------|
| `545131d` | (see git log) |
| `3f7f2bf` | (see git log) |

### Testing

- [OK] (Add test results)

### Status

[OK] **Completed**

### Next Steps

- None - task complete
