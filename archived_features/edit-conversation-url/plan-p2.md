# P2 - Optional UX Enhancements

## Goal

在不改变“点击 URL 文本进入编辑”的前提下，补齐一些更易发现/更顺手的操作（打开链接、复制 URL）。

> 2026-03-24：按反馈移除了 URL 行右侧的打开/复制按钮；当前仅保留“点击 URL 文本进入编辑”的入口。本文件保留作为历史记录。

## P2-T1

### 任务：URL 行增加外链小按钮（打开链接，点击文本仍为编辑）

### 修改范围（预期文件）

- `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- `webclipper/src/ui/conversations/DetailNavigationHeader.tsx`
- （可选）复用 `@services/integrations/open-external-url`

### 实现步骤

1. URL 行右侧增加一个小 icon button（↗ 或 ExternalLink icon）
2. 点击该按钮调用 `openExternalUrl(url)` 打开链接
3. 无 URL / 非 http(s) 时按钮应 disabled（避免误触后才报错）
4. 保持点击 URL 文本仍是进入编辑态

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task1 - URL 行增加外链打开按钮`

---

## P2-T2

### 任务：增加复制 URL 的轻量操作

### 修改范围（预期文件）

- `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- `webclipper/src/ui/conversations/DetailNavigationHeader.tsx`

### 实现步骤

1. URL 行增加“复制”小按钮（或在编辑态提供 Copy）
2. 复用现有 clipboard 写入逻辑（可参考 `chatwith-detail-header-actions.ts`，建议封装为 `@ui/shared/clipboard` 以供复用）

### 验证

- `npm --prefix webclipper run compile`

### 提交要求

- `feat: task2 - 支持复制会话 URL`

---

## P2-T3

### 任务：加固 UPSERT_CONVERSATION 合并规则（sourceType/lastCapturedAt 默认继承 existing）

### 背景

当前 `upsertConversation`（IDB 层）在 payload 缺少 `sourceType` 时会默认写入 `chat`，且缺少 `lastCapturedAt` 时会写入 `now`。这对“只想更新 URL/少量字段”的调用方是一个容易踩坑的 footgun。

### 修改范围（预期文件）

- `webclipper/src/services/conversations/data/storage-idb.ts`

### 实现步骤

1. 将 `sourceType: payload.sourceType || 'chat'` 改为优先继承 existing：
   - `sourceType: payload.sourceType || (existing ? existing.sourceType || 'chat' : 'chat')`
2. 将 `lastCapturedAt: payload.lastCapturedAt || now` 改为优先继承 existing：
   - `lastCapturedAt: payload.lastCapturedAt || (existing ? existing.lastCapturedAt || now : now)`
3. 确保现有 capture 流程（会显式提供 `sourceType/lastCapturedAt`）不受影响

### 验证

- `npm --prefix webclipper run compile`
- （可选）跑 `npm --prefix webclipper run test -- tests/storage/conversations-idb.test.ts`

### 提交要求

- `fix: task3 - 加固会话 upsert 的默认合并规则`

---

## Phase Audit（P2 结束后写入 audit-p2.md）

- 点击文本 vs 点击按钮的命中区域是否清晰且不会误触
- 无 URL 时按钮是否正确 disabled
