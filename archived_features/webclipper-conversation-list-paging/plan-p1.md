# Plan P1 - webclipper-conversation-list-paging

**Goal:** 建立分页数据基线：高效分页查询、稳定过滤口径、精确打开契约和可调用协议。

**Non-goals:** 本 phase 不改 `ConversationsProvider`/`ConversationListPane` UI 渲染逻辑。

**Approach:**
- 先定义统一分页域模型（query/cursor/page/summary/facets/open-target），避免 Provider 与存储层语义分叉。
- 通过 schema + 派生键 + 复合索引支撑分页与过滤，不依赖每页全表扫描。
- 同步补齐 `by-loc` / `by-id` 精确查找，支撑目标未加载时的打开链路。

**Acceptance:**
- 存储层提供 `bootstrap/page/by-loc/by-id` API。
- 分页排序稳定（`lastCapturedAt desc` + `id desc`）。
- summary/facets 可被 UI 直接消费（筛选项与统计不依赖全量 `items`）。
- 旧全量接口 `GET_CONVERSATIONS/listConversations/getConversations` 在迁移后不可再被会话列表链路调用。

---

<a id="p1-t1"></a>
## P1-T1 定义分页查询域模型（query/cursor/page/summary/facets/open-target）

**Files:**
- Add: `webclipper/src/services/conversations/domain/list-query.ts`
- Add: `webclipper/src/services/conversations/domain/list-pagination.ts`
- Modify: `webclipper/src/services/conversations/domain/models.ts`（仅补充必要类型导出）

**Step 1: 实现**
1. 定义统一 query：`sourceKey`、`siteKey`、`limit`。
2. 定义 cursor（至少含 `lastCapturedAt` + `id`）。
3. 定义 page result：`items/cursor/hasMore/summary/facets`。
4. 定义 open-target result（按 `source+conversationKey` 与 `id` 查找所需最小字段）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 新增域类型可被 data/background/client 引用。

**Step 3: 原子提交**
- `feat: task1 - 定义会话列表分页查询域模型`

---

<a id="p1-t2"></a>
## P1-T2 Schema 迁移：补充分页索引与过滤派生键

**Files:**
- Modify: `webclipper/src/platform/idb/schema.ts`
- Modify: `webclipper/tests/storage/schema-migration.test.ts`

**Step 1: 实现**
1. 为 `conversations` 增加分页索引（建议包含 `lastCapturedAt + id` 复合键）。
2. 增加过滤索引（建议 source/site 相关复合键，避免分页过滤时反复全表扫描）。
3. 在 upgrade 中为历史记录回填派生键（如 `listSourceKey/listSiteKey`）。
4. 明确迁移异常策略：仅对“非关键修复”容错，不吞掉会破坏数据一致性的错误。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/storage/schema-migration.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 升级后索引存在、派生键可回填、编译通过。

**Step 3: 原子提交**
- `feat: task2 - 增加分页索引与过滤派生键迁移`

---

<a id="p1-t3"></a>
## P1-T3 实现 IDB 分页查询、精确查找与统计摘要接口

**Files:**
- Modify: `webclipper/src/services/conversations/data/storage-idb.ts`
- Modify: `webclipper/src/services/conversations/data/storage.ts`

**Step 1: 实现**
1. 新增 `getConversationListBootstrap(query, limit)`。
2. 新增 `getConversationListPage(query, cursor, limit)`。
3. 新增 `findConversationBySourceAndKey(source, conversationKey)`。
4. 新增 `getConversationById(conversationId)`（供 active 快照补全）。
5. 返回 summary/facets：用于“总计/今日统计”和筛选菜单。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/storage/conversations-idb.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 分页正确、精确查找可用、编译通过。

**Step 3: 原子提交**
- `feat: task3 - 实现分页查询与精确查找存储接口`

---

<a id="p1-t4"></a>
## P1-T4 写路径收敛：upsert/merge 同步维护分页派生键

**Files:**
- Modify: `webclipper/src/services/conversations/data/storage-idb.ts`

**Step 1: 实现**
1. `upsertConversation` 写入时同步维护派生过滤键。
2. `mergeConversationsByIds` 后保持派生键正确。
3. 保证旧数据被更新后会逐步修正派生键，避免新旧口径混用。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/storage/conversations-idb.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 写入/合并后分页过滤行为稳定。

**Step 3: 原子提交**
- `refactor: task4 - 收敛写路径并维护分页派生键`

---

<a id="p1-t5"></a>
## P1-T5 扩展分页消息协议与 repo API，并删除旧全量列表接口

**Files:**
- Modify: `webclipper/src/platform/messaging/message-contracts.ts`
- Modify: `webclipper/src/services/conversations/background/handlers.ts`
- Modify: `webclipper/src/services/conversations/client/repo.ts`

**Step 1: 实现**
1. 新增分页消息类型并保持现有命名风格（camelCase 值）。
2. background handlers 增加 query 参数校验和统一错误结构。
3. repo 新增 bootstrap/page/by-loc/by-id 封装。
4. 删除 `GET_CONVERSATIONS` 消息类型、background handler 的对应路由，以及 repo `listConversations()` 导出。
5. 若仍有调用点依赖旧接口，直接改为分页接口或精确打开接口，不引入 fallback。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/background-router-conversations-events.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 背景路由与 repo 调用通过，且不存在旧全量接口残留调用。

**Step 3: 原子提交**
- `feat: task5 - 扩展分页协议并删除旧全量列表接口`

---

<a id="p1-t6"></a>
## P1-T6 补齐 P1 存储与路由测试（迁移/分页/精确查找）

**Files:**
- Modify: `webclipper/tests/storage/conversations-idb.test.ts`
- Add: `webclipper/tests/storage/conversations-pagination.test.ts`
- Add: `webclipper/tests/services/conversations-pagination-handlers.test.ts`

**Step 1: 实现**
1. 增加同时间戳 tie-break 场景（稳定顺序）。
2. 增加翻页去重/不漏项测试。
3. 增加 summary/facets 口径测试（source/site、today/total）。
4. 增加 by-loc/by-id 精确查找测试与 handler 参数校验测试。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/storage/conversations-pagination.test.ts tests/services/conversations-pagination-handlers.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 新增测试通过，编译通过。

**Step 3: 原子提交**
- `test: task6 - 补齐分页存储与路由测试`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环。
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
