# Design: Seedance Prompt 引用 Token 解析

## Architecture Overview

所有改动集中在 `static/canvas.html`，纯前端 JS。无后端改动。

```
┌─────────────────────────────────────────────────┐
│  renderSeedanceBody(node)                       │
│  ┌───────────────────────────────────────────┐  │
│  │ [素材预览条] 图片1 | 图片2 | 视频1        │  │  ← NEW: media preview row
│  ├───────────────────────────────────────────┤  │
│  │ [prompt textarea]                         │  │
│  │  "一只猫[图片1]在跑步，参考[视频1]风格"    │  │
│  └───────────────────────────────────────────┘  │
│  [模式切换] [模型/比例/分辨率] [生成按钮]       │
└─────────────────────────────────────────────────┘
         │
         ▼ 点击"生成"
┌─────────────────────────────────────────────────┐
│  runSeedanceNode(nodeId)                        │
│  1. buildSeedanceTokenInventory(node)           │  ← NEW
│  2. parseSeedanceTokens(prompt, inventory)      │  ← NEW
│  3. 合并 matchedImageUrls → imagePayload        │
│  4. 合并 matchedVideoUrls → refVideos           │
│  5. 发送 cleanPrompt + 合并后的 images/videos   │
└─────────────────────────────────────────────────┘
```

## New Functions

### `buildSeedanceTokenInventory(node)`

构建当前节点的 token inventory。

```js
function buildSeedanceTokenInventory(node) {
    const sources = orderedSources(node, generatorSources(node));
    const imageRefs = sources.flatMap(s => s.refs || []);
    const images = imageRefs.map((ref, i) => ({
        labelNumber: i + 1,
        url: ref.url,
        name: ref.name || ''
    }));
    const videoItems = (node.referenceVideos || []).filter(v => v);
    const videos = videoItems.map((url, i) => ({
        labelNumber: i + 1,
        url
    }));
    return { images, videos };
}
```

### `parseSeedanceTokens(prompt, inventory)`

移植 Studio 的 `parseSeedanceReferenceTokens`，简化为纯 JS（去掉 TypeScript、去掉 audio）。

```js
function parseSeedanceTokens(prompt, inventory) {
    const TOKEN_RE = /\[(图片|视频)(\d+)\]/g;
    const matchedImageUrls = [];
    const matchedVideoUrls = [];
    const invalidTokens = [];
    let match;
    while ((match = TOKEN_RE.exec(prompt)) !== null) {
        const [, type, idx] = match;
        const num = Number(idx);
        const list = type === '图片' ? inventory.images : inventory.videos;
        const item = list.find(x => x.labelNumber === num);
        if (!item) { invalidTokens.push(match[0]); continue; }
        const target = type === '图片' ? matchedImageUrls : matchedVideoUrls;
        if (!target.includes(item.url)) target.push(item.url);
    }
    const cleanPrompt = prompt
        .replace(TOKEN_RE, ' ')
        .replace(/\s+([,，.。!！?？])/g, '$1')
        .replace(/\s{2,}/g, ' ')
        .trim();
    return { cleanPrompt, matchedImageUrls, matchedVideoUrls, invalidTokens };
}
```

### Media Preview Row（素材预览条）

在 `renderSeedanceBody` 中，prompt textarea **上方**插入预览条 HTML：

```html
<div class="sdn-media-preview" style="display:flex;gap:4px;overflow-x:auto;padding:4px 0;margin-bottom:4px">
  <!-- 动态生成 -->
  <div class="sdn-media-thumb" data-token="[图片1]" style="...">
    <img src="..." style="width:48px;height:36px;object-fit:cover;border-radius:3px">
    <span class="sdn-media-label">图片1</span>
    <span class="sdn-media-role">首帧</span>  <!-- first_last_frame 模式下 -->
  </div>
  <div class="sdn-media-thumb" data-token="[图片2]" style="...">
    <img src="..." style="width:48px;height:36px;object-fit:cover;border-radius:3px">
    <span class="sdn-media-label">图片2</span>
    <span class="sdn-media-role">尾帧</span>  <!-- first_last_frame 模式下 -->
  </div>
  <div class="sdn-media-thumb" data-token="[视频1]" style="...">
    <div style="...">🎬</div>
    <span class="sdn-media-label">视频1</span>
  </div>
</div>
```

**模式相关标注**：
- `first_last_frame` 模式：图片1 标注"首帧"，图片2 标注"尾帧"，其余无标注
- `multi_image` 模式：图片1 标注"主图"，其余无标注
- `single_image` 模式：无额外标注

标注以小字显示在编号标签下方，颜色用 `#f97316`（与 mode hint 一致）。

点击事件：在 prompt textarea 光标位置插入 `data-token` 值。

### `insertAtCursor(textarea, text)` 辅助函数

```js
function insertAtCursor(textarea, text) {
    const start = textarea.selectionStart;
    const end = textarea.selectionEnd;
    const before = textarea.value.substring(0, start);
    const after = textarea.value.substring(end);
    textarea.value = before + text + after;
    textarea.selectionStart = textarea.selectionEnd = start + text.length;
    textarea.dispatchEvent(new Event('input', {bubbles: true}));
}
```

### 预览条更新时机

预览条内容依赖连线（connections）。连线变化时需要刷新：

- 方案：在 `renderSeedanceBody` 中直接根据当前 connections 构建预览条
- 连线增删时，已有的 `refreshNode(node)` 会重新调用 `renderSeedanceBody`，预览条自然更新

验证：检查连线增删是否触发 Seedance 节点 re-render。

## Integration into `runSeedanceNode`

修改点（在现有 `const refs = ...` 之后）：

```js
// Token 解析
const inventory = buildSeedanceTokenInventory(node);
const tokenResult = parseSeedanceTokens(prompt, inventory);
const submissionPrompt = tokenResult.cleanPrompt || prompt;

// 合并 token 引用的图片到 imagePayload
if (tokenResult.matchedImageUrls.length) {
    const extraImages = tokenResult.matchedImageUrls
        .filter(url => !imagePayload.some(img => (img.url || img) === url))
        .map(url => ({url, name:'', role:'reference_image'}));
    imagePayload = [...imagePayload, ...extraImages];
}

// 合并 token 引用的视频到 refVideos
if (tokenResult.matchedVideoUrls.length) {
    const extraVideos = tokenResult.matchedVideoUrls.filter(url => !refVideos.includes(url));
    refVideos.push(...extraVideos);
}

// 发送时用 submissionPrompt 替代 prompt
```

注意：`refVideos` 需要从 `const` 改为 `let`。

## Edge Cases

1. **无连线输入** → inventory 为空 → 预览条不显示 → token 全部 invalid → cleanPrompt 移除 token → 正常发送
2. **token 引用的图片已在 imagePayload 中**（如 first_last_frame 模式的首帧）→ 去重，不重复添加
3. **连线删除后 token 失效** → 预览条更新（缩略图消失），生成时 token 被当作 invalid 移除
4. **prompt 无 token** → `parseSeedanceTokens` 返回空数组 → 行为与当前完全一致

## File Changes

| File | Change |
|------|--------|
| `static/canvas.html` | 新增 `buildSeedanceTokenInventory()` 函数 |
| `static/canvas.html` | 新增 `parseSeedanceTokens()` 函数 |
| `static/canvas.html` | `renderSeedanceBody()` 中添加素材预览条 HTML + 点击事件 |
| `static/canvas.html` | `runSeedanceNode()` 中集成 token 解析 + 合并逻辑 |
