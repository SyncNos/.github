# P1 - Edit Conversation URL (MVP)

## Goal

在 WebClipper 的 conversations detail header 中提供“就地编辑 URL”的能力，并保证 `article` 评论按新 URL 继续可见（迁移/合并旧 canonicalUrl 下的评论）。

## Non-goals

- 本 feature 的“手动编辑 URL”流程不主动重写 `conversationKey`（但已知后续 `article` 重新抓取可能基于 URL 触发既有 `conversationKey` 迁移逻辑；见 audit）
- 不改变现有 Open actions 的语义（点击 URL 文本只负责进入编辑）
- 不做全局 URL normalize 逻辑重构

## Approach（高层）

- UI：把 detail header 第二行从 `source · conversationKey` 替换为 URL 文本；点击进入编辑态 input。
- 保存：调用 `UPSERT_CONVERSATION` 更新 `conversation.url`（保持 `source + conversationKey` 不变），并且 **必须携带 `sourceType`** 防止把 `article` 错写成 `chat`。
- 保存：更新 URL 时建议同时携带当前 `lastCapturedAt`，避免无意把“最近捕获时间/列表排序”更新为 `now`。
- 评论：新增 comments 侧的 `migrateArticleCommentsCanonicalUrl(old, next)`，把 `article_comments` 表中旧 canonicalUrl 的记录更新为新 canonicalUrl，实现“迁移/合并线程”。
- 冲突：若新 canonicalUrl 已存在于另一条 `article` conversation，保存前 `confirm` 提示用户；确认后继续并执行评论合并（不做会话合并/删除）。
- 工具：编辑态提供 `清理参数` 按钮，调用 `cleanTrackingParamsUrl()` 回填 input（不自动保存）。

## Acceptance（P1）

- 宽屏与窄屏 header 都显示 URL 行且可进入编辑态
- 保存成功后 URL 持久化，且 `article` 评论不丢
- 编辑 URL 不应无意修改 `sourceType` / `lastCapturedAt`
- 冲突场景有确认提示，确认后评论合并

---

## P1-T1

### 任务：新增评论 canonicalUrl 迁移/合并能力（background+client+idb）

### 修改范围（预期文件）

- `webclipper/src/platform/messaging/message-contracts.ts`
- `webclipper/src/services/comments/background/handlers.ts`
- `webclipper/src/services/comments/client/repo.ts`
- `webclipper/src/services/comments/data/storage.ts`
- `webclipper/src/services/comments/data/storage-idb.ts`

### 实现步骤

1. 在 `COMMENTS_MESSAGE_TYPES` 增加一个新消息类型，例如 `MIGRATE_ARTICLE_COMMENTS_CANONICAL_URL`
2. comments background handler 注册该消息：
   - 输入：`fromCanonicalUrl`, `toCanonicalUrl`
   - 输出：`{ moved: number }`（或 `{ updated: number }`）
3. comments storage-idb 新增迁移函数：
   - 规范化 from/to（http/https + 去 hash）
   - 若两者相同直接返回 `0`
   - 使用 `article_comments` 的 index `by_canonicalUrl_createdAt` 做范围扫描，逐条 `put` 更新 canonicalUrl（避免全表扫描）
   - 建议同步更新 `updatedAt`（保持 `createdAt` 不变）
   - `conversationId` 字段保持原值（本 feature 的“合并”以 canonicalUrl 作为主线程 key；后续如需要按会话合并 conversationId 再单独规划）
4. client repo 增加对应调用方法供 UI 使用
5. 迁移成功后可广播 `UI_EVENT_TYPES.CONVERSATIONS_CHANGED`（reason 例如 `articleCommentsMigrated`），用于触发 UI 刷新

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task1 - 支持文章评论 canonicalUrl 迁移`

---

## P1-T2

### 任务：新增 upsertConversation 客户端调用用于更新 url

### 修改范围（预期文件）

- `webclipper/src/services/conversations/client/repo.ts`

### 实现步骤

1. 在 client repo 增加 `upsertConversation(payload)` 方法，发送 `CORE_MESSAGE_TYPES.UPSERT_CONVERSATION`
2. 确保返回 unwrap 后的数据含 `id/source/conversationKey/url`
3. 调用方约束（写在实现注释/类型中即可）：用于“更新 URL”时，payload 至少要携带：
   - `source`
   - `conversationKey`
   - `sourceType`（关键：避免覆盖为 `chat`）
   - `lastCapturedAt`（建议：保持稳定，不要因为编辑 URL 重排列表）
   - `url`

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task2 - 提供会话 upsert 客户端接口`

---

## P1-T3

### 任务：宽屏 detail header：URL 文本展示 + 点击进入编辑态 + Enter/Esc

### 修改范围（预期文件）

