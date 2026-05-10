# Plan P1 - ai-chat-outline-panel

**Goal:** 在 `conversation.sourceType === 'chat'` 的会话详情页渲染目录把手与 overlay 面板，并支持点击目录条目平滑跳转到对应 user 消息。

**Non-goals:**
- 不做滚动自动高亮（P2 做）。
- 不做历史 `sourceType` 缺失兜底推断。
- 不注入原始 AI 网页，仅限本地 UI（app + popup）。

**Approach:**
- 先把“目录条目”抽象成纯函数：输入 `detail.messages`，输出 `{index, messageId, messageKey, previewText}` 列表，便于单测与复用。
- UI 先按示意稿 `webclipper/src/ui/mocks/ai-chat-outline-panel.html` 落地：把手常驻（滚动时仍可触达）、贴右侧、无边框/无阴影；hover 把手展开面板；面板打开时覆盖把手。
- 跳转在 P1 先做到“点击条目 -> `scrollIntoView({behavior:'smooth', block:'center'})`”。

**Acceptance:**
- `conversation.sourceType !== 'chat'` 时不显示把手与面板。
- `conversation.sourceType === 'chat'` 时显示把手（滚动时仍可触达）；hover 把手可打开面板；面板条目与 user 消息数量一致；点击可平滑跳转。

---

<a id="p1-t1"></a>
## P1-T1 定义目录条目模型与纯文本提取工具

**Files:**
- Add: `webclipper/src/ui/conversations/chat-outline/outline-entries.ts`
- (Optional) Add: `webclipper/src/ui/conversations/chat-outline/types.ts`

**Step 1: 实现功能**
- 定义 `ChatOutlineEntry`（最小字段）：
  - `index: number`（从 1 开始）
  - `messageId: number`
  - `messageKey: string`
  - `previewText: string`（前 30 字 + `…`，无换行，trim）
- 提供纯函数：
  - `buildChatOutlineEntries(messages: ConversationMessage[], options?: { maxLen?: number }): ChatOutlineEntry[]`
  - `extractMessagePlainText(message: ConversationMessage): string`
- 纯文本提取规则：
  - 优先 `contentText`（trim 后非空）
  - 否则 fallback `contentMarkdown`：做轻量 markdown->text（移除常见符号、链接 URL、代码围栏标记；保留可读文字即可；不追求完美）
- 截断规则：
  - 以 JS 字符长度为准（不做 grapheme 级切分）
  - `<= 30` 不加 `…`，`> 30` 切前 30 + `…`
  - 永远单行：把 `\n` / `\r` / `\t` 归一为单空格

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: TypeScript 无错误。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/chat-outline/outline-entries.ts`
- Run: `git commit -m "feat: P1-T1 - 定义AI chat目录条目构建工具"`

---

<a id="p1-t2"></a>
## P1-T2 为目录条目构建逻辑补齐单元测试

**Files:**
- Add: `webclipper/tests/unit/chat-outline-entries.test.ts`

**Step 1: 实现功能**
- 覆盖至少这些断言：
  1. 只选 `role === 'user'`，顺序保持原 messages 顺序，index 从 1 开始。
  2. `contentText` 优先级高于 `contentMarkdown`。
  3. 多行文本会被压成单行（空白归一）。
  4. `>30` 会截断并追加 `…`，`<=30` 不追加。
  5. 空/缺失字段时返回空字符串并且该条目仍可生成（previewText 允许为空，但条目存在）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- chat-outline-entries`
- Expected: 测试通过。

**Step 3: 原子提交**
- Run: `git add webclipper/tests/unit/chat-outline-entries.test.ts`
- Run: `git commit -m "test: P1-T2 - 补齐AI chat目录条目构建单测"`

---

<a id="p1-t3"></a>
## P1-T3 实现目录把手与面板 UI（静态渲染 + hover 展开/收回）

**Files:**
- Add: `webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`
- (Optional) Add: `webclipper/src/ui/conversations/chat-outline/styles.ts`（仅当 className 组合需要抽离）

**Step 1: 实现功能**
- 组件职责：
  - 接收 `entries: ChatOutlineEntry[]`
  - 接收 `activeIndex?: number`（P1 先允许为空，P2 再接入）
  - 暴露 `onPickEntry(entry)` 回调用于跳转
- UI 约束（对齐示意稿 `webclipper/src/ui/mocks/ai-chat-outline-panel.html`）：
  - 把手常驻、贴右侧、无边框/无阴影、不呈现 button 视觉（但可用 `<button>`/`role="button"` 做 a11y）
  - hover 触发区域只限把手（不额外加透明热区）
  - 面板是 overlay：不挤占详情宽度；打开时覆盖把手
  - 面板条目单行截断；active（若传入）有轻底色 + 文本高亮
- 状态管理：
  - `open` 状态由把手 `mouseenter/leave` 与面板 `mouseenter/leave` 控制（P3 再加 close delay 防闪）

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: TypeScript 无错误。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/chat-outline/ChatOutlinePanel.tsx`
- Run: `git commit -m "feat: P1-T3 - 增加AI chat目录面板UI组件"`

---

<a id="p1-t4"></a>
## P1-T4 把目录面板接入会话详情页并支持点击平滑跳转

**Files:**
- Modify: `webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Modify: `webclipper/src/ui/conversations/ConversationsScene.tsx`（仅当需要透传信息；能不改就不改）

**Step 1: 实现功能**
- 在 `ConversationDetailPane` 内：
  - 只在 `String(selected?.sourceType).toLowerCase() === 'chat'` 时渲染目录把手与面板。
  - `entries` 数据源来自已加载的 `detail.messages`（不额外查询 IndexedDB）。
  - 把手位置与“常驻”：
    - 优先把目录把手/面板锚定到 sticky header 的右下方（渲染在 header 内，`position: absolute; top: 100%`），确保滚动时把手仍可触达。
    - 当 `hideHeader === true`（窄屏/某些 popup 详情布局）时，改为锚定到内容区容器顶部右侧（与 `containerPaddingClassName` 对齐），同样保持可触达。
  - 点击条目：
    - 先找到对应 user 消息的 DOM（P1 可先用 “messageId -> ref map” 的最小实现）
    - 调用 `scrollIntoView({ behavior: 'smooth', block: 'center' })`
- 注意：为避免 P1/P2 重复改动消息结构，P1 仅为 `role === 'user'` 增加最小 wrapper+ref（用于点击跳转）；P2 在同一层 wrapper 上补齐 `data-*` 与观察列表，不再二次包裹。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Expected: 编译与单测通过。

**Step 3: 原子提交**
- Run: `git add webclipper/src/ui/conversations/ConversationDetailPane.tsx`
- Run: `git commit -m "feat: P1-T4 - 在详情页接入AI chat目录并支持点击跳转"`

---

## Phase Audit

- Audit file: `audit-p1.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（`npm --prefix webclipper run compile && npm --prefix webclipper run test`）
