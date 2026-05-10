# 背景 / 触发

- 当前 SyncNos 的核心会话（conversation）分为 `web articles`、`AI chats`、`video transcript` 三类；它们在产品语义上应当是**同一级别的一等公民**（同样的列表、详情、导出、同步、评论能力），而不是“article 优先，video/chat 例外”。
- 目前代码与数据层仍存在明显的 “article-only” 假设与命名（例如 `article_comments`、`ArticleCommentsSection`、`ArticleCommentsSidebar*`、Notion layout 对 video 只有 Transcript），导致 video/chat 在功能可用性与同步链路上被降级处理。
- 近期暴露的具体问题：
  - Video 会话缺少右侧评论侧边栏入口/闸门（之前仅对 article 打开）。
  - Video transcript 导出/展示出现额外的 `transcript` 标题层级（本质是 markdown/导出层把 `role=transcript` 当作普通 chat role 生成了标题）。
  - Notion/Obsidian 同步逻辑只在 `kind=article` 时加载并同步 comments（video/chat 缺失 comments section）。

# 核心需求（原始需求精炼）

1. **同级能力（Video 一等公民）**：`video transcript` 必须与 `web articles`、`AI chats` 同级：详情页/导出/同步/评论等关键能力不应有“只对 article 开放”的硬编码闸门。
2. **根因级建模（统一 kind/target）**：对“评论目标（comment target）/canonical target”的建模必须由 conversation kind 推导，不再绑定 `article-only` 命名与数据结构。
3. **正确 canonicalization**：
   - article 使用 `canonicalizeArticleUrl`
   - video 使用 `canonicalizeVideoUrl`
   - chat 使用稳定的 http url 规范化（至少 `normalizeHttpUrl`/去 hash；如已有 chat canonicalizer 则复用并在单点收敛）
4. **Orphan comments 语义保留**：允许在 “尚未 capture conversation” 的页面上先写评论（orphan），后续 capture 成 conversation 后能自动 attach 到该 conversation。
5. **同步一等公民**：Notion/Obsidian 同步对 video/chat 也应能同步 comments（至少具备 section + digest + rebuild）。
6. **Video transcript 标题问题根治**：对 `sourceType=video` 的 markdown 导出/呈现，不能再因为 `role=transcript` 注入额外标题层级。

# 默认值与兼容策略

- 兼容策略（必须）：
  - 现有 `article_comments` 数据不能丢失。
  - 升级后应可继续读取并展示旧 comments。
  - 新模型上线后，旧 message type/旧 adapter 允许短期保留为兼容 wrapper（避免一次性大爆炸改动）。
- 默认行为（目标）：
  - app 侧右侧 comments sidebar：对所有有稳定 comment target 的 conversation 默认可用（非 narrow），包括 video/chat。
  - popup “打开当前页评论侧边栏”：从 “article only” 放宽为 “当前页可捕获/可定位的 conversation kind”（至少包括 video 页面与 ai chat 页面）。

# 非目标（明确不做什么）

- 不做新的 UI 设计/视觉重做（保持 ThreadedCommentsPanel 现有 UI）。
- 不引入新的评论类型（仍然是 quote + commentText + replies）。
- 不强制为 chat/video 立即引入复杂 locator（第一版允许 locator 为 null，或仅复用现有 TextQuote/TextPosition selector）。
- 不做跨设备/云端 comments 同步（仍然是 local-first + Notion/Obsidian output）。

# 验收标准（可检查）

- App（宽/中屏）：
  - 选择任一 `sourceType=article|video|chat` 的 conversation，右侧 “评论” 按钮可用，能打开右侧 sidebar，并能新增/回复/删除评论。
- Orphan attach：
  - 在页面侧边栏（inpage）先写 comments（无 conversationId），后续 capture 生成对应 conversation 后，comments 能自动出现在该 conversation 的 app sidebar 中。
- 导出：
  - 对 `sourceType=video` 导出的 markdown 不包含 `## transcript` / `### transcript` 这类由 role 注入的标题。
- 同步：
  - Notion/Obsidian 同步对 video/chat 具备 comments section，并在 comments 变化时触发 digest 更新与 rebuild（与 article 同级别的更新语义）。

# 移交备注（给低上下文执行者）

- 不要提交 `.github/features/webclipper-video-kind-parity/*` 计划文件（除非用户明确要求）。
- 所有 shell 命令必须用 `rtk` 前缀。
- 先做 “数据模型 + 协议 + 存储” 的抽象再改 UI；不要在 UI 层继续堆 `if (sourceType===...)` 的 patch。
- DB schema 变更必须通过 `DB_VERSION` 升级 + migration；不能靠“运行时偷偷修”。
- 保留旧数据/旧入口：先实现新能力并保持兼容，再逐步删旧 wrapper（分 phase）。

## 关键风险/决策点（必须在 P1 明确）

- **Chat 的 comments target 不能简单用 `normalizeHttpUrl(location.href)`**：
  - 部分 AI chat 站点的 URL 可能不唯一/不稳定（同一域名不同 thread 可能共享 URL 或带易变参数）。
  - 若用 URL 作为 key，可能把不同 chat thread 的 comments 合并到同一条目标上。
- 建议在 comments 领域引入 `commentTargetKey`（字符串）并支持至少两类：
  - `url:<canonicalUrl>`（article/video 主用）
  - `convo:<source>:<conversationKey>`（chat 主用；当 conversation 已存在时优先）
- 对 “inpage 未 capture 时的 orphan comments”：
  - 允许使用 `url:` 写入 orphan
  - capture 成 conversation 后，再把同一 `url:` 下的 orphan attach 到该 conversation（并在可能时迁移/归并到 `convo:` key）
