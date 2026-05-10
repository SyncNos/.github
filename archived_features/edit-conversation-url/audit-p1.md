# Audit P1 - Edit Conversation URL (MVP)

## Checklist

- [x] URL 点击入口仅进入编辑态，不会意外触发打开/跳转
- [x] canonical 化规则符合预期：trim + http/https + 去 hash
- [x] 更新 URL 时不会把 `sourceType` 覆盖为 `chat`（必须携带现有 `sourceType`）
- [x] 更新 URL 时不会意外把 `lastCapturedAt` 改成 `now`（避免列表排序抖动）
- [x] 本 feature 不主动修改 `conversationKey`；但确认后续 `article` 重新抓取触发的既有迁移不会导致数据丢失（sync_mappings / notionPageId）
- [x] 冲突检测准确：仅 article、canonicalUrl 相同且 id 不同
- [x] 冲突 confirm：取消不落库；确认后继续并合并评论
- [x] 评论迁移覆盖 reply 场景，迁移后新 URL 下可见，旧 URL 下不再分裂
- [x] 清理参数按钮只回填 input，不自动保存

## Evidence

- 验证命令：
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper run test -- tests/storage/article-comments-idb.test.ts`
- 关键实现点（供回看）：
  - URL 更新 + 冲突确认 + 评论迁移主流程：`webclipper/src/viewmodels/conversations/conversations-context.tsx`
  - 宽屏 header URL 编辑：`webclipper/src/ui/conversations/ConversationDetailPane.tsx`
  - 窄屏 header URL 编辑：`webclipper/src/ui/conversations/DetailNavigationHeader.tsx`
  - comments canonicalUrl 迁移：`webclipper/src/services/comments/data/storage-idb.ts`

## Follow-ups

- URL 编辑 UI（宽/窄屏）存在重复实现；如后续继续扩展（例如 Copy/Open 按钮），建议抽一个共享组件避免漂移。
