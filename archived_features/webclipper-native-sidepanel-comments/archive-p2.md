# Archive P2 - webclipper-native-sidepanel-comments (per-tab AI tab + tab group)

> 状态：**已解除归档 / 已恢复推进**（crh2，2026-03-26）。见 `plan-p2.md`。
>
> 备注：本文件保留“归档时的记录”；当前实现计划与验收口径以 `plan-p2.md` 为准。

## 目标（归档记录）

当 P1 的 iframe 方案不可用时，提供可落地的降级：为每个内容 tab 创建/复用一个 AI tab，并做 `contentTabId -> aiTabId` 映射，尽量把两个 tab 放到一起（move + tab group），触发时聚焦 AI tab（open-only）。

## 关键点（归档记录）

- 不创建新 window（只用 tab）。
- 不做真同屏（同屏只能靠 P1 iframe 或未来 layout two-column）。
- 维护 `contentTabId -> aiTabId` 映射（优先 `chrome.storage.session`）。
- 通过 `tabs.move` 尽量紧挨内容 tab；若支持 `tabs.group` 则放到同一 group。
- 通过 `tabs.onRemoved` / `tabs.onUpdated` 做映射清理与解绑。
