# 计划 P1 - webclipper-dollar-mention-in-ai-chats

**目标：** 建立 `$` 提及（mention）的协议与数据能力底座：候选查询、插入文本构建、消息协议与 background 路由一次打通。

**非目标：** 本阶段不落地输入框 UI，不处理 ChatGPT/NotionAI 的光标插入细节。

**实现思路：**
- 先定义 mention 领域契约（query、candidate、排序、错误返回），避免 content/background 语义分叉。
- 再扩展消息协议并在 background 注册独立 handler，保持与 conversations handler 解耦。
- 候选查询走本地会话库过滤（标题/来源/域名），但不复用 `getConversationListBootstrap/getConversationListPage` 分页接口；改为独立存储查询，保证“未加载项”也可命中，并复用 IDB 的 `by_lastCapturedAt_id` 索引做降序游标读取（避免全表 `getAll()`）。
- 插入文本直接复用现有 `formatConversationMarkdownForExternalOutput` 真源，不新增第二套格式化逻辑。

**验收标准：**
- 提供稳定 API：`searchMentionCandidates`、`buildMentionInsertText`。
- 空查询（仅 `$`）按最近保存时间倒序返回。
- 查询匹配口径固定为标题/来源/域名。
- 插入文本与现有“Copy full markdown”同源，且不做截断。
- 不引入多余/旧路径：候选检索不得调用 `getConversationListBootstrap/getConversationListPage`，且不得通过 `getAll()` 全表读取实现“最近项”。

---

<a id="p1-t1"></a>
## P1-T1 定义 `$` mention 协议契约与候选匹配模型

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/mention-contract.ts`
- Add: `webclipper/src/services/integrations/item-mention/mention-search.ts`
- Add: `webclipper/tests/unit/item-mention-search.test.ts`

**Step 1: 实现**
1. 定义 `MentionQuery`、`MentionCandidate`、`MentionSearchResult`、`MentionInsertPayload` 类型。
2. 定义过滤口径：标题/来源/URL 域名；统一 normalize（trim/lowercase）。
3. 定义排序：空查询按 `lastCapturedAt desc`，有查询时按“匹配优先级 + recency”。
4. 明确默认 limit 与上限（例如默认 20，上限 50），防止 content 高频输入放大查询成本。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/item-mention-search.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 匹配字段、排序、limit 约束都通过单测。

**Step 3: 原子提交**
- `feat: task1 - 定义mention协议契约与候选匹配模型`

---

<a id="p1-t2"></a>
## P1-T2 扩展消息协议并注册 mention 背景处理入口

**Files:**
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`
- Modify: `webclipper/src/entrypoints/background.ts`
- Add: `webclipper/src/services/integrations/item-mention/background-handlers.ts`

**Step 1: 实现**
1. 在 `message-contracts.ts` 新增 `ITEM_MENTION_MESSAGE_TYPES`，值遵循现有约定（lowerCamel），例如：`searchMentionCandidates`、`buildMentionInsertText`。
2. 在 background 入口注册 mention handlers，保持独立模块，不把逻辑塞进 conversations handler。
3. 在 `messageContracts` 与 `MessageType` 联合类型中纳入 item-mention 新消息类型，确保 content/client 可安全引用。
4. 统一返回结构与错误风格，保持与 background router 现有约定一致。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `rg -n "SEARCH_MENTION_CANDIDATES|BUILD_MENTION_INSERT_TEXT" webclipper/src || true`（不应再出现旧命名）
- Expected: 新消息类型可被 content/client 引用且背景路由可注册。

**Step 3: 原子提交**
- `feat: task2 - 扩展mention消息协议并注册背景入口`

---

<a id="p1-t3"></a>
## P1-T3 实现分页架构下候选搜索接口（独立存储查询）

**Files:**
- Modify: `webclipper/src/services/integrations/item-mention/background-handlers.ts`
- Modify: `webclipper/src/services/conversations/data/storage.ts`
- Modify: `webclipper/src/services/conversations/data/storage-idb.ts`
- Add: `webclipper/tests/storage/item-mention-search-storage.test.ts`

**Step 1: 实现**
1. 在 `storage-idb.ts` 增加独立候选查询（用 `conversationsStore.index('by_lastCapturedAt_id').openCursor(null, 'prev')` 读取最近记录），不要走列表分页 API。
2. 候选结果返回最小字段：`conversationId/title/source/url/lastCapturedAt/sourceType`。
3. 对 legacy 记录兼容：缺失 `listSourceKey/listSiteKey` 时回退 `source/url` 推导过滤字段，避免漏检。
4. handler 复用 `mention-search` 模型；空查询（用户只输入 `$`）直接走 recency，不要求至少 1 个字符才触发。
5. 确保 `$100` 这类场景不被提前过滤（遵循“出现 `$` 就弹”契约）。
6. 增加扫描成本上限（例如最多扫描 2000 条或 300ms），避免输入时高频触发导致后台长事务阻塞。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/item-mention-search.test.ts tests/storage/item-mention-search-storage.test.ts`
- Run: `npm --prefix webclipper run compile`
- Run: `rg -n "getConversationListBootstrap|getConversationListPage|getAll\\(" webclipper/src/services/integrations/item-mention`
- Expected: 候选返回稳定、空查询排序正确、数字查询不被特殊屏蔽。

**Step 3: 原子提交**
- `feat: task3 - 实现mention候选搜索接口与排序规则`

---

<a id="p1-t4"></a>
## P1-T4 实现插入文本构建接口并复用现有 Markdown copy 真源

**Files:**
- Modify: `webclipper/src/services/integrations/item-mention/background-handlers.ts`
- Modify: `webclipper/src/services/conversations/data/storage.ts`

**Step 1: 实现**
1. `buildMentionInsertText` 按 `conversationId` 读取 conversation + detail（新增/导出 `getConversationById` wrapper），并调用 `formatConversationMarkdownForExternalOutput` 生成插入文本。
2. 严格禁用截断路径：不调用 `truncateForChatWith`。
3. 对 conversation 不存在、detail 空消息等场景返回可识别错误。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/detail-header-actions.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: Markdown 构建链可复用且不影响既有 Chat-with 行为。

**Step 3: 原子提交**
- `refactor: task4 - 复用现有markdown真源构建mention插入文本`

---

<a id="p1-t5"></a>
## P1-T5 补齐 mention 背景接口与匹配逻辑测试

**Files:**
- Add: `webclipper/tests/smoke/background-router-item-mention.test.ts`
- Modify: `webclipper/tests/smoke/background-router-conversations-events.test.ts`（如需共用 mock）
- Modify: `webclipper/tests/unit/item-mention-search.test.ts`
- Modify: `webclipper/tests/storage/item-mention-search-storage.test.ts`

**Step 1: 实现**
1. 增加背景路由测试：未知 id、空查询、有查询、limit 边界、构建插入文本失败路径。
2. 增加一致性测试：mention 构建文本与现有 copy markdown 输出一致。
3. 覆盖 no-truncation 契约，防止后续回归到 `maxChars` 路径。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/background-router-item-mention.test.ts tests/unit/item-mention-search.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: P1 新增测试通过，编译通过。

**Step 3: 原子提交**
- `test: task5 - 补齐mention背景接口与匹配测试`

---

## 阶段审计

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环。
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
