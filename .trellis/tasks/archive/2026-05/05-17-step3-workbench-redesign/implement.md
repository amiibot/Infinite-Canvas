# Implement：Step3 资产弹窗工作台改造

## 执行顺序

### Step 1：CSS — 弹窗容器 & 左侧面板
文件：`static/project.html`，CSS 块

- [ ] `.asset-modal` max-width: 900px → 1040px
- [ ] `.asset-modal-left` width: 55% → 65%，新增 flex-direction:column, align-items:stretch
- [ ] `.asset-modal-right` width: 45% → 35%
- [ ] `#assetModalImg`（图片容器）新增 flex:1, overflow:hidden（通过 refreshAssetModalImage 内联样式或 CSS）
- [ ] `.variant-strip`（弹窗内）新增 flex-shrink:0, border-top

### Step 2：HTML — 变体条迁移
- [ ] 将 `<div class="variant-strip" id="variantStrip"></div>` 从
  `.asset-modal-left` 和 `.asset-modal-right` 之间，移入 `.asset-modal-left` 内部末尾

### Step 3：HTML — 右侧参数区重排
- [ ] 将 `#assetModalCharTabs` 移到 `.asset-modal-body` 最顶部
- [ ] 将 `#assetModalPromptSection` 移到 `#assetModalCharTabs` 之后
- [ ] 将描述区包裹为折叠面板（新增 `#assetModalDescToggle` + `#assetModalDescBody`）

### Step 4：JS — 描述折叠逻辑
- [ ] `state` 初始化新增 `modalDescOpen: false`
- [ ] 新增 `toggleAssetModalDesc()` 函数
- [ ] `openAssetModal` 中初始化 `state.modalDescOpen = false`，同步 body/arrow 显示

### Step 5：CSS — 主按钮强化
- [ ] `#assetModalRegenBtn` 高度 38px → 42px，font-size 13px → 14px

## 验证命令

```bash
# 语法检查（无构建工具，用 HTML 结构验证）
python3 -c "
from html.parser import HTMLParser
class D(HTMLParser):
    def __init__(self):
        super().__init__()
        self.depth = 0
        self.max_depth = 0
    def handle_starttag(self, tag, attrs):
        void = {'area','base','br','col','embed','hr','img','input','link','meta','param','source','track','wbr'}
        if tag not in void:
            self.depth += 1
            self.max_depth = max(self.max_depth, self.depth)
    def handle_endtag(self, tag):
        void = {'area','base','br','col','embed','hr','img','input','link','meta','param','source','track','wbr'}
        if tag not in void:
            self.depth -= 1
p = D()
p.feed(open('static/project.html').read())
print('Final depth:', p.depth, '| Max depth:', p.max_depth)
assert p.depth == 0, 'HTML depth imbalance!'
print('OK')
"
```

```bash
# 确认关键元素顺序
python3 -c "
import re
html = open('static/project.html').read()
positions = {
    'charTabs': html.find('id=\"assetModalCharTabs\"'),
    'promptSection': html.find('id=\"assetModalPromptSection\"'),
    'descSection': html.find('id=\"assetModalDescSection\"'),
    'styleSection': html.find('id=\"assetModalStyleSection\"'),
    'advancedSection': html.find('id=\"assetModalAdvancedSection\"'),
    'variantStrip_in_left': html.find('id=\"variantStrip\"'),
}
for k,v in positions.items(): print(k, v)
assert positions['charTabs'] < positions['promptSection'], 'charTabs must be before promptSection'
assert positions['promptSection'] < positions['descSection'], 'promptSection must be before descSection'
assert positions['descSection'] < positions['styleSection'], 'descSection must be before styleSection'
print('Order OK')
"
```

## 风险点

| 风险 | 说明 | 缓解 |
|------|------|------|
| 变体条迁移后高度溢出 | 左侧 flex column 时变体条可能撑高 | 给 variant-strip 加 max-height + overflow-x:auto |
| 描述折叠后 setAssetModalEditing 失效 | 该函数操作 #assetModalDescView/Edit | 确认两个元素仍在 DOM，仅外层包裹变化 |
| 右侧 35% 宽度过窄 | 提示词 textarea 可能太窄 | 测试时确认 min-width 或调整到 36% |

## 回滚

```bash
git checkout static/project.html
```
