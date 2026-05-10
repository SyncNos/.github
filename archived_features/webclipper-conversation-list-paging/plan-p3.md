# Plan P3 - webclipper-conversation-list-paging

**Goal:** 收口跨页面联动（AppShell/Insight/窄屏桥接）、文档同步和最终验证。

**Non-goals:** 不扩展“筛选下全部选择”，不新增“加载全部”入口。

**Approach:**
- 先把 `loc` 与 Insight 打开路径迁移到 provider 精确打开能力。
- 再处理“目标未加载”下详情头、评论侧栏和窄屏桥接一致性。
- 最后做文档同步与完整验证归因。

**Acceptance:**
- `loc` 打开与 Insight 跳转不依赖全量 `items`。
- 目标未加载时仍有正确详情上下文（标题/URL/动作/评论上下文）。
- deep-link/locate/窄屏桥接稳定，文档与实现一致。
- 旧全量列表接口在代码中无残留调用（含 repo/message handler）。

---

<a id="p3-t1"></a>
## P3-T1 AppShell loc 消费切换到 provider 精确打开链路

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`

**Step 1: 实现**
1. 将 `AppShell` 的 `loc` 处理从 `items.find()` 改为 provider `openConversationExternalByLoc`。
2. 保持 `encodeConversationLoc` 写回行为与兼容契约。
3. 处理 `processedLocRef/lastInternalLocRef` 细节，避免重复消费或误吞外部 `loc`。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/app-shell-sidebar-collapse.test.ts tests/smoke/app-shell-narrow-header-actions.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: AppShell 关键联动不回归。

**Step 3: 原子提交**
- `feat: task15 - AppShell切换为provider级loc精确打开`

---

<a id="p3-t2"></a>
## P3-T2 InsightPanel 去除全量 items 路由依赖

**Files:**
- Modify: `webclipper/src/viewmodels/settings/insight-stats.ts`
- Modify: `webclipper/src/ui/settings/sections/InsightPanel.tsx`
- Modify: `webclipper/tests/storage/insight-stats.test.ts`

**Step 1: 实现**
1. `InsightTopConversation` 增加精确打开所需字段（例如 `source + conversationKey` 或 `loc`）。
2. Insight 链接与点击行为改为使用该字段，不构建 `items -> route` 映射。
3. 保持 Link 可访问性语义与窄屏兼容行为。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/storage/insight-stats.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: Insight 统计与跳转稳定。

**Step 3: 原子提交**
- `refactor: task16 - Insight跳转去除全量items依赖`

---

<a id="p3-t3"></a>
## P3-T3 详情头与评论侧栏上下文兼容“目标未加载”场景

**Files:**
- Modify: `webclipper/src/viewmodels/conversations/conversations-context.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`
- Modify: `webclipper/src/ui/app/AppShell.tsx`

**Step 1: 实现**
1. 详情头标题/URL/动作在目标未加载时仍可从 active 快照渲染。
2. 评论侧栏上下文不再硬依赖“当前会话必须在 loaded list 中”。
3. 保持 URL 编辑、详情工具按钮在该场景下行为可解释（必要时显式禁用并给提示）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 目标未加载时仍能稳定显示详情上下文。

**Step 3: 原子提交**
- `fix: task17 - 修复未加载目标会话的详情上下文回归`

---

<a id="p3-t4"></a>
## P3-T4 完成 deep-link/locate/窄屏桥接回归修复

**Files:**
- Modify: `webclipper/src/ui/conversations/pending-open.ts`
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`
- Modify: `webclipper/tests/smoke/conversations-scene-popup-escape.test.ts`
- Modify: `webclipper/tests/smoke/popup-shell-header-actions.test.ts`

**Step 1: 实现**
1. 回归 `pending-open -> setActiveId -> openDetail` 在分页下的一致性。
2. 验证 locate 加载收敛与窄屏往返 scroll restore 不冲突。
3. 明确终止条件，避免无限加载/无限重试。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/conversations-scene-popup-escape.test.ts tests/smoke/popup-shell-header-actions.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: deep-link 与窄屏桥接关键 smoke 通过。

**Step 3: 原子提交**
- `fix: task18 - 修复分页后的deep-link定位与窄屏桥接`

---

<a id="p3-t5"></a>
## P3-T5 同步文档真源（AGENTS/deepwiki）

**Files:**
- Modify: `webclipper/AGENTS.md`
- Modify: `.github/deepwiki/modules/webclipper.md`
- Modify: `.github/deepwiki/storage.md`

**Step 1: 实现**
1. 更新“会话列表分页 + 自动加载”行为说明。
2. 更新“Select All 仅当前已加载可见项”语义说明。
3. 更新 `loc` 精确打开和定位收敛排查入口。

**Step 2: 验证**
- Run: `rg -n "分页|自动加载|已加载|Select All|loc" webclipper/AGENTS.md .github/deepwiki/modules/webclipper.md .github/deepwiki/storage.md`
- Expected: 文档关键语义可检索且与实现一致。

**Step 3: 原子提交**
- `docs: task19 - 同步会话分页与选择语义文档`

---

<a id="p3-t6"></a>
## P3-T6 执行最终验证链并记录已知阻塞归因

**Files:**
- Modify: `.github/features/webclipper-conversation-list-paging/audit-p3.md`
- Modify: `.github/features/webclipper-conversation-list-paging/todo.toml`（执行流回写）

**Step 1: 实现**
1. 运行完整验证链：`compile -> test -> build`。
2. 若 `test` 有历史阻塞，记录失败用例并标注“是否由本 feature 引入”。
3. 在审计记录 residual risks 和后续建议。
4. 增加残留检查：`rg -n \"listConversations\\(|GET_CONVERSATIONS|getConversations\\(\" webclipper/src webclipper/tests` 应无会话列表主链路调用。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Run: `npm --prefix webclipper run build`
- Expected: compile/build 通过；test 失败需可追踪归因。

**Step 3: 原子提交**
- `chore: task20 - 完成会话分页重构最终验证与归因`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环。
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
