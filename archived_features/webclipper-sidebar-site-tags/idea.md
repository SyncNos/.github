# 背景 / 触发

- WebClipper 的 Conversations 左侧列表里，`web` 来源目前只显示一个笼统的 `Web/网页` tag，无法快速看出文章来自哪个站点。
- 同时，左侧列表的 collectors/source tag 配色需要更贴近各 AI 平台的品牌主色；但 `web` 的 tag 颜色要求固定一种颜色，不随站点变化。
- 这次改动希望以“破坏性重构”为主：抽离/重组相关逻辑，并在每个 task 内及时删除旧代码，避免最后再统一清理导致双实现长期并存。

# 核心需求（原始需求精炼）

1. 在 Conversations 左侧列表中，`web` 类型条目的 tag 文案显示更具体的站点信息（优先显示域名/hostname）。
2. `web` 的 tag 颜色固定一种，不因站点不同而改变。
3. 非 `web` 的 collectors/source tag 颜色按各平台“品牌主色”统一（ChatGPT/Claude/DeepSeek/Notion AI/Gemini/Google AI Studio/Kimi/豆包/元宝/Poe/z.ai）。
4. 尽可能复用已有数据与代码路径，不引入新依赖（例如不新增 eTLD+1 解析库）。
5. 做破坏性重构：旧的 tag 颜色/文案计算逻辑在迁移后立即删除，不保留长期兼容分支。

# 默认值与兼容策略

- 新安装与升级用户一致：列表展示逻辑基于现有数据字段计算，不新增设置项。
- 站点提取策略：
  - 优先复用 `Conversation.listSiteKey`（现有存储会从 `conversation.url` 推导 `domain:<hostname>`）。
  - 若 `listSiteKey` 不可用，则回退用现有 `parseHostnameFromUrl(conversation.url)` 提取 hostname。
  - 若仍无法得到 hostname，则显示 `Unknown`（复用现有文案 key，例如 `t('insightUnknownLabel')`）。
- 不做 IndexedDB/DB 版本迁移：缺失 `url/listSiteKey` 的历史记录按 Unknown 展示即可。

# 非目标（明确不做什么）

- 不做 “站点友好名映射”（例如把 `medium.com` 显示成 `Medium`）。
- 不改 `site` facets / 筛选菜单的行为与数据口径（只改列表行 tag 展示与配色）。
- 不引入/维护一个“每站点不同颜色”的体系；`web` 始终固定同色。
- 未明确要求时，不改 i18n 字段与现有翻译。

# 验收标准（可检查）

- 在 Conversations 左侧列表中：
  - `web` 来源的条目，tag 文案显示为域名（例如 `example.com`），而不是 `Web/网页`。
  - `web` 的 tag 配色固定，不随站点变化。
  - 非 `web` 的 tag 配色与品牌主色一致，并保持可读性（浅色/深色主题下均可辨识）。
- 代码层面：
  - `ConversationListPane` 中旧的 tag tone/label 计算逻辑已被替换并删除，不存在双实现长期并存。
  - 新逻辑以可复用的小模块/函数承载，便于后续在 app/popup 一致复用。

