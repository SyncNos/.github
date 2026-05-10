# Plan P1 - webclipper-sidebar-site-tags

**Goal:** 在 Conversations 左侧列表中让 web articles 显示站点域名，并将非 web 的 collectors/source tag 配色改为品牌主色，且以破坏性重构方式清理旧逻辑。

**Non-goals:** 不引入站点友好名映射、不改 facets/筛选口径、不做每站点不同颜色、不新增依赖库。

**Approach:** 把 Conversations 列表行的 tag “文案 + tone(颜色)”计算从 `ConversationListPane` 中抽离为可复用 helper，并在同一 task 内删除旧实现；优先一次性把 `web` 的 label 改成 hostname（避免中间态），随后引入品牌色 CSS tokens 并切换非 web 渲染；`web` 的 tone 始终固定，不随站点变化。

**Acceptance:**
- web 条目 tag 显示为 `example.com` 这样的 hostname，且颜色固定。
- 非 web tag 使用对应平台品牌主色（浅色/深色主题均可读）。
- `ConversationListPane` 不再保留旧的 tag 计算逻辑（无双实现残留）。

---

## 品牌色（需确认，可随时改 HEX）

> 这些颜色是“UI 标签识别色”的默认提案；最终以你确认的品牌主色为准。实现上会集中到 `tokens.css` 的 `--brand-xxx`，后续改色只改一处。

- ChatGPT: `#10A37F`
- Claude: `#D97757`
- DeepSeek: `#6B7280`
- Notion AI: `#111111`
- Gemini: `#4285F4`
- Google AI Studio: `#4285F4`
- Kimi: `#3B82F6`
- 豆包: `#F97316`
- 元宝: `#EF4444`
- Poe: `#EC4899`
- z.ai: `#14B8A6`

---

## P1-T1 破坏性重构：抽离列表 tag 计算入口 + web 显示站点域名（删旧逻辑）

**Files:**
- Add: `webclipper/src/ui/conversations/conversation-list-tags.ts`
- Modify: `webclipper/src/ui/conversations/ConversationListPane.tsx`

**Step 1: 实现功能**

- 新增 `conversation-list-tags.ts`，提供单一入口函数（例如 `resolveConversationListTag(conversation, { t })`）输出：
  - `label`: 列表行 tag 文案
  - `toneClassName`: 列表行 tag 的 Tailwind class
- `label` 规则：
  - 优先用 `conversation.listSourceKey ?? conversation.source` 归一化得到 `sourceKey`
  - 当 `sourceKey === 'web'`：显示站点域名
    - 优先 `conversation.listSiteKey`（`domain:<hostname>` → 剥掉 `domain:`；`unknown/空` → `t('insightUnknownLabel')`）
    - 回退 `parseHostnameFromUrl(conversation.url)`（复用现有工具）
    - 仍拿不到时：`t('insightUnknownLabel')`
  - 非 `web`：沿用现有 i18n 文案（复用当前 `getSourceMeta` 的映射，但迁移到新文件里）
- 在 `ConversationListPane.tsx` 中统一改为调用新入口：
  - 列表行 tag（替换 `sourceLabel + sourceTagToneClass(sourceKey)`）
  - source filter options（替换 `getSourceMeta` 的复用点，避免删旧函数后漏改）
- **破坏性清理（同一个 task 内完成）**：删除 `ConversationListPane.tsx` 里的 `SourceMeta`/`getSourceMeta`/`sourceTagToneClass` 等旧实现与相关导入残留。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: TypeScript 编译通过，无未使用函数/导入残留。

---

## P1-T2 为非 web collectors/source tag 引入品牌色 tokens 并切换渲染

**Files:**
- Modify: `webclipper/src/ui/styles/tokens.css`
- Modify: `webclipper/src/ui/conversations/conversation-list-tags.ts`

**Step 1: 实现功能**

- 在 `tokens.css` 的 `:root` 中新增一组“品牌色” CSS 变量（例如 `--brand-chatgpt`、`--brand-claude` 等），值使用上面已确认的 HEX。
  - 这些变量不需要在 dark media 内重复定义（保持常量）；背景/边框通过 `color-mix` 与 `--bg-card/--border` 自适配。
- 在 `conversation-list-tags.ts` 中把非 web 的 `toneClassName` 切换为基于 `--brand-*` 变量生成（沿用当前 tailwind arbitrary value 的写法）：
  - `border`: `tw-border-[var(--brand-xxx)]`（或混到 `--border`，按实际观感定）
  - `bg`: `tw-bg-[color-mix(in_srgb,var(--brand-xxx)_12%,var(--bg-card))]`
  - `text`: `tw-text-[var(--text-primary)]`
- `web` 的 `toneClassName` 保持固定（不使用 `--brand-*`，不按站点变化）。
- **破坏性清理（同一个 task 内完成）**：移除 helper 中旧的语义色映射（例如 `--info/--success/...`）在“source tag”上的使用；但不移除 tokens.css 中的语义色变量（它们仍可能被其他组件使用）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；`ConversationListPane` 中的 tag 样式仍正常渲染，且非 web 颜色明显按品牌色变化。

---

## P1-T3 为 tag 解析与配色添加单测并跑通验证链路

**Files:**
- Add: `webclipper/tests/unit/conversation-list-tags.test.ts`
- Modify: `webclipper/src/ui/conversations/conversation-list-tags.ts`（如需额外导出纯函数以便测试）

**Step 1: 实现功能**

- 添加单测覆盖：
  - `web`：`listSiteKey=domain:example.com` → label `example.com`；`listSiteKey=unknown`/缺失时回退 url；都缺失时为 `t('insightUnknownLabel')`
  - 非 `web`：tone class 中包含对应 `--brand-xxx` 变量引用（至少测 3 个 key）
  - `web`：tone 不包含 `--brand-*`（固定样式）
- 如测试需要，把“sourceKey 归一化 / site label 推导 / tone 拼装”拆成纯函数并导出；主入口 API 保持单一入口。

**Step 2: 验证**

Run: `npm --prefix webclipper run test`

Expected: 新增测试通过，且不引入 flaky 行为。

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
