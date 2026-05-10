# Plan P4 - webclipper-comments-chat-with-ai

**Goal:** 同一篇文章的多条 root comment 点击 `Chat with AI` 时：
1) 复用同一个 AI tab（同一平台）并聚焦；2) 把“文章 tab + AI tab”放入同一个 tab group。

**Non-goals:**
- 本 phase 不实现自动填入输入框并发送
- 本 phase 不提供设置开关（后续如有投诉再加）

**Approach:**
- 由 background 统一执行 tabs/tabGroups 相关操作（inpage 通过 message 调用）。
- 背景服务按 `platformId` 解析平台 URL（不信任 content 侧直接传入任意 URL）。
- tab group membership 规则：
  - group 必须包含文章 tab（触发来源的 active tab）
  - group 包含本文章对应平台的 AI tab（可复用聚焦）
- 复用定位：
  - 以 `articleKey + platformId` 为 key 存储 `aiTabId`（storage.local），并在触发时校验 tab 是否仍存在且 URL host 匹配。
  - groupId 不作为稳定标识（可能随 session restore 变化），以“当前文章 tab 的 groupId”作为即时真源。
- 关键边界（必须在 runner 里处理）：
  - tab group 只能作用于同一 `windowId`：创建 AI tab 时要确保落在文章 tab 的同一窗口；已存在 AI tab 若在其他窗口，要么移动到同窗、要么重建（以实现为准，但必须可预测）。
  - app/popup 场景未必能稳定拿到“对应文章 tab”：拿不到时仅做“复用/聚焦 AI tab”（不强行分组），并给出可诊断的 notice/log。
  - 若文章 tab 已在用户自建 group 中：把 AI tab 加入该 group 会改变用户分组语义（可接受但需在 plan 中明确；后续如投诉再加 setting 开关）。
  - Firefox/不支持 tabGroups 的运行时：自动降级为“复用/聚焦 AI tab 或新开 tab”，不得抛未捕获异常。

**Acceptance:**
- 第一次点击：复制 → 打开 AI 平台 tab → 将文章 tab 与 AI tab group 在一起
- 第二次点击（同文章同平台）：复制 → 聚焦到同一 AI tab（不新开）
- 在无 tabGroups API 的浏览器中仍可完成“复制 + 打开/聚焦”
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build && npm --prefix webclipper run build:firefox && npm --prefix webclipper run check` 通过

---

## P4-T1 新增tab group能力封装与权限

**Files:**
- Add: `webclipper/src/platform/webext/tab-groups.ts`
- Modify: `webclipper/src/platform/webext/tabs.ts`
- Modify: `webclipper/wxt.config.ts`
- Add: `webclipper/tests/unit/tab-groups.test.ts`

**Step 1: 实现功能**
- 增加对 `tabs.group()/tabs.ungroup()` 的封装（browser/chrome 双实现），并确保错误信息可诊断。
- 在 `tabs.ts` 按需补齐 `tabsMove` 等 Phase 4 必需能力，避免在业务层直接触碰原生 API。
- manifest permissions 增加 `tabGroups`，并明确运行时特性检测（无 API 时自动降级，不阻塞主流程）。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/tab-groups.test.ts`

Run: `npm --prefix webclipper run build:firefox`

Run: `npm --prefix webclipper run check`

Expected: 编译通过；manifest 与 Firefox 构建均不报错。

**Step 3: 原子提交**
Run: `git add webclipper/src/platform/webext/tab-groups.ts webclipper/src/platform/webext/tabs.ts webclipper/wxt.config.ts webclipper/tests/unit/tab-groups.test.ts`

Run: `git commit -m "feat: P4-T1 - 引入tabGroups能力与权限"`

---

## P4-T2 实现文章tab+AI tab的分组与复用聚焦

**Files:**
- Add: `webclipper/src/services/integrations/chatwith/tabgroup-store.ts`
- Add: `webclipper/src/services/integrations/chatwith/tabgroup-runner.ts`
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-background-handlers.ts`
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`
- Modify: `webclipper/src/services/protocols/message-contracts.ts`
- Modify: `webclipper/src/entrypoints/background.ts`
- Add: `webclipper/tests/unit/chatwith-tabgroup-runner.test.ts`
- Add: `webclipper/tests/smoke/background-router-chatwith-tabgroup.test.ts`

