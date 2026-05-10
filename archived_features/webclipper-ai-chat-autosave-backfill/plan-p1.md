# Plan P1 - webclipper-ai-chat-autosave-backfill

**Goal:** 让开启 autosave 的 AI chats 在打开对话时自动补齐缺失历史（窗口内，v1=200），解决跨设备缺口。

**Non-goals:**
- 不做 destructive snapshot（不删除、不替换本地历史）。
- 不新增 UI 开关、不改 i18n。
- 不承诺全量历史；v1 只覆盖最近窗口（与现状一致：200）。

**Approach:**
- 在 content autosave tick 中引入“reconcile/backfill”阶段：先从 background 读取本地 tail(<=200)，再与页面 window(<=200) 做 overlap 对齐，计算缺口并 append-only 写入。
- 如果找不到可靠 overlap：
  - **本地完全为空（conversation 不存在或 tail=0）**：视为“首次落盘”，允许把 page window（<=200）直接 append-only 写入（无删除/无替换），从而解决“打开对话但本地一条都没有”的根因场景。
  - **本地不为空但无法对齐**：走安全策略 A：不写入，仅 console warn；同时保留后续增量保存能力。
- backfill 支持“窗口变化触发的有限重试”：仅当 page window signature 变化时重试，带节流与上限，成功一次即停止。

**Acceptance:**
- 打开对话时无需继续输入即可自动补齐缺失历史（窗口内）。
- 无 overlap 时不写入但增量保存仍可继续。
- 本地完全为空时（首次打开该会话、尚未落盘任何消息），也能在无需继续输入的情况下把 page window（<=200）写入本地一次。

**实施纪律（必须写进每个 task 的执行方式）**
- “及时清理老旧代码”：任何新接口/新抽象一旦落地并被调用，必须在同一个 task 内删除对应旧实现/旧分支/旧 helper，避免积累到最后统一清理。

---

<a id="p1-t1"></a>
## P1-T1 为IndexedDB增加按conversation取messages tail窗口（<=200）能力

**Files:**
- Modify: `webclipper/src/services/conversations/data/storage-idb.ts`
- Modify: `webclipper/tests/storage/conversations-idb.test.ts`

**Step 1: 实现功能**
- 在 `storage-idb.ts` 新增 `getMessagesTailByConversationId(conversationId, limit)`：
  - 使用 `by_conversationId_sequence` 索引反向游标（`openCursor(range, 'prev')`）读取最后 N 条，避免 `getAll` 全量加载。
  - 返回按 sequence 升序排序的数组（与现有 `getMessagesByConversationId` 一致的输出排序）。
  - 注：该方案依赖 message `sequence` 在同一 conversation 内具有“可比较的先后关系”；若某些 collector 仅给出窗口内局部序号，则当前 UI 渲染/排序本身也会受影响，本次按现状假设不额外修正。
- 在 `conversations-idb.test.ts` 增加用例：
  - 写入 1 条 conversation + 300 条 messages；
  - 断言 tail(200) 返回最后 200 条且排序正确。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/storage/conversations-idb.test.ts`
- Expected: 新增测试通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/conversations/data/storage-idb.ts webclipper/tests/storage/conversations-idb.test.ts`
- Run: `git commit -m "feat: P1-T1 - IndexedDB支持按conversation读取messages tail窗口"`

---

<a id="p1-t2"></a>
## P1-T2 增加storage层API：按source+conversationKey读取本地conversation tail窗口

**Files:**
- Modify: `webclipper/src/services/conversations/data/storage-idb.ts`
- Modify: `webclipper/src/services/conversations/data/storage.ts`
- Modify: `webclipper/tests/storage/conversations-idb.test.ts`

**Step 1: 实现功能**
- 在 `storage-idb.ts` 增加 `getConversationTailWindowBySourceAndKey(source, conversationKey, limit)`：
  - 先 `getConversationBySourceConversationKey` 拿到 conversation（或 null）
  - 若存在则调用 `getMessagesTailByConversationId(conversationId, limit)` 返回 tail
  - 返回 `{ conversation, messages }`（conversation 为空时 messages 为空）
