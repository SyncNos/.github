# Audit P1 - webclipper-chatwith-entry-in-comments-sidebar

> 由 `executing-plans` 在 P1 完成后填写审计记录与修复回填。

## Checklist

- [x] 旧入口清理：`app/popup` 详情页 header actions 不再出现 `Chat with...`
- [x] 新入口一致：`app` 与 `inpage` 的 comments sidebar header 都有 `Chat with...`
- [x] 功能一致：复制 payload + 打开新 tab 语义与旧入口一致（静态链路审查通过）
- [x] Shadow DOM 稳定：菜单布局、点击外部关闭、键盘可用性（Esc）已覆盖
- [x] 失败路径：无 enabled 平台 / 无 detail / runtime 不可用时有可理解提示

## Findings

1. **失败路径提示不明确（已修复）**
   - 现象：`app/inpage` resolver 之前将多类失败都归并为 `[]`，UI 会落到空态文案，不易区分“未启用平台”与“detail/runtime 失败”。
   - 修复：在 resolver 中对 `detail/runtime/resolve` 失败显式抛错，由菜单 error 状态显示具体原因。
   - 修复提交：`9e25db70`

## Validation

- `npm --prefix webclipper run compile`：通过
- 手工冒烟：待执行（需在真实页面验证剪贴板写入与新 tab 打开）