**Step 1: 实现功能**
- `tabgroup-store.ts`：
  - storage key：`chat_with_tab_reuse_v1`
  - key：`${platformId}::${articleKey}` → value：`{ aiTabId, updatedAt }`
  - 清理：监听 `tabs.onRemoved` 或在每次使用时惰性清理
- `tabgroup-runner.ts`（background-only）：
  - 输入：`{ platformId, articleKey, articleTabId }`
  - 行为：
    1. 校验/聚焦已有 aiTabId（存在且 URL host 匹配）
    2. 否则创建新 AI tab（active，且尽量在文章 tab 的 `windowId` 下创建）
    3. 确保文章 tab 与 AI tab 同组：若文章 tab 未分组则创建组；若已分组则把 AI tab 加入该组
    4. 聚焦窗口（如可得 windowId）
  - 额外边界：
    - 若已有 aiTabId 在其他窗口：优先移动到文章 tab 的窗口再分组；若移动失败则重建新 tab 并更新 store（以实现为准，但要写清楚并单测/手测覆盖）
- 在 `chatwith-background-handlers.ts` 暴露 message：`CHATWITH_MESSAGE_TYPES.OPEN_OR_FOCUS_GROUPED_CHAT_TAB`
  - 输入优先使用 `platformId + articleKey`，URL 在 background 解析
  - 同步更新 `MessageType` union / 导出类型（与 P2-T2 一致）
  - 运行时无 tabGroups 能力时自动降级为 P2 行为

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/unit/chatwith-tabgroup-runner.test.ts tests/smoke/background-router-chatwith-tabgroup.test.ts`

Expected: background 编译通过；runner 在“复用/重建/跨窗/无tabGroups”场景可预测。

**Step 3: 原子提交**
Run: `git add webclipper/src/services/integrations/chatwith/tabgroup-store.ts webclipper/src/services/integrations/chatwith/tabgroup-runner.ts webclipper/src/services/integrations/chatwith/chatwith-background-handlers.ts webclipper/src/platform/messaging/message-contracts.ts webclipper/src/services/protocols/message-contracts.ts webclipper/src/entrypoints/background.ts webclipper/tests/unit/chatwith-tabgroup-runner.test.ts webclipper/tests/smoke/background-router-chatwith-tabgroup.test.ts`

Run: `git commit -m "feat: P4-T2 - 同文章复用AI tab并与文章tab分组"`

---

## P4-T3 inpage接入Phase4: background执行分组与聚焦

**Files:**
- Modify: `webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-comment-actions.ts`

**Step 1: 实现功能**
- inpage comment action 触发时，把 `platformId + articleKey` 发送给 background：
  - 优先使用 background message 的 `sender.tab.id`/`sender.tab.windowId` 作为 article tab 上下文（无需 content script 自己猜 tabId）
  - `articleKey` 必须使用 canonical URL（避免同文多 key）
  - 若 sender.tab 不可用（app/popup）：走“仅复用/聚焦 AI tab，不强制分组”
- 在 Phase 4 的 action 中改为调用 `OPEN_OR_FOCUS_GROUPED_CHAT_TAB`：
  - background 返回 ok 后即可
  - 失败降级：回退到 Phase 2 的“仅打开平台 tab”

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Run: `npm --prefix webclipper run test -- tests/smoke/background-router-chatwith-tabgroup.test.ts`

Run: `npm --prefix webclipper run build`

Run: `npm --prefix webclipper run build:firefox`

Run: `npm --prefix webclipper run check`

Expected: 编译与构建通过

**Step 3: 原子提交**
Run: `git add webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts webclipper/src/ui/app/AppShell.tsx webclipper/src/services/integrations/chatwith/chatwith-comment-actions.ts`

Run: `git commit -m "feat: P4-T3 - inpage接入tab分组与复用"`

---

## Phase Audit

- Audit file: `audit-p4.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（`compile + test + build + build:firefox + check`）
