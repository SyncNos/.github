# P3 - Merge Article Conversation On URL Conflict

## Goal

当用户把某条 `article` 的 URL 手动改成“另一条已存在的 article URL”时，除了合并评论线程（canonicalUrl），还要把**会话本体也去重合并**（列表只剩 1 条 article）。

## Approach

- 保存 URL 后若检测到冲突且用户确认：
  - 继续执行评论 canonicalUrl 迁移（已实现）
  - 合并 conversations：
    - 保留“当前正在编辑的这条 conversation”（保持 `source + conversationKey` 不变）
    - 将冲突那条 conversation 的可用元数据（如 `notionPageId`、`warningFlags`、`lastCapturedAt`）合并进来
    - 将冲突那条的 `sync_mappings` 迁移/合并到保留目标（避免 Notion 增量状态丢失）
    - 将冲突那条的 messages/image_cache conversationId 迁移到保留目标（尽量不丢数据）
    - 删除冲突那条 conversation

---

## P3-T1

### 任务：新增 MERGE_CONVERSATIONS core message + IDB 合并实现

### 修改范围（预期文件）

- `webclipper/src/platform/messaging/message-contracts.ts`
- `webclipper/src/services/conversations/background/handlers.ts`
- `webclipper/src/services/conversations/client/repo.ts`
- `webclipper/src/services/conversations/data/storage.ts`
- `webclipper/src/services/conversations/data/storage-idb.ts`

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task1 - URL 冲突时合并文章会话`

---

## P3-T2

### 任务：URL 冲突确认后执行会话合并（viewmodel）

### 修改范围（预期文件）

- `webclipper/src/viewmodels/conversations/conversations-context.tsx`

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task2 - 冲突确认后去重合并 article`

---

## P3-T3

### 任务：补齐单测（IDB merge）

### 修改范围（预期文件）

- `webclipper/tests/storage/conversations-idb.test.ts`

### 验证

- `npm --prefix webclipper run test -- tests/storage/conversations-idb.test.ts`

### 提交要求

- `test: task3 - 覆盖会话合并`

