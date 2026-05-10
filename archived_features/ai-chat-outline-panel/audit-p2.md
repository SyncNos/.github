# Audit P2 - ai-chat-outline-panel

> 由 `executing-plans` 在 P2 完成后进入本文件，记录问题 -> 修复 -> 回归验证。

## Checklist

- [x] 自动高亮正确：高亮始终对应“最接近视口中线”的 user 消息
- [x] app + popup、宽屏/窄屏都可用（root=`.route-scroll` 时精度正常）
- [x] 大对话滚动不卡顿（避免每次 scroll 全量测量）
- [x] `npm --prefix webclipper run compile && npm --prefix webclipper run test` 通过

## Findings

- [medium] `useChatOutlineActiveIndex.ts` 在 scroll 重算时仍全量扫描 `safeUserMessageEls`，大对话下有性能风险（subagent 审计发现）。

## Fixes

- 用 `fallbackIndexByEl` + 迭代 `visibleSet` 替换每次 scroll 的全量 `filter`，仅在必要时才构造全量候选。
- 修复提交：`5ce9e06345cd6a6a076fae7773afb0b0e5af892c`

## Verification

- `npm --prefix webclipper run compile && npm --prefix webclipper run test` ✅
