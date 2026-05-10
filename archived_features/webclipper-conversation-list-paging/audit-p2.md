# Audit P2 - webclipper-conversation-list-paging

## 审计范围

- 覆盖 `P2-T1` ~ `P2-T8`：
  - Provider 分页状态机与 active 快照
  - 过滤重载与请求竞态保护
  - 近底自动加载与定位收敛
  - summary/facets 口径渲染
  - 仅已加载可见项选择语义与 tooltip
  - UI/Provider 定向测试

## 进入审计前置条件

- `todo.toml` 中 P2 tasks 全部 `completed`。
- P2 任务都有 compile + 定向测试记录。

## 审计检查清单

1. 状态机一致性
- `loadingInitialList/loadingMoreList/hasMore` 转换是否可解释。
- active conversation 快照是否覆盖“目标未加载”场景。

2. 竞态与并发闸门
- 快速切筛选时旧响应是否被丢弃。
- loadMore 并发是否被正确抑制。

3. 自动加载与定位收敛
- sentinel 触发条件是否稳定。
- `pendingListLocateId` 在目标未渲染时是否可增量加载并有界终止。
- 是否存在无限加载/无限重试风险。

4. 口径一致性
- 筛选菜单是否来自 facets 而非 loaded subset。
- 今日/总计是否来自 summary。

5. 选择语义与提示
- `Select All` 是否仅覆盖当前已加载可见项。
- `Delete/Export/Sync` tooltip 是否明确同语义。

6. 旧接口残留
- Provider/列表主链路是否仍引用 `listConversations`。
- 是否存在分页失败后静默回退到全量读取的分支。

## 风险重点

- observer 与定位重试叠加导致循环请求。
- 统计/筛选退化为当前已加载子集。
- tooltip 文案与实际执行对象不一致。
- 全量读取回退分支遗漏清理，导致性能目标失效。

## 审计记录（执行时填写）

- Findings:
  - [x] 无阻断问题
  - [ ] 存在阻断问题（需列出）
- 阻断问题列表：
  1. 无
- 修复提交：
  - 无
- 审计结论：
  - 代码实现与 `P2-T1~T8` 目标对齐，未发现阻断问题。
  - `listConversations / GET_CONVERSATIONS / getConversations(` 在 `webclipper/src` 与 `webclipper/tests` 中无残留命中。
  - 分页闸门（`loadingInitialList/loadingMoreList/listHasMore`）与 `pendingListLocateId` 有界重试（`MAX_LOCATE_LOAD_ROUNDS`）已落地，无无限加载路径。
  - summary/facets 与“仅已加载可见项”选择语义、tooltip 行为一致。

## 逐项证据

### 1) 状态机一致性

- Provider 状态字段与过渡路径存在并且闭环：
  - `loadingInitialList/loadingMoreList/listCursor/listHasMore/listSummary/listFacets`：`conversations-context.tsx` 的 state 与 `refreshList/loadMoreList` 更新链路。
  - active 快照回退：`selectedConversation` 在未命中已加载 `items` 时会回退 `activeConversationSnapshot`（`toConversationFromOpenTarget`）。
- 证据测试：
  - `tests/unit/conversations-provider-pagination.test.ts`：覆盖“目标不在 loaded items 仍可 `openConversationExternalBySourceKey` 并设为 active”。

### 2) 竞态与并发闸门

- 竞态防护：
  - `listRequestSeqRef` 与 `openTargetRequestSeqRef` 都采用 request sequence，旧响应在 `requestSeq` 不一致时被丢弃。
- loadMore 并发抑制：
  - `if (loadingInitialList || loadingMoreList) return;`
  - `if (!cursor || !listHasMore) return;`
- 证据测试：
  - `tests/unit/conversations-provider-pagination.test.ts`：`drops stale bootstrap responses during fast filter switching`。

### 3) 自动加载与定位收敛

- sentinel 自动加载条件：
  - `IntersectionObserver` + `loadingMoreList/listHasMore` 双闸门。
  - test 环境无 `IntersectionObserver` 时直接返回，避免旧测试环境崩溃。
- locate 收敛终止：
  - 目标命中行则 `scrollIntoView + consumeListLocate`。
  - `!listHasMore` 或 `rounds >= MAX_LOCATE_LOAD_ROUNDS` 都会终止并消费。
  - 加载中（initial/more）不重复发请求。
- 证据测试：
  - `tests/unit/conversation-list-pagination.test.ts`：覆盖 near-bottom 自动加载、gating、以及 pending locate 增量收敛。

### 4) 口径一致性

- source/site 筛选来源于 `listFacets`，非当前 loaded subset 推导。
- 今日/总计来自 `listSummary`，与分页窗口解耦。
- 证据代码：
  - `ConversationListPane.tsx` 中 `sourceOptions/siteOptions/todayCount/totalCount` 全部由 `listFacets/listSummary` 计算。

### 5) 选择语义与 tooltip

- Select All 使用 `toggleAll(visibleIds)`，仅作用于“当前已加载可见项”。
- `Delete/Export/Sync` tooltip 统一追加 `tooltipLoadedVisibleSelectionScope`。
- 证据测试：
  - `tests/unit/conversation-list-pagination.test.ts`：断言 select-all 参数为 `[11, 22]`，并验证三类 tooltip 均含该提示 key。

### 6) 旧接口残留

- 扫描结果：`rg -n "listConversations\\(|GET_CONVERSATIONS|getConversations\\(" webclipper/src webclipper/tests` 无命中（exit code 1，表示无匹配）。
- 主链路未看到分页失败后静默回退全量读取分支。

## 回归命令（审计后）

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/unit/conversation-list-pagination.test.ts tests/unit/conversations-provider-pagination.test.ts tests/smoke/conversations-scene-popup-escape.test.ts`
- `npm --prefix webclipper run test -- tests/unit/conversation-list-delete-inline-confirm.test.ts tests/storage/conversations-pagination.test.ts tests/services/conversations-pagination-handlers.test.ts`
