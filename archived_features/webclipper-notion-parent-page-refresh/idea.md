# 背景 / 触发

- WebClipper 的 Notion 设置里存在“Parent Page 刷新长期空 / 过很久才出现列表”的体验问题：用户点击刷新后没有可见 loading 状态，且列表可能持续显示空占位（历史上出现过“半天后才突然显示出 pages”的现象）。
- 现状实现存在分叉与边界不清：Settings ViewModel 侧直接对 `https://api.notion.com/v1/search` 发起 fetch，并且只请求单页 + 过滤后可能得到 0；同时 services 层已有另一套 Notion API helper（分页策略也不同），形成双实现与不一致错误口径。
- 需求目标不是“补丁式加分页/加 spinner”，而是做一次结构性重构：把 parent page discovery 做成单一真源，UI 只消费状态与结果，并在迁移过程中及时删除老旧代码（不拖到最后统一清理）。

# 核心需求（原始需求精炼）

1. **单一真源**：Notion parent pages discovery（search / pagination / filtering / title extraction / savedId resolve）必须收敛到 `webclipper/src/services/**`（并通过 background handler 对 UI 暴露），Settings 不再直连 Notion API。
2. **可见 loading**：点击刷新（以及连接后自动加载）必须有可见 loading 状态；不能只 disable 按钮让用户“看起来没反应”。
3. **去除误导文案**：不再使用“click refresh / clicking xx”类占位文案掩盖状态；空态、loading、error 必须可区分（优先用 icon/spinner/disabled 等无文案方式，不改 i18n）。
4. **分页鲁棒性**：Parent pages 列表必须具备“跨分页寻找可用页”的能力（对齐 macOS App 的策略：当前页找不到可用 page 时继续翻页），避免因为第一页恰好都被过滤而长期空。
5. **连接状态即时更新**：Notion 的 connect/disconnect 按钮点击后必须在当前插件页面内立即更新状态，不能依赖刷新整个插件页面才能同步 UI。
6. **迁移即时清理**：每个迁移 task 必须在同一提交内删除被替代的老代码与调用路径，禁止“先堆新代码、最后再清理”。

# 默认值与兼容策略

- 新安装用户：连接 Notion 后自动加载 parent pages；若用户未选 parent page 且返回列表非空，自动持久化一个默认 parentPageId（与现有行为一致：避免 sync 因缺失 parentPageId 失败）。
- 已有用户升级：保留既有 `notion_parent_page_id` / `notion_parent_page_title`；刷新后若 savedId 不在列表中，仍应尝试 resolve（GET /pages/:id）并把该页注入到 options 顶部（行为保持一致，但实现迁移到 services 单一真源）。
- 未来扩展：保留 `searchQuery` 的扩展点（未来若需要搜索框），但本次不新增 UI。

# 非目标（明确不做什么）

- 不新增“手动输入 Parent Page ID”的设置项（后续若 Notion search 仍存在不可控，可单独开 feature 做兜底）。
- 不改 Notion sync orchestrator 的 DB 创建逻辑与存储键策略（本次只解决 parent page 发现/刷新链路与 UI 体验）。
- 不做 i18n 文案调整/清理（仅移除对相关 key 的引用；是否清理翻译表由后续明确需求决定）。

# 验收标准（可检查）

- 连接 Notion 后自动触发 parent pages 加载，UI 能看到明确 loading 指示（例如 refresh 按钮 spinner）。
- connect/disconnect 点击后无需刷新插件页面，按钮与状态文本能立即更新。
- 点击刷新时：
  - 刷新期间 Select 与刷新按钮呈现一致的 loading 状态（禁用 + spinner）。
  - 若 Notion search 的第一页没有可用 pages，但后续页存在可用 pages，最终能显示出列表（不再卡在空占位）。
- Settings 的 Notion parent pages 列表获取不再直接 `fetch api.notion.com`，而是通过 services/background handler。
- 迁移完成后仓库内不再存在被替代的旧实现（例如 `searchNotionParentPages` / `retrieveNotionParentPage` 等 UI 直连函数与调用点）。
