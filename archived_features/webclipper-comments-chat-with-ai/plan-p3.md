# Plan P3 - webclipper-comments-chat-with-ai

**Goal:** comment 级 `Chat with AI` 发送给 AI 的 payload 从“粗粒度”升级为“精准上下文”（引用原文 + 我的想法 + 文章元信息），并确保截断与边界正确。

**Non-goals:**
- 本 phase 不做 tab group / tab 复用（Phase 4 做）
- 本 phase 不做自动填入 AI 输入框并发送

**Approach:**
- 使用现有 Chat with AI 的模板渲染能力 `renderChatWithTemplate()` + 截断 `truncateForChatWith()`。
- 不新增新的 settings key：将精准上下文拼装为 `article_content`，复用现有 `promptTemplate/maxChars/platforms` 配置。

**Acceptance:**
- payload 格式符合 spec：quote/comment/title/url
- 无 quote 的 root comment 正确降级
- 超限正确截断并携带 truncated suffix
- 至少为 payload builder 增加单元测试（放在 `tests/unit/**`）

---

## P3-T1 实现精准payload(quote+comment+文章元信息)

**Files:**
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-comment-payload.ts`

**Step 1: 实现功能**
- 在 `chatwith-comment-payload.ts` 中构造 comment 精准上下文，并用 settings 的模板渲染（不修改 `chatwith-settings.ts`，避免影响文章级 chatwith）：
  - 将精准上下文拼装成 `article_content` 变量
  - `article_title/article_url` 来自 article 会话上下文（拿不到则降级为空/占位，但不要抛异常）
- 精准上下文默认拼装（示例）：
  - 引用原文（可空）
  - 我的想法（commentText）
  - 文章标题与 canonicalUrl
  - 保持输出以 `\n` 结尾（与现有 Chat with AI payload 行为一致）

**Step 2: 验证**
Run: `npm --prefix webclipper run compile`

Expected: 编译通过

**Step 3: 原子提交**
Run: `git add webclipper/src/services/integrations/chatwith/chatwith-comment-payload.ts`

Run: `git commit -m "feat: P3-T1 - comment级chatwith生成精准上下文payload"`

---

## P3-T2 补齐边界与截断策略并加单测

**Files:**
- Add: `webclipper/tests/unit/chatwith-comment-payload.test.ts`
- Modify: `webclipper/src/services/integrations/chatwith/chatwith-comment-payload.ts`

**Step 1: 实现功能**
- 边界：
  - quoteText 为空：不输出“引用原文”段，或输出但为空（以 spec 为准）
  - commentText 为空：按钮 action disabled（避免复制空内容）
  - canonicalUrl 为空：仍能生成 payload（URL 行为空或省略）
  - articleTitle 为空：仍能生成 payload（标题行可为空或占位，但不得抛异常）
- 单测覆盖：
  - 有 quote / 无 quote
  - 截断行为（maxChars 很小的情况）
  - 模板渲染包含 title/url/content
  - 输出末尾换行与截断 suffix 行为

**Step 2: 验证**
Run: `npm --prefix webclipper run test -- tests/unit/chatwith-comment-payload.test.ts`

Run: `npm --prefix webclipper run compile`

Expected: 测试通过

**Step 3: 原子提交**
Run: `git add webclipper/tests/unit/chatwith-comment-payload.test.ts webclipper/src/services/integrations/chatwith/chatwith-comment-payload.ts`

Run: `git commit -m "test: P3-T2 - comment级chatwith payload与截断单测"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（`compile + test`）