- 在 `storage.ts` 透出同名函数（保持 background handler 依赖方向不变）。
- 在 `conversations-idb.test.ts` 增加用例验证：
  - source+key 不存在返回空；
  - 存在时返回 conversationId + tail(200)。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/storage/conversations-idb.test.ts`
- Expected: 新增用例通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/conversations/data/storage-idb.ts webclipper/src/services/conversations/data/storage.ts webclipper/tests/storage/conversations-idb.test.ts`
- Run: `git commit -m "feat: P1-T2 - 提供按source+key读取conversation tail窗口的storage API"`

---

<a id="p1-t3"></a>
## P1-T3 增加background message：getConversationTailWindowBySourceAndKey（含路由与测试）

**Files:**
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`
- Modify: `webclipper/src/services/conversations/background/handlers.ts`
- Modify: `webclipper/tests/services/conversations-pagination-handlers.test.ts`

**Step 1: 实现功能**
- 在 `message-contracts.ts` 的 `CORE_MESSAGE_TYPES` 增加：
  - `GET_CONVERSATION_TAIL_WINDOW_BY_SOURCE_AND_KEY: 'getConversationTailWindowBySourceAndKey'`
- 在 `registerConversationHandlers` 中新增 handler：
  - 参数：`source`, `conversationKey`, `limit?`（v1 默认 200，最大硬上限 200）
  - 先做参数校验，返回 `INVALID_ARGUMENT` 风格 extra（复用现有 invalidArgument 模式）
  - 调用 storage 的 `getConversationTailWindowBySourceAndKey` 并返回（最小可用形状）：
    - `{ conversationId: number | null, messages: ConversationMessage[] }`
    - 其中 `conversationId=null` 表示本地尚无该 conversation（content 侧将其视为“本地完全为空”的 backfill 首次落盘条件）
- 测试（在 `conversations-pagination-handlers.test.ts`）：
  - 记得同步更新该测试文件的 `storageMocks` + `vi.mock('@services/conversations/data/storage', ...)` 导出列表与 reset（否则新增 handler 会触发未 mock 的 import/调用）
  - invalid source/key/limit 的错误分支
  - 正常分支：mock storage 返回 tail，断言 handler 透传并 normalize limit。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/services/conversations-pagination-handlers.test.ts`
