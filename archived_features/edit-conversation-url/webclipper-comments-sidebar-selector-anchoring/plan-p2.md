# Plan P2 - webclipper-comments-sidebar-selector-anchoring

**Goal:** 提升 selector 定位在动态页面/复杂滚动容器下的成功率与体验一致性。

**Non-goals:** 不引入跨环境定位；不引入高亮。

**Approach:** 在 P1 基础上增强：滚动容器识别、失败重试与“误匹配”防护；打磨轻提示 UX 与可访问性；补充手动回归清单（hostile CSS）。

**Acceptance:**
- inpage 常见动态站点：第一次定位失败时可重试后成功（或明确提示失败），不出现误滚/滚 sidebar 自己。
- app：定位不会在 sidebar 内命中错误文本；只滚动内容区。

---

## P2-T1 inpage：增强滚动容器处理与失败重试（动态加载更稳）

**Files:**
- Modify: `webclipper/src/services/comments/threaded-comments-panel.ts`

**Step 1: 实现功能**

- 当 `toRange` 失败或 `scrollIntoView` 后仍未接近目标时：
  - 增加一次轻量重试（带短延迟），并在重试前尝试触发内容渲染（例如先滚到 position hint 附近）
- 对“内部滚动容器”的情况做更稳处理（避免只滚 window）：
  - 优先对 range 所在元素使用 `scrollIntoView`（让浏览器选择正确的滚动祖先）

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；动态站点手动验证成功率提升。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/threaded-comments-panel.ts`

Run: `git commit -m \"feat: task1 - 强化inpage selector定位的滚动与重试\"`

---

## P2-T2 app：增强定位 root 选择与防误匹配（避免侧边栏内命中）

**Files:**
- Modify: `webclipper/src/ui/app/AppShell.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/ArticleCommentsSection.tsx`

**Step 1: 实现功能**

- 明确 app 的 anchoring root 只包含“消息内容区”（不包含 sidebar DOM）。
- 对 selector 还原加一层 guard：
  - 还原出来的 range 必须落在内容区 root 内，否则视为失败（提示）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；app 中不会滚到 sidebar 内的同名文本。

**Step 3: 原子提交**

Run: `git add webclipper/src/ui/app/AppShell.tsx webclipper/src/ui/conversations/ConversationDetailPane.tsx webclipper/src/ui/conversations/ArticleCommentsSection.tsx`

Run: `git commit -m \"feat: task2 - app侧selector定位防误匹配\"`

---

## P2-T3 UX：轻提示样式与可访问性打磨（不打断输入）

**Files:**
- Modify: `webclipper/src/services/comments/threaded-comments-panel.ts`
- Modify: `webclipper/src/ui/styles/inpage-comments-panel.css`（若需要）

**Step 1: 实现功能**

- 轻提示规则：
  - 不用 `alert`
  - 不抢焦点、不阻止输入
  - 自动消失（可选）
- 文案最小化：例如“无法定位”（避免引入 i18n，若必须国际化再单独提任务）。

**Step 2: 验证**

Run: `npm --prefix webclipper run compile`

Expected: 编译通过；提示不影响回复/删除/发送操作。

**Step 3: 原子提交**

Run: `git add webclipper/src/services/comments/threaded-comments-panel.ts webclipper/src/ui/styles/inpage-comments-panel.css`

Run: `git commit -m \"feat: task3 - 评论定位失败的轻提示体验打磨\"`

---

## P2-T4 回归：hostile CSS / 典型站点（ChatGPT/Notion/Medium）手动验证清单

**Files:**
- Modify: `.github/features/webclipper-comments-sidebar-selector-anchoring/audit-p2.md`

**Step 1: 实现功能**

- 在 `audit-p2.md` 里记录本 feature 的手动回归 checklist（仅记录，不入库提交默认可跳过）。

**Step 2: 验证**

- 手动验证（至少一次）：
  - inpage：ChatGPT（内部滚动容器）、Notion（复杂 DOM）、Medium（长文）
  - app：选择文本 -> 写根评论 -> 点击定位

**Step 3: 原子提交**

- 本任务默认不提交代码（只更新本地 audit 文档）；如需提交由用户另行指示。

---

## Phase Audit

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环

