# Plan Review (2026-04-02) - webclipper-conversation-list-comment-threads

- 审计方式：plan-level read-only review（实现前）
- 审计范围：`.github/features/webclipper-conversation-list-comment-threads/{idea,todo,plan-p1,plan-p2,plan-p3,audit-p1,audit-p2,audit-p3}`
- 目标：揪出计划中的实现/验证不可执行点与潜在回归风险，并回写到 plan 文档本身

## Findings

## 发现 F-01

- 任务：`P1/P2/P3（验证命令）`
- 严重级别：`High`
- 状态：`Resolved`
- 位置：`webclipper/package.json:11`
- 摘要：plan/audit 中使用 `npm --prefix webclipper run test -- --run <file>`，但当前脚本为 `vitest run`，`--run` 会被当成未知参数，导致验证命令不可执行。
- 风险：执行阶段会在验证环节直接失败，影响 `executing-plans` 的“每 task 必须可验证”的闭环。
- 预期修复：把所有 target tests 命令改为 `npm --prefix webclipper run test -- <file> [more files...]`。
- 验证：`npm --prefix webclipper run test -- tests/smoke/obsidian-markdown-writer.test.ts`
- 解决证据：已回写 plan/audit 文档（不涉及代码实现）。

## 发现 F-02

- 任务：`P2-T2`
- 严重级别：`High`
- 状态：`Resolved（plan 已修正）`
- 位置：`webclipper/src/services/sync/notion/notion-sync-orchestrator.ts:~650/~900`
- 摘要：当前 orchestrator 在 `createPageInDatabase/updatePageProperties` 之前就调用 `pageSpec.buildCreateProperties/buildUpdateProperties`；若按旧 plan 在后面才计算并注入 `commentThreadCount`，则 `Comment Threads` 无法写入（或永远为 0）。
- 风险：Notion 验收标准（属性值与本地根评论数一致）无法达成。
- 预期修复：在 create/update 路径中，确保在 build properties 前先读取 comments 并计算 root count，再注入 `convo.commentThreadCount`。
- 验证：`npm --prefix webclipper run test -- tests/smoke/notion-sync-orchestrator-kind-routing.test.ts`
- 解决证据：`plan-p2.md` 已更新，要求 properties 构建前完成注入，并尽量复用已读取的 comments。

## 发现 F-03

- 任务：`P2-T2`
- 严重级别：`Medium`
- 状态：`Resolved（plan 已补齐）`
- 位置：`webclipper/src/services/sync/notion/notion-sync-orchestrator.ts:206`
- 摘要：`pagePropertiesNeedUpdate()` 当前未对 Notion `number` 属性做值级比较，会因结构差异（page 返回包含 `id/type` 等字段）而倾向于每次都触发 `updatePageProperties`。
- 风险：额外 API 调用与潜在 rate limit；难以用测试验证“仅变化时更新”。
- 预期修复：补齐 `normalizePagePropertyValue()` 对 `prop.number` 的比较口径（至少支持 `{ number: <n> }`）。
- 验证：`npm --prefix webclipper run test -- tests/smoke/notion-sync-orchestrator-kind-routing.test.ts`
- 解决证据：`plan-p2.md` 已明确要求在同一 task 内补齐 number 比较。

## 发现 F-04

- 任务：`P1-T3`
- 严重级别：`High`
- 状态：`Resolved（plan 已补齐）`
- 位置：`webclipper/src/services/conversations/data/storage-idb.ts:~880`
- 摘要：conversation list 的读取 transaction 目前只包含 `conversations` store；按页注入 comments 需要读取 `article_comments`，必须扩展 tx storeNames 或明确采用独立 transaction 策略。
- 风险：实现时容易出现“store 不在 transaction 中”或被迫为每条 item 额外 openDb/开 tx，造成性能与一致性问题。
- 预期修复：把 `readConversationListPage()` 的 tx 调整为 `['conversations','article_comments']` 并在按页注入中使用 `article_comments` store 的 index 查询。
- 验证：`npm --prefix webclipper run test -- tests/storage/conversations-pagination.test.ts`
- 解决证据：`plan-p1.md` 已补充 transaction 扩展与 canonicalize 口径说明。

## Notes / Remaining Risks

- `Notion DB` 若已存在同名属性 `Comment Threads` 但类型不是 `number`，当前 db schema patch 只会“补齐缺失”，不会修正类型冲突；这会导致 sync 可能报错。建议实现阶段视情况增加类型冲突检测与更明确的错误提示（可作为后续增强）。
- `deleteArticleCommentById()` 的级联删除目前是“删除 id + 直接 child”，若未来支持多层 reply，可能残留更深层的 orphan replies（会影响 root count 语义）。本 feature 目前按既定口径（orphan reply 视为 root）可接受，但建议后续单独审视删除策略。

