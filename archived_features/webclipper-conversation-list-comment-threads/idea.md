# 背景 / 触发

- 当前 WebClipper 的会话列表（popup/app 左侧列表）仅展示 title/source/time 等元信息，无法一眼判断某条 **web article** 是否有本地注释（comments）。
- 用户希望在列表层面“看见积累”：哪些文章被留下了多少条“根评论线程”（顶层评论），并把这个数字同步到 Notion / Obsidian 作为可筛选/可检索的元数据。
- 现有 comments 数据已在本地 `article_comments` 表中持久化；同步侧（Notion/Obsidian）在文章流里也会读取 comments（作为正文 sections），但缺少一个稳定、可排序的“线程数”属性。

## 核心需求（原始需求精炼）

1. **会话列表行展示根评论数（仅 web article）**  
   在 popup/app 的会话列表每条行内展示 `💬 <n>`，其中 `n` 为该 article 的 **根评论数**（thread 数）。

2. **根评论数口径稳定**  
   - 根评论定义：`parentId` 为空，或 `parentId` 指向的父评论不存在（父被删/不在同一集合）时，计为根评论。  
   - 只展示/同步根评论数；不展示 reply 数、总条数等其它口径。

3. **列表计数实时刷新**  
   当用户新增/删除评论后，会话列表不需要手动刷新即可更新显示（依赖既有 `UI_EVENT_TYPES.CONVERSATIONS_CHANGED` 刷新节奏）。

4. **同步到 Notion（Number 属性，可排序/筛选）**  
   - 在 Notion 的 `SyncNos-Web Articles` 数据库新增一个 Number 属性（建议命名：`Comment Threads`）。  
   - 同步时将根评论数写入该属性；后续评论变化再次 sync 时应更新。

5. **同步到 Obsidian（YAML frontmatter）**  
   - 在 Obsidian 文章笔记 frontmatter 写入 `comments_root_count: <n>`。  
   - 后续评论变化再次 sync 时应更新该字段。

## 默认值与兼容策略

- 不引入新的本地事实源：根评论数为 `article_comments` 的派生视图，默认在“列表查询按页返回”时计算注入；不新增 conversations 表字段或迁移（方案 1）。
- Notion：若数据库 schema 里不存在 `Comment Threads`，由 `dbSpec.ensureSchemaPatch` 创建/补齐；旧用户升级后下一次同步自动出现属性。
- Obsidian：旧笔记不做离线批量改写；在下一次对该 article 执行同步（append/rebuild）时写入/更新 frontmatter。

## 非目标（明确不做什么）

- 不把 comments 内容额外同步到 Notion/Obsidian（现有 comments sections 行为保持不变，本 feature 只新增“计数元数据”）。
- 不为 chat conversations 展示/同步评论数。
- 不新增 settings 开关、不改 Insight 统计口径。
- 不引入 conversation 列表“全量 materialize + 统计”的新路径（避免性能与一致性风险）。

## 验收标准（可检查）

- UI：
  - 任意 article 会话存在 3 个根评论时，该会话行展示 `💬 3`；无根评论时不显示该 chip。
  - 新增/删除根评论后，会话列表在既有刷新节奏内更新显示（无需手动刷新）。
- Notion：
  - `SyncNos-Web Articles` 数据库出现 `Comment Threads`（Number）属性。
  - 同步某 article 后，该页面属性值与本地根评论数一致；评论变化后二次同步会更新该值。
- Obsidian：
  - 同步某 article 后，生成/更新的笔记 frontmatter 包含 `comments_root_count: <n>`，且与本地根评论数一致；评论变化后二次同步会更新该字段。

