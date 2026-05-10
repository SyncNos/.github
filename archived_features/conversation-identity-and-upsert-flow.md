# 会话同一性判定与 Update/Add 机制说明（WebClipper）

> 目标：回答“如何判断是不是同一条对话？”以及“为什么有时是更新（update）而不是新增（add/save）？”

## 1. 先说结论

- 当前系统对“同一条会话”的主判定键是：`(source, conversationKey)`。
- 只要同平台（`source`）且 `conversationKey` 不变，`upsertConversation()` 就会走 **UPDATE**。
- 一旦 `conversationKey` 发生漂移（例如 URL 形态变化、fallback key 切换），就可能走 **ADD**，形成“看起来重复”的会话。

---

## 2. 为什么不同平台差异很大

根因是：各平台 URL 结构和可提取的“稳定 ID”能力不一致。

1. **有稳定会话 ID 的平台**（如 ChatGPT、z.ai、NotionAI）
   - 可以直接从 URL 里提 ID（如 `/c/{id}` 或 `?t={threadId}`）。
   - 这类平台理论上稳定性更好。

2. **没有明确会话 ID、更多依赖 pathname 的平台**（Claude/Gemini/Kimi/DeepSeek/Poe/Doubao/Yuanbao/Google AI Studio）
   - 常见做法是 `conversationKeyFromLocation(location)`，主要由 `pathname` 推导。
   - 若 pathname 改版、路由重写、同会话不同入口，key 可能漂移。

3. **需要 fallback 的场景**
   - 例如 ChatGPT 在某些时刻提不到 `/c/{id}`，会退到 `fallback_${hash(host|path|firstUserText)}`。
   - fallback 本质是“临时身份”，一旦后续拿到真实 ID，若不做映射/迁移，容易被当成两条会话。

---

## 3. 各平台“同一性判定”策略总表

| 平台 | 主要 `conversationKey` 来源 | URL 主导 | fallback | 漂移风险 |
|---|---|---|---|---|
| ChatGPT | `findConversationIdFromUrl()` 提取 `/c/{id}` 或 `/g/.../c/{id}` | 是 | `makeFallbackConversationKey(messages)`（hash） | 高（主要在 fallback 阶段） |
| NotionAI | `findChatThreadIdFromHref()` 提取 `t=32hex` | 是 | `notionai_${pageId/path}_${firstUserSeed}` | 中高 |
| z.ai | `findConversationIdFromUrl()` | 是 | `conversationKeyFromLocation()` | 中 |
| Claude | `conversationKeyFromLocation()` | 是 | 同函数内部兜底 href | 中 |
| Gemini | `conversationKeyFromLocation()` | 是 | 同上 | 中 |
| Google AI Studio | `conversationKeyFromLocation()` | 是 | 同上 | 中 |
| DeepSeek | `conversationKeyFromLocation()` | 是 | 同上 | 中 |
| Kimi | `conversationKeyFromLocation()` | 是 | 同上 | 中 |
| Doubao | `conversationKeyFromLocation()` | 是 | 同上 | 中 |
| Yuanbao | `conversationKeyFromLocation()` | 是 | 同上 | 中 |
| Poe | `conversationKeyFromLocation()` | 是 | 同上 | 中 |

`conversationKeyFromLocation()`（`webclipper/src/collectors/collector-utils.ts`）核心行为：
- 优先使用 `pathname`（将 `/` 转成 `_`）
- pathname 不可用时，退到 `href` 去掉 query 的部分

---

## 4. 当前“Update vs Add”是怎么判断的

### 4.1 Conversation 级别

入口：`webclipper/src/services/conversations/data/storage-idb.ts` 的 `upsertConversation(payload)`

判定链：

1. 调 `findExistingConversationForPayload(conversationsStore, payload)`
2. 取 `source` + `conversationKey`（article 类型有 canonical URL 特殊处理）
3. 用索引 `by_source_conversationKey` 查询是否已存在
4. 若命中：`stores.conversations.put(record)`（**UPDATE**）
5. 若未命中：`stores.conversations.add(record)`（**ADD**）

索引定义（唯一）：
- `webclipper/src/platform/idb/schema.ts`
- `conversations` store 的 `by_source_conversationKey = ['source', 'conversationKey']`，`unique: true`

这就是“怎么做到 updated 而不是 saved（新增）”的核心：
- **key 稳定 => update**
- **key 漂移 => add**

### 4.2 Message 级别

入口同文件：`syncConversationMessages(...)`

- 每条消息按 `(conversationId, messageKey)` 判断是否存在
- 命中则 `put`（更新）
- 未命中则 `add`（新增）

对应唯一索引：
- `messages` store 的 `by_conversationId_messageKey`，`unique: true`

---

## 5. 关键流程图

```mermaid
flowchart TD
    A[开始采集 capture()] --> B{平台是否能提取结构化会话ID?}
    B -- 是 --> C[使用结构化ID\n如 chatgpt /c/{id}, notion t=threadId]
    B -- 否 --> D{是否可用URL路径稳定派生?}
    D -- 是 --> E[conversationKeyFromLocation(pathname/href)]
    D -- 否 --> F[使用fallback key\n如 hash(host|path|firstUser)]

    C --> G[形成 conversation: source + conversationKey + title + url]
    E --> G
    F --> G

    G --> H[upsertConversation(payload)]
    H --> I[findExistingConversationForPayload]
    I --> J{by_source_conversationKey 是否命中?}

    J -- 命中 --> K[UPDATE: conversations.put(existing.id)]
    J -- 未命中 --> L[ADD: conversations.add(new id)]

    K --> M[syncConversationMessages\n按 messageKey upsert]
    L --> M
    M --> N[完成]

    O[风险点: key漂移\nURL变化/从fallback切到真实ID] -.-> J
```

---

## 6. 关键代码入口索引（便于继续追）

- 采集器总表：`webclipper/src/collectors/ai-chat-sites.ts`
- 通用 URL key 工具：`webclipper/src/collectors/collector-utils.ts`
- ChatGPT key 逻辑：`webclipper/src/collectors/chatgpt/chatgpt-collector.ts`
- NotionAI key 逻辑：`webclipper/src/collectors/notionai/notionai-collector.ts`
- 其它平台（Claude/Gemini/...）key 逻辑：各自 `*-collector.ts` 的 `findConversationKey()`
- Conversation upsert：`webclipper/src/services/conversations/data/storage-idb.ts`
- IDB schema 与唯一索引：`webclipper/src/platform/idb/schema.ts`

---

## 7. 一句话解释你最关心的问题

“同一条对话”目前不是靠 title、也不是靠 message 内容本身，而是靠 **`source + conversationKey`**；只要这对键稳定，系统就 update，不稳定就 add。
