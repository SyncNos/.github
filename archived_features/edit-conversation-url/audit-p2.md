# Audit P2 - Optional UX Enhancements

> 2026-03-24：按反馈移除了 URL 行右侧的打开/复制按钮，保留“点击 URL 文本进入编辑”的入口。

## Checklist

- [x] URL 行不再提供右侧按钮（更干净，避免误触）
- [x] URL 文本点击样式更隐蔽（无 hover underline，接近原本“不可点击”的表现）

## Evidence

- 实现：
  - 宽屏 header URL 行：`webclipper/src/ui/conversations/ConversationDetailPane.tsx`
  - 窄屏 header URL 行：`webclipper/src/ui/conversations/DetailNavigationHeader.tsx`
- 验证命令：
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper run test -- tests/storage/conversations-idb.test.ts`
