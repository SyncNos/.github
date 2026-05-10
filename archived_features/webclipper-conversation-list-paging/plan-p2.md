# Plan P2 - webclipper-conversation-list-paging

**Goal:** 完成 Provider 与列表 UI 分页化重构，落地自动加载与“仅已加载可见项”选择语义。

**Non-goals:** 本 phase 不处理 AppShell/Insight 跳转入口改造（留到 P3）。

**Approach:**
- 先把 Provider 从“全量 items 单态”改为“分页列表态 + activeConversation 快照态”。
- 再接入 `IntersectionObserver` 自动加载与定位收敛。
- 最后收敛筛选/统计口径与 tooltip 语义，避免 UI 行为误导。
- Provider/列表读取统一只走分页接口，不保留旧全量读取回退路径。

**Acceptance:**
- 列表加载不依赖全量 `items`。
- 近底自动加载稳定可控，不死循环。
- 筛选菜单与统计口径在分页后保持正确。
- `Select All` 与批量动作 tooltip 清晰表达“仅已加载可见项”。
- 会话列表路径不存在 `listConversations` 调用残留。

---

<a id="p2-t1"></a>
## P2-T1 Provider 状态机重构：分页状态与 activeConversation 快照解耦

**Files:**
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`

**Step 1: 实现**
1. 引入分页状态：`listCursor/listHasMore/loadingInitialList/loadingMoreList/listSummary/listFacets`。
2. 引入 active conversation 快照状态（目标未加载时仍可提供标题/URL/sourceType）。
3. `selectedConversation` 计算改为“loaded item 优先 + active snapshot 兜底”。
4. 移除 Provider 对旧全量 `refreshList -> listConversations()` 读取链路依赖。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: Context 对外类型完整，调用点可编译。

**Step 3: 原子提交**
- `refactor: task7 - Provider重构为分页状态机与active快照`

---

<a id="p2-t2"></a>
## P2-T2 过滤重载与请求竞态保护（分页 + 精确打开）

**Files:**
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`

**Step 1: 实现**
1. source/site 变化触发 bootstrap 重载，重置 cursor 与错误态。
2. 引入 request 序列号（或 token）丢弃过期响应。
3. 暴露 `openConversationExternalByLoc` / `openConversationExternalBySourceKey` / `openConversationExternalById`，支持未加载目标精确打开。
4. 过滤切换时清理选择态，避免跨筛选误操作。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/conversations-sync-feedback.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 快速切筛选不串页，精确打开类型通过。

**Step 3: 原子提交**
- `feat: task8 - Provider支持分页过滤重载与精确打开`

---

<a id="p2-t3"></a>
## P2-T3 ConversationListPane 接入近底自动加载 sentinel

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`

**Step 1: 实现**
1. 在列表底部增加 sentinel，`IntersectionObserver` 以滚动容器为 root。
2. 触发条件受 `loadingMoreList`/`listHasMore` 闸门保护。
3. 保持 `initialScrollTop/scrollRestoreKey` 现有恢复链路。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 列表可在滚动近底触发增量加载。

**Step 3: 原子提交**
- `feat: task9 - 列表接入近底自动分页加载`

---

<a id="p2-t4"></a>
## P2-T4 实现定位收敛：未渲染目标触发增量加载并有界重试

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`

**Step 1: 实现**
1. `pendingListLocateId` 命中不到 DOM 行时，若 `hasMore=true` 自动触发 `loadMoreList()`。
2. 每轮加载后重试定位；达到上限或 `hasMore=false` 必须终止并消费 pending。
3. 避免 `requestAnimationFrame` + observer 叠加造成无限循环。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 定位流程可收敛，不死循环。

**Step 3: 原子提交**
- `fix: task10 - 增量加载场景下收敛列表定位`

---

<a id="p2-t5"></a>
## P2-T5 筛选菜单与统计改为 summary/facets 口径

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`

**Step 1: 实现**
1. source/site 选项来源改为 provider `listFacets`，不依赖已加载子集推断。
2. 今日/总计显示改为 `listSummary`，不再直接用 `filteredItems.length`。
3. 处理持久化 siteKey 在 facets 中缺失时的安全回退。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 分页后筛选项和统计口径稳定。

**Step 3: 原子提交**
- `refactor: task11 - 列表筛选与统计切换到summary-facets口径`

---

<a id="p2-t6"></a>
## P2-T6 落地“仅已加载可见项”选择语义与双处 tooltip

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`
- Modify: `webclipper/src/ui/i18n/locales/zh.ts`
- Modify: `webclipper/src/ui/i18n/locales/en.ts`

**Step 1: 实现**
1. `Select All` tooltip 明确“仅当前已加载可见项”。
2. `Delete/Export/Sync` tooltip 在有选择时补充同语义提示。
3. 批量动作执行对象保持 `selectedIds`，不扩展到未加载项。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 文案键和 UI 引用一致。

**Step 3: 原子提交**
- `feat: task12 - 明确已加载可见项选择语义与tooltip`

---

<a id="p2-t7"></a>
## P2-T7 完善分页加载态/错误态/无更多数据边界

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`

**Step 1: 实现**
1. 增加底部状态：加载中、失败可重试、已全部加载。
2. 失败时保留已加载列表，不清空。
3. 切换筛选时重置错误态/hasMore/loadingMore，避免脏状态串联。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/conversation-list-delete-inline-confirm.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 现有列表交互不回归。

**Step 3: 原子提交**
- `refactor: task13 - 完善分页加载边界状态`

---

<a id="p2-t8"></a>
## P2-T8 补齐分页 UI/Provider 定向测试

**Files:**
- Add: `webclipper/tests/unit/conversation-list-pagination.test.tsx`
- Add: `webclipper/tests/unit/conversations-provider-pagination.test.ts`
- Modify: `webclipper/tests/smoke/conversations-scene-popup-escape.test.ts`

**Step 1: 实现**
1. 覆盖自动加载触发/停止条件。
2. 覆盖筛选竞态（过期响应丢弃）。
3. 覆盖“Select All 仅已加载可见项”与 tooltip。
4. 覆盖定位收敛场景（目标不在首批加载）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/conversation-list-pagination.test.tsx tests/unit/conversations-provider-pagination.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 新增测试通过，编译通过。

**Step 3: 原子提交**
- `test: task14 - 补齐分页UI与provider竞态测试`

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环。
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
