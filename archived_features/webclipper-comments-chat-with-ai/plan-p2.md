# Plan P2 - webclipper-comments-chat-with-ai

**Goal:** comment 级 `Chat with AI` 在复制 payload 后，自动打开（或聚焦）到用户启用的平台首页 tab（仍由用户手动粘贴开聊）。

**Non-goals:**
- 本 phase 不做“精准上下文模板”（Phase 3 做）
- 本 phase 不做 tab group / 复用同一文章同一平台 tab（Phase 4 做）

**Approach:**
- 复制仍在 UI 侧执行；background 仅负责“打开/聚焦外部 tab”。
- 对 app/popup：沿用现有 `openExternalUrl()`/detail header 的“复制 + 打开”语义。
- 对 inpage：统一改为 background handler 执行 tab 打开，避免以 `window.open` 作为主路径。
- 安全约束：message 不直接接收任意 URL，改为接收 `platformId`（可选 `url` 仅作向后兼容），由 background 基于 settings 解析并校验目标平台。

**Acceptance:**
- comment action 触发后：复制成功 → 自动打开目标平台
- inpage 场景不依赖 `window.open` 作为主路径（仅 runtime 不可用时降级）
- 非法/禁用平台不会被打开，并返回可诊断错误
- `npm --prefix webclipper run compile && npm --prefix webclipper run test` 通过

---

## P2-T1 comment级Chat with实现复制后自动跳转

**Files:**
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-comment-actions.ts`
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts`
- Add: `webclipper/src/services/integrations/chatwith/chatwith-open-port.ts`
- Add: `webclipper/tests/unit/chatwith-comment-actions.test.ts`

**Step 1: 实现功能**
- 将 comment action 的 `onTrigger` 从“仅复制”升级为“复制成功后再打开平台”
- 抽象打开能力为可注入 port（`openPlatform(platformId, fallbackUrl)`）：
  - app/popup 默认 port 走现有 `openExternalUrl()`
  - inpage 在 P2-T2 切到 background messaging port
- 失败策略：
  - 复制失败：不打开；提示失败原因
  - 打开失败：提示失败原因（不静默）

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/chatwith-comment-actions.test.ts`

Expected: 编译通过

**Step 3: 原子提交**
Run: `git add webclipper/src/services/integrations/chatwith/chatwith-comment-actions.ts webclipper/src/services/integrations/chatwith/chatwith-detail-header-actions.ts webclipper/src/services/integrations/chatwith/chatwith-open-port.ts webclipper/tests/unit/chatwith-comment-actions.test.ts`

Run: `git commit -m "feat: P2-T1 - comment级chatwith复制后自动跳转"`

---

## P2-T2 inpage场景通过background打开外部tab

**Files:**
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`
- Modify: `webclipper/src/services/protocols/message-contracts.ts`
- Add: `webclipper/src/services/integrations/chatwith/chatwith-background-handlers.ts`
- Modify: `webclipper/src/entrypoints/background.ts`
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- Modify: `webclipper/tests/smoke/background-router-testkit.ts`
- Add: `webclipper/tests/smoke/background-router-chatwith-open-platform.test.ts`

**Step 1: 实现功能**
- 新增 `CHATWITH_MESSAGE_TYPES`：
  - `OPEN_PLATFORM_TAB`（输入：`platformId` + 可选 fallbackUrl；输出：`ok + tabId/windowId/url`）
- 同步更新 `MessageType` union 与 `services/protocols/message-contracts.ts` 的 re-export（避免类型漂移）。
- background handler 执行：
  - 基于 `platformId` 读取 `loadChatWithSettings()`，仅允许启用平台；拒绝任意 URL 注入
  - URL 校验仅允许 http/https
  - `tabs.create({ url, active: true, windowId: sender?.tab?.windowId ?? undefined })`
- inpage 侧统一改为通过该 message 打开外部 tab（包含两部分）：
  - comment 级 chatwith（Phase 2 的新能力）
  - 现有 header chatwith（避免继续走 `window.open` 主路径）
  - 失败策略：提示失败并停止（优先），runtime 不可用时才降级 `window.open`（次选）

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/smoke/background-router-chatwith-open-platform.test.ts`

Expected: background 与 inpage 均可编译，且消息路由可正确打开启用平台。

**Step 3: 原子提交**
Run: `git add webclipper/src/platform/messaging/message-contracts.ts webclipper/src/services/protocols/message-contracts.ts webclipper/src/services/integrations/chatwith/chatwith-background-handlers.ts webclipper/src/entrypoints/background.ts webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts webclipper/tests/smoke/background-router-testkit.ts webclipper/tests/smoke/background-router-chatwith-open-platform.test.ts`

Run: `git commit -m "feat: P2-T2 - inpage通过background打开AI平台tab"`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（`compile + test`）
