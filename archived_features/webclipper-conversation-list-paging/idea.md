# WebClipper：会话侧边栏分页与动态加载重构

## 背景 / 触发

- 当前列表链路是“全量读取 + 前端过滤 + 全量渲染”：`getConversations()` 走 `getAll()`，`ConversationListPane` 基于 `items` 做 source/site 过滤并渲染全部行。
- 数据规模上来后，首屏 CPU 与内存占用明显增加。用户明确目标是优先降低插件性能和内存压力，并接受结构性重构。
- 现有链路还承载了 deep-link(`loc`)、Insight 跳转、窄屏 list/detail bridge、`pendingListLocateId` 定位，这些能力不能回退。

## 核心需求（确认版）

1. 列表从全量读取改为分页读取，滚动接近底部自动加载下一页。
2. deep-link / Insight 打开目标会话时，即使目标未加载，也要按 `source + conversationKey` 精确打开，并能定位到列表行（可通过增量加载收敛）。
3. `Select All` 语义固定为“仅选择当前已加载的可见项”。
4. tooltip 明确该语义，并覆盖两处：
   - `Select All` 控件
   - `Export / Sync / Delete` 批量操作按钮
5. 不做“加载全部”按钮。
6. 保持现有筛选持久化键、`loc` 编码契约、窄屏路由桥接契约不变。

## 默认值与切换策略

- 默认分页开启，app/popup 使用同一分页链路。
- 保持筛选存储键不变：
  - `webclipper_conversations_source_filter_key`
  - `webclipper_conversations_site_filter_key`
- `loc` 继续使用 `base64url(source||conversationKey)`，不改历史链接语义。
- 采用破坏性重构：`GET_CONVERSATIONS` / `listConversations()` / `getConversations()` 在主链路迁移时直接下线，不保留运行时回退。
- 若有调用点未迁移，编译或测试必须直接失败（通过类型与测试暴露），不做静默兼容。

## 新增约束（本轮审查补充）

- 分页后 source/site 过滤项、今日/总计统计不能退化为“仅已加载子集”；必须保持筛选能力与统计口径稳定。
- 选中会话若不在当前已加载列表中，`selectedConversation` 仍需可用（标题、URL、详情头动作、评论侧栏上下文不能丢）。
- 分页查询必须有稳定排序和 tie-break，避免重复/漏项（尤其 `lastCapturedAt` 相同场景）。

## 非目标

- 不实现“筛选条件下全部会话”的全局选择状态机。
- 不新增“加载全部”入口。
- 不改 Notion/Obsidian 同步语义，不改备份格式。

## 验收标准（可检查）

1. Provider 初始化不再依赖全量 `getAll()` 返回；列表走分页 bootstrap + loadMore。
2. 近底自动加载可触发且可停止（`hasMore=false` 不再请求）。
3. deep-link/Insight 在目标未加载时可精确打开，且最终可定位到目标行（有界重试，不死循环）。
4. source/site 筛选项与今日/总计统计在分页后口径保持正确，不退化为“当前页统计”。
5. `Select All` 与批量操作 tooltip 明确“仅当前已加载可见项”。
6. 验证链至少覆盖：
   - `npm --prefix webclipper run compile`
   - `npm --prefix webclipper run test`
   - `npm --prefix webclipper run build`
7. 若存在既有阻塞，必须标注是否由本 feature 引入。
