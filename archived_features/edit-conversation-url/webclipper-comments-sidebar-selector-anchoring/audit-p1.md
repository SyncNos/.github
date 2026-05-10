# Audit P1 - webclipper-comments-sidebar-selector-anchoring

## Checklist

- [x] Locator 数据结构是否稳定可扩展（版本字段、env 边界清晰）
- [x] “只对 root comment 生效、reply 不触发”的交互是否严格
- [x] 是否会误滚 sidebar 自己（anchor root 选择是否正确）
- [x] open-only / close-only-by-header 语义是否保持
- [x] 失败提示是否轻量且不抢焦点

## Notes

- Locator：`v=1 + env` 清晰；还原优先走 position selector（确定性更强），失败才 fallback 到 quote selector + hint。
- Root/reply：定位 click handler 仅绑定在 root thread 的 quote/comment 容器上；reply 不绑定点击定位。
- Anchor root：
  - inpage：`document.body || document.documentElement`（不穿透 shadow DOM）
  - app：`document.querySelector('.route-scroll')`，避免 sidebar 内误匹配；缺失时直接失败并提示
- UX：失败提示为 panel 内 `notice`，自动消失，不抢输入焦点。
- 验证：
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper test`