- Expected: 新增用例通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/platform/messaging/message-contracts.ts webclipper/src/services/conversations/background/handlers.ts webclipper/tests/services/conversations-pagination-handlers.test.ts`
- Run: `git commit -m "feat: P1-T3 - 增加background message用于读取conversation tail窗口"`

---

<a id="p1-t4"></a>
## P1-T4 实现content侧纯函数reconcile：local tail vs page window 计算backfill delta（含单测）

**Files:**
- Add: `webclipper/src/services/conversations/content/autosave-backfill-reconciler.ts`
- Add: `webclipper/tests/unit/autosave-backfill-reconciler.test.ts`

**Step 1: 实现功能**
- 新增纯函数模块，输入：
  - `localTailMessages`（<=200，来自 background）
  - `pageWindowMessages`（<=200，来自 collector.capture）
  - `stateKeyHash`（用于生成稳定 synthetic messageKey；算法需与 `autosave-incremental-engine.ts` 的 `computeStateKeyHash` 一致）
- 输出：
  - `ok`（是否找到了可靠 overlap）
  - `addedMessages`（需要写入本地的缺口消息；append-only）
  - `diff.added`（新增 messageKey 列表；不产生 updated/removed）
  - `pageSignature`（用于“窗口变化触发重试”的签名；只依赖 page window，避免“本地 tail 变化”导致无意义重试）
- overlap 策略：
  - 以 **与增量引擎一致** 的 identityHash 序列为对齐依据（不依赖 messageKey）：
    - 复用同一套 `normalizeContent/normalizeText` 语义
    - 复用同一套 `IDENTITY_PREFIX_LEN`（当前为 40）
  - 只在“找到 overlap”时生成缺口；找不到则 `ok=false`
  - **本地完全为空**（`localTailMessages.length===0`）时：`ok=true` 且 `addedMessages=pageWindowMessages`（<=200），作为首次落盘路径
  - 缺口消息统一分配 `autosave_${stateKeyHash}_${identityHash}_bf{occ}` 形式的 key（确保跨窗口/跨会话尽可能稳定）

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/unit/autosave-backfill-reconciler.test.ts`
- Expected: 用例覆盖：
  - local 是 page 的前缀（补齐尾部新增）
  - local 是 page 的后缀（补齐头部历史）
  - 完全无 overlap（ok=false，不产生写入）

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/conversations/content/autosave-backfill-reconciler.ts webclipper/tests/unit/autosave-backfill-reconciler.test.ts`
- Run: `git commit -m "feat: P1-T4 - 增加autosave backfill reconcile纯函数与单测"`

---

<a id="p1-t5"></a>
## P1-T5 把backfill reconcile接入content-controller autosave tick（安全策略A + 有限重试 + console日志）

**Files:**
- Modify: `webclipper/src/services/bootstrap/content-controller.ts`

**Step 1: 实现功能**
- 在 autosave `handleTick` 中（在 `collector.capture()` 得到 snapshot 后、`computeIncremental` 之前）插入 backfill 流程：
  1) 计算 pageSignature（复用 reconciler 输出）
  2) 若满足重试条件（signature 变化 + 未超次数/时长 + 节流）：
     - `send('getConversationTailWindowBySourceAndKey', { source, conversationKey, limit: 200 })`
     - 调用 reconciler 得到 `addedMessages`
     - 若 `ok=true && addedMessages.length>0`：
       - 通过现有 `saveSnapshot(..., { mode: 'append', diff })` 写入（append-only）
       - `console.info` 输出一次 `{ source, conversationKey, addedCount }`
       - **同一 tick 内仍然调用** `incrementalUpdater.computeIncremental(snapshot)` 以刷新 session baseline，但**本 tick 不再执行增量写入**（避免 backfill 与 incremental 同 tick 双写导致重复）
     - 若 `ok=false`：
       - 不写入；`console.warn` 输出一次“no overlap, skipped”
- 安全策略 A：任何无 overlap/异常都不写入；不得调用 snapshot mode；不得删除本地消息。
- 有限重试：状态保存在 content controller 运行期内（单 tab 单会话），成功一次后停止重试。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: TypeScript 编译通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/services/bootstrap/content-controller.ts`
- Run: `git commit -m "feat: P1-T5 - autosave tick加入backfill补齐逻辑（安全跳过+有限重试）"`

---

<a id="p1-t6"></a>
## P1-T6 补齐content-controller backfill的smoke测试：成功补齐/无overlap跳过/窗口变化触发重试

**Files:**
- Add: `webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`

**Step 1: 实现功能**
- 参考现有 smoke harness（例如 `content-controller-web-inpage-article-fetch.test.ts`）搭建：
  - collectorsRegistry 返回一个 AI chat collector（如 `chatgpt`）与稳定的 `capture()` snapshot
  - runtime.send mock：
    - 对 `getConversationTailWindowBySourceAndKey` 返回本地 tail
    - 对 `upsertConversation`/`syncConversationMessages` 记录调用参数
- 覆盖四类用例：
  0) 本地为空（conversation 不存在或 tail=0）：应触发一次 backfill 写入（append-only，写入 page window<=200）
  1) overlap 存在且缺口在尾部：应触发一次 backfill 写入（append-only）
  2) 无 overlap：不写入（不调用 `syncConversationMessages`），但仍能继续跑后续 tick
  3) 首次无 overlap，第二次 window signature 变化后 overlap 成立：应在重试后成功写入一次

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- Expected: 四类用例通过。

**Step 3: 原子提交**
- Run: `git add webclipper/tests/smoke/content-controller-ai-chat-autosave-backfill.test.ts`
- Run: `git commit -m "test: P1-T6 - 增加autosave backfill在content-controller的smoke测试"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令：`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`
