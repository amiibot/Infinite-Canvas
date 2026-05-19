# char-workbench Implement

## Checklist

### HTML
- [ ] 1. 在 `.asset-modal` 内新增 `#charWorkbench` 容器（3列结构）

### CSS
- [ ] 2. 新增 `.char-workbench` 及子元素样式
- [ ] 3. 角色模式下 `.asset-modal` 宽度改为 `max-width: 1280px`

### JS
- [ ] 4. 新增 `renderColumnVariantStrip(stripEl, variants, selectedId)` 通用函数
- [ ] 5. 新增 `openCharWorkbench(asset)` 函数（填充3列）
- [ ] 6. 修改 `openAssetModal`：角色时切换到工作台布局
- [ ] 7. 每列生成按钮事件绑定
- [ ] 8. 每列变体条事件委托（select/delete/favorite）
- [ ] 9. 关闭按钮事件

### 验证
- [ ] 10. 启动服务器，打开角色弹窗确认3列显示
- [ ] 11. 打开场景/道具弹窗确认无回归