- `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- （可选）新增：`webclipper/src/ui/conversations/ConversationUrlInlineEditor.tsx`

### 实现步骤

1. 将 header 第二行由 `source · conversationKey` 替换为 URL 行（无 URL 时显示占位文案）
2. URL 行渲染为“可点击文本按钮”（点击进入编辑态）
3. 编辑态展示 input：
   - 初值为当前 `conversation.url`（或 canonical 形态）
   - `Enter`：触发保存
   - `Esc`：取消并退出编辑态
4. 保存时做最小 canonical 化（http/https + trim + 去 hash）
5. 保存成功后退出编辑态并刷新 UI（依赖既有 eventsHub + conversationsContext 的刷新逻辑）
6. 建议把“保存、冲突确认、评论迁移”的主流程收敛到 `conversations-context.tsx` 的一个方法里（例如 `updateSelectedConversationUrl(nextUrl)`），避免宽/窄屏重复实现与语义漂移

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task3 - detail header 支持就地编辑 URL`

---

## P1-T4

### 任务：窄屏 DetailNavigationHeader：URL 替换 subtitle + 支持编辑入口

### 修改范围（预期文件）

- `webclipper/src/ui/conversations/DetailNavigationHeader.tsx`
- `webclipper/src/ui/conversations/ConversationsScene.tsx`
- `webclipper/src/ui/popup/PopupShell.tsx`（如需透传）
- `webclipper/src/ui/app/AppShell.tsx`（如需透传）

### 实现步骤

1. `ConversationsScene` 计算 header state 时，将 `subtitle` 替换为 `selectedConversation.url`（canonical 形式）
2. `DetailNavigationHeader` 的 subtitle 区域改为可交互并支持进入编辑态：
   - 推荐：抽一个可复用组件（例如 `ConversationUrlHeaderLine`），内部用 `useConversationsApp()` 读取/更新 URL；宽屏与窄屏共用同一套保存/迁移/确认逻辑
   - 避免把 `ReactNode` 存进 `PopupShell/AppShell` 的 headerState（减少状态序列化/刷新复杂度）
3. 确保窄屏返回按钮与 actions 布局不受影响；编辑态的键盘事件（Enter/Esc）不应触发返回/菜单

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task4 - 窄屏 header 使用 URL 替换 subtitle 并支持编辑`

---

## P1-T5

### 任务：冲突检测与确认：新 canonicalUrl 已存在时提示并合并评论

### 修改范围（预期文件）

- `webclipper/src/viewmodels/conversations/conversations-context.tsx`
- `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- `webclipper/src/ui/conversations/DetailNavigationHeader.tsx`

### 实现步骤

1. 保存前在 UI 侧基于 `items` 做冲突检测：
   - 仅 `sourceType=article`
   - canonicalUrl 相同且 `id` 不同
2. 若冲突：
   - 弹出 `confirm`：提示“该 URL 已存在于另一条文章，会将评论合并到该 URL，是否继续？”
   - 用户取消：不落库、不迁移
3. 若继续：
   - 先更新当前 conversation 的 `url`（必须携带 `sourceType`）
   - 再执行 `migrateArticleCommentsCanonicalUrl(oldCanonical, newCanonical)`（等价于合并线程）
   - 注意：该操作只“合并评论线程”，不会自动合并/删除重复 conversation；计划外如需合并会话需另开 phase

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task5 - URL 冲突确认并合并评论线程`

---

## P1-T6

### 任务：编辑态工具：清理参数按钮（cleanTrackingParamsUrl）

### 修改范围（预期文件）

- `webclipper/src/services/url-cleaning/tracking-param-cleaner.ts`（复用，无需改动；如需导出调整再改）
- `webclipper/src/ui/conversations/ConversationDetailPane.tsx` / `ConversationUrlInlineEditor.tsx`

### 实现步骤

1. 编辑态增加 `清理参数` 按钮
2. 点击后对当前 input 值调用 `cleanTrackingParamsUrl(value)`：
   - 得到 cleaned 后回填 input（不自动保存）
   - 若失败：提示错误，不修改 input
3. 回填后再执行一次最小 canonical 化（去 hash / trim）

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task6 - URL 编辑支持一键清理跟踪参数`

---

## P1-T7

### 任务：补齐单测：comments idb 迁移/合并用例

### 修改范围（预期文件）

- `webclipper/tests/storage/article-comments-idb.test.ts`

### 实现步骤

1. 为迁移函数新增测试用例：
   - old canonicalUrl 下有多条评论（含 reply）
   - migrate 到 new canonicalUrl 后，新 url 下可查询到原评论
   - old url 下查询为空
   - 迁移到已存在评论的新 url 时应“合并”（数量为两边之和）

### 验证

- `npm --prefix webclipper run test -- tests/storage/article-comments-idb.test.ts`

### 提交要求

- `test: task7 - 覆盖文章评论 canonicalUrl 迁移`

---

## P1-T8

### 任务：验证链：compile + targeted tests

### 验证命令（按顺序）

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/storage/article-comments-idb.test.ts`

### 提交要求

- 本 task 只跑验证命令与记录结果（可在 `todo.toml` 的 `note` 中记录），不强制代码提交

---

## Phase Audit（P1 结束后写入 audit-p1.md）

- URL 保存逻辑是否只修改 `url` 字段且不会把 `sourceType` 覆盖为 `chat`
- 本 feature 不主动改 `conversationKey`；但需确认后续 `article` 重新抓取触发的既有迁移不会导致数据丢失（sync_mappings / notionPageId）
- 冲突确认是否覆盖“取消不落库”的路径
- 评论迁移是否真正按 canonicalUrl 合并（含 reply）
- `清理参数` 是否仅回填 input，不隐式保存
