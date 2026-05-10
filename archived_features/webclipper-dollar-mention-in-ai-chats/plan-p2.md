# 计划 P2 - webclipper-dollar-mention-in-ai-chats

**目标：** 完成 content 侧交互闭环（触发、候选窗、键盘导航、触发片段替换）并在 ChatGPT 先跑通端到端。

**非目标：** 本阶段不实现 NotionAI 的 `contenteditable` 适配。

**实现思路：**
- 先把 `$` 触发解析与会话状态机独立成纯逻辑模块，确保行为可单测。
- 再实现小型候选窗（轻量、可键盘操作）与异步搜索联动。
- 最后接入 ChatGPT Composer 适配器（`contenteditable #prompt-textarea`），完成“替换 `$query` 并在光标位置插入”的闭环。

**验收标准：**
- 输入 `$` 立即弹窗，继续输入实时过滤。
- `↑/↓` 导航，`Tab/Enter` 选中并插入，`Esc` 仅关闭窗口且保留文本。
- ChatGPT 可稳定替换触发片段并插入完整 Markdown，且插入后宿主编辑器能感知内容变化（必要时触发 `input` 事件）。
- 不遗留多余逻辑：候选窗关闭/停用时必须清理事件监听与 DOM（不允许“挂在页面上但不可见”的残留）。

---

<a id="p2-t1"></a>
## P2-T1 实现 `$` 触发会话状态机与触发片段跟踪

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/content/mention-session.ts`
- Add: `webclipper/src/services/integrations/item-mention/content/mention-trigger-parser.ts`
- Add: `webclipper/tests/unit/item-mention-session.test.ts`

**Step 1: 实现**
1. 定义触发规则：输入框中出现 `$` 即可开启 mention 会话。
2. 跟踪会话字段：`triggerStart`、`triggerEnd`、`query`、`open`、`highlightIndex`。
3. 会话更新支持光标移动与文本编辑，始终能回溯当前 `$query` 片段边界。
4. `Esc` 行为仅变更会话状态为关闭，不改输入框文本。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/item-mention-session.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 触发/关闭/片段边界跟踪稳定。

**Step 3: 原子提交**
- `feat: task6 - 实现mention触发状态机与片段跟踪`

---

<a id="p2-t2"></a>
## P2-T2 实现小型候选窗口与键盘导航高亮

**Files:**
- Add: `webclipper/src/ui/inpage/inpage-item-mention-shadow.ts`
- Add: `webclipper/src/ui/styles/inpage-item-mention.css`
- Modify: `webclipper/src/ui/styles/tokens.css`（如需补 token）
- Add: `webclipper/tests/unit/item-mention-ui-state.test.ts`

**Step 1: 实现**
1. 在独立 shadow root 中渲染轻量候选窗，避免宿主页面样式污染。
2. 渲染候选项字段：标题、来源、域名（与过滤字段对齐）。
3. 支持键盘高亮索引与滚动跟随：`ArrowUp/ArrowDown`。
4. 会话关闭时清理浮层与事件监听，避免泄漏。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/item-mention-ui-state.test.ts`
- Run: `npm --prefix webclipper run compile`
- Run: `rg -n "__webclipperItemMention|webclipper-inpage-item-mention" webclipper/src || true`（确认唯一挂载点与可清理逻辑）
- Expected: 候选窗渲染、导航高亮、销毁清理通过。

**Step 3: 原子提交**
- `feat: task7 - 实现mention候选窗口与键盘高亮导航`

---

<a id="p2-t3"></a>
## P2-T3 实现 ChatGPT Composer（contenteditable）光标替换插入适配器

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/content/editor-adapter.ts`
- Add: `webclipper/src/services/integrations/item-mention/content/editor-chatgpt.ts`
- Add: `webclipper/tests/unit/item-mention-chatgpt-adapter.test.ts`

**Step 1: 实现**
1. 抽象 adapter 接口：`detectActiveEditor`、`getSelectionRange`、`replaceRange`、`focus`。
2. ChatGPT 适配器定位 `contenteditable #prompt-textarea`，实现 `replaceRange($queryRange, markdownText)`（优先走兼容性更好的 insertText/execCommand 路径，再做 DOM fallback）。
3. 插入后光标停在插入文本后，且保留输入框其余文本不变。
4. 不触发发送消息动作；仅处理插入（并确保宿主 UI 感知到输入变化）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/item-mention-chatgpt-adapter.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: ChatGPT 文本替换语义正确且不误发送。

**Step 3: 原子提交**
- `feat: task8 - 实现chatgpt输入框mention替换插入适配器`

---

<a id="p2-t4"></a>
## P2-T4 接入 content 入口并完成 ChatGPT 端到端链路

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`
- Modify: `webclipper/src/entrypoints/content.ts`
- Modify: `webclipper/src/services/bootstrap/content-controller.ts`
- Add: `webclipper/src/services/integrations/item-mention/client.ts`

**Step 1: 实现**
1. 将 mention controller 挂接到现有 content controller 生命周期中（随 `inpage_display_mode` 的 start/stop 同步），P2 阶段仅在 ChatGPT（`chatgpt.com`）路径激活（其余 host 静默跳过）。
2. controller 打通流程：触发 -> 调 background 搜索 -> 渲染候选 -> 选中 -> 调 background 构建文本 -> 插入。
3. 处理查询竞态：仅消费最后一次输入对应响应，丢弃过期结果。
4. 无结果、请求错误时显示轻量 empty/error 状态，不破坏输入焦点与既有 inpage 功能。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/content-controller-inpage-combo.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: ChatGPT 端到端链路可运行且不影响既有 inpage 功能。

**Step 3: 原子提交**
- `feat: task9 - 打通chatgpt端到端mention插入链路`

---

<a id="p2-t5"></a>
## P2-T5 补齐触发/键盘/替换语义测试（含 Esc 保留文本）

**Files:**
- Add: `webclipper/tests/smoke/item-mention-chatgpt.test.ts`
- Modify: `webclipper/tests/unit/item-mention-session.test.ts`
- Modify: `webclipper/tests/unit/item-mention-chatgpt-adapter.test.ts`

**Step 1: 实现**
1. 覆盖 `$` 立即触发、输入过滤、候选排序。
2. 覆盖键盘语义：`↑/↓` 导航，`Tab/Enter` 插入，`Esc` 关闭并保留文本。
3. 覆盖替换片段语义：仅替换 `$query` 区间，不改前后文本。
4. 覆盖 no-truncation：插入内容与构建文本一致。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/item-mention-chatgpt.test.ts tests/unit/item-mention-session.test.ts tests/unit/item-mention-chatgpt-adapter.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: P2 行为级测试通过。

**Step 3: 原子提交**
- `test: task10 - 补齐chatgpt mention触发与键盘语义测试`

---

## 阶段审计

- Audit file: `audit-p2.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环。
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
