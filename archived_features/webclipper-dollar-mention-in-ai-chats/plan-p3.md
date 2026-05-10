# 计划 P3 - webclipper-dollar-mention-in-ai-chats

**目标：** 完成 NotionAI 适配与跨站点收口，补齐边界处理、文档同步、Settings 开关与最终验证，并为后续扩展更多 AI chats 预留扩展位。

**非目标：**
- 本阶段不做 Claude（你当前未登录 Claude，可先跳过）。
- 其他站点扩展（Gemini/DeepSeek/Perplexity 等）作为本计划的后续 task（见 `P3-T7+`）。

**实现思路：**
- 先实现 NotionAI `contenteditable` 插入适配器，保证与 ChatGPT 共享同一 mention 控制器协议。
- 再收敛 adapter 抽象与边界行为（IME、失焦、异步乱序、runtime invalidation）。
- 最后同步文档并执行完整验证链，形成可交付闭环。

**验收标准：**
- NotionAI 中 `$` 流程与 ChatGPT 行为一致（触发、过滤、导航、插入、关闭）。
- 触发片段替换在 `contenteditable` 场景稳定，不破坏前后文本。
- 文档入口与实现一致，验证链可复现。
- 不残留多余站点分叉：跨站点差异只允许存在于 editor adapter 内，其余状态机/控制器必须保持单一真源。

---

<a id="p3-t1"></a>
## P3-T1 实现 NotionAI contenteditable 光标替换适配器

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/content/editor-notionai.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/editor-adapter.ts`
- Add: `webclipper/tests/unit/item-mention-notionai-adapter.test.ts`

**Step 1: 实现**
1. 在 NotionAI 场景识别可编辑根节点（`contenteditable`）。
2. 实现 range 读取与替换：精确替换 `$query` 区间，保留其它内容。
3. 插入后恢复焦点与光标，保持用户后续输入连续。
4. 适配器不耦合候选窗 UI，仅负责 editor 操作。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/item-mention-notionai-adapter.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: NotionAI range 替换稳定且可重复执行。

**Step 3: 原子提交**
- `feat: task11 - 实现notionai contenteditable mention适配器`

---

<a id="p3-t2"></a>
## P3-T2 收敛跨站点 editor adapter 接口与焦点恢复策略

**Files:**
- Modify: `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/editor-adapter.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/editor-chatgpt.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/editor-notionai.ts`

**Step 1: 实现**
1. 统一 adapter 选择策略：ChatGPT / NotionAI 自动选择当前激活 editor。
2. 插入前后焦点行为统一：失败不吞事件，成功后定位到插入末尾。
3. 保持 `Tab/Enter` 仅在候选窗打开时拦截，候选关闭后恢复宿主默认行为。
4. `Esc` 行为统一为“关闭候选窗，不改文本”。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Expected: 两类 adapter 在同一控制器下行为一致。

**Step 3: 原子提交**
- `refactor: task12 - 收敛跨站点mention适配接口与焦点策略`

---

<a id="p3-t3"></a>
## P3-T3 补齐竞态与边界处理（IME/失焦/异步响应乱序）

**Files:**
- Modify: `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/mention-session.ts`
- Add: `webclipper/tests/smoke/item-mention-notionai.test.ts`
- Modify: `webclipper/tests/smoke/item-mention-chatgpt.test.ts`

**Step 1: 实现**
1. 增加 composition guard：IME 组合输入期间不误触发确认。
2. 查询响应乱序时只接受最新 request，过期响应直接丢弃。
3. 编辑器失焦/DOM 替换时自动关闭候选窗，避免悬挂 UI。
4. runtime invalidation 时安全降级并清理监听器。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/item-mention-chatgpt.test.ts tests/smoke/item-mention-notionai.test.ts`
- Run: `npm --prefix webclipper run compile`
- Expected: 边界场景不崩溃、不误插入、不残留悬挂浮层。

**Step 3: 原子提交**
- `fix: task13 - 修复mention竞态与边界场景稳定性`

---

<a id="p3-t4"></a>
## P3-T4 同步 WebClipper 文档真源（AGENTS/deepwiki/storage）

**Files:**
- Modify: `webclipper/AGENTS.md`
- Modify: `.github/deepwiki/modules/webclipper.md`
- Modify: `.github/deepwiki/storage.md`
- Modify: `.github/deepwiki/business-context.md`（仅补导航与行为说明）

**Step 1: 实现**
1. 更新“其他 AI chats 中 `$` mention”功能边界与当前支持站点（ChatGPT + NotionAI）。
2. 更新触发/过滤/键盘语义说明（`$` 触发、`Tab/Enter` 插入、`Esc` 关闭保留文本）。
3. 明确插入文本来自现有 copy markdown 真源，且当前不截断。

**Step 2: 验证**
- Run: `rg -n "\$ mention|ChatGPT|NotionAI|Tab|Enter|Esc|markdown|candidate" webclipper/AGENTS.md .github/deepwiki/modules/webclipper.md .github/deepwiki/storage.md .github/deepwiki/business-context.md`
- Expected: 文档可检索到与实现一致的行为约束。

**Step 3: 原子提交**
- `docs: task14 - 同步dollar mention行为与边界文档`

---

<a id="p3-t5"></a>
## P3-T5 完成最终验证链与 phase 审计收口

**Files:**
- Modify: `.github/features/webclipper-dollar-mention-in-ai-chats/audit-p3.md`
- Modify: `.github/features/webclipper-dollar-mention-in-ai-chats/todo.toml`（执行流回写）

**Step 1: 实现**
1. 执行完整验证链：`compile -> test -> build`。
2. 若存在历史阻塞用例，记录失败项并明确“是否由本 feature 引入”。
3. 在审计文件记录 residual risks 与后续扩展建议（新增站点 adapter）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Run: `npm --prefix webclipper run build`
- Expected: compile/build 通过；test 结果可归因。

**Step 3: 原子提交**
- `chore: task15 - 完成dollar mention最终验证与审计收口`

---

<a id="p3-t6"></a>
## P3-T6 Settings 中支持开关 `$` mention

**目标：** 允许用户在 Settings 面板中显式开启/关闭 `$` mention（默认开启），并且切换后能动态生效。

**Files:**
- Modify: `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`
- Modify: `webclipper/src/ui/settings/SettingsScene.tsx`
- Modify: `webclipper/src/ui/settings/sections/InpageSection.tsx`
- Modify: `webclipper/src/services/bootstrap/content-controller.ts`
- Modify: `webclipper/src/ui/i18n/locales/en.ts`
- Modify: `webclipper/src/ui/i18n/locales/zh.ts`
- Add: `webclipper/tests/smoke/content-controller-item-mention-setting.test.ts`

**Step 1: 实现**
1. 新增存储开关：`ai_chat_dollar_mention_enabled`（默认 true）。
2. Settings `General -> Inpage` 中新增独立卡片（不放在 Auto-save 卡片内），写入 storage。
3. Content controller 启动时读取该开关，决定是否启动 mention controller；同时监听 `storage.onChanged` 动态启停。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/smoke/content-controller-item-mention-setting.test.ts`
- Run: `npm --prefix webclipper run gate`

**Step 3: 原子提交**
- `feat: add Settings toggle for $ mention`

---

<a id="p3-t7"></a>
## P3-T7（后续）将 `$` mention 的站点门控与 AI chat 站点清单对齐

**目标：** 避免 “Settings 显示支持站点” 与 “mention-controller 实际启用站点” 分叉，保持单一真源；后续新增站点只需要改一处。

**Files:**
- Modify: `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`
- Modify: `webclipper/src/collectors/ai-chat-sites.ts`（如需补充站点元信息）
- Add: `webclipper/src/services/integrations/item-mention/content/mention-sites.ts`
- Add: `webclipper/tests/unit/item-mention-sites.test.ts`

**Step 1: 实现**
1. 引入 `mention-sites.ts`：基于 `SUPPORTED_AI_CHAT_SITES` 生成可复用的 `isMentionSupportedHost(hostname)` / `listMentionSupportedHosts()`。
2. `mention-controller.ts` 不再硬编码正则判断 ChatGPT/Notion，仅使用 `mention-sites.ts` 判定是否启用与选择 adapter。
3. 保持 adapter 仍是唯一的站点差异承载点（站点门控只决定“是否启用”和“选哪个 adapter”，不分叉状态机）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run test -- tests/unit/item-mention-sites.test.ts`
- Run: `npm --prefix webclipper run compile`

**Step 3: 原子提交**
- `refactor: task16 - 对齐mention站点门控与AI chat站点清单`

---

<a id="p3-t8"></a>
## P3-T8（后续）新增 Gemini 编辑器适配器（Claude 跳过）

**目标：** 为 `gemini.google.com` 实现 editor adapter，并在真实页面完成端到端冒烟验证（无需依赖 Claude 登录）。

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/content/editor-gemini.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`
- Add: `webclipper/tests/smoke/item-mention-gemini.test.ts`（如可稳定跑）

**Step 1: 实现**
1. 识别 Gemini 输入框（通常为 `contenteditable` 或带 `role="textbox"` 的可编辑节点），实现 selection/range 替换与焦点恢复。
2. 插入后确保宿主能感知输入变化（如需要触发 `input` 事件）。
3. 若站点结构不稳定，将 “selector + range 替换逻辑” 尽量收敛在 adapter 内，不污染 controller。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- 手工冒烟（web-access）：打开 Gemini，输入 `$`，验证弹窗/过滤/Tab/Enter 插入链路。

**Step 3: 原子提交**
- `feat: task17 - 支持Gemini的dollar mention插入`

---

<a id="p3-t9"></a>
## P3-T9（后续）新增 DeepSeek 编辑器适配器

**目标：** 为 DeepSeek Chat 增加 adapter，保持 `$` 行为与 ChatGPT/Notion 一致。

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/content/editor-deepseek.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`

**Step 1: 实现**
1. 识别 DeepSeek 输入框（优先 contenteditable；若是 textarea 则走 textarea 路径）。
2. 实现替换 `$query` 范围并插入 markdown；插入后恢复焦点与光标。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- 手工冒烟（web-access）：打开 DeepSeek，验证 `$` 触发与插入闭环。

**Step 3: 原子提交**
- `feat: task18 - 支持DeepSeek的dollar mention插入`

---

<a id="p3-t10"></a>
## P3-T10（后续）新增 Kimi 编辑器适配器

**目标：** 为 Kimi（`kimi.moonshot.cn` / `kimi.com`）增加 adapter，保持键盘语义一致（`Tab/Enter` 仅在候选窗打开时拦截）。

**Files:**
- Add: `webclipper/src/services/integrations/item-mention/content/editor-kimi.ts`
- Modify: `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`

**Step 1: 实现**
1. 识别 Kimi 输入框并实现 range 替换插入。
2. 若站点对 `Enter` 有强绑定（例如发送），确保只在候选窗打开时拦截；候选窗关闭时恢复默认行为。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- 手工冒烟（web-access）：打开 Kimi，验证 `$` 触发与插入闭环。

**Step 3: 原子提交**
- `feat: task19 - 支持Kimi的dollar mention插入`

---

<a id="p3-t11"></a>
## P3-T11（后续）清理旧代码与最终验证链收口（含 lint/format/gate）

**目标：** 在扩展站点前后都保持代码库干净：无旧 selector/旧逻辑残留；并确保完整验证链可复现通过。

**Files:**
- Modify: `webclipper/src/services/integrations/item-mention/**`（仅在确认无用时删除/收敛）
- Modify: `webclipper/tests/**`（如因 adapter 调整需要补/修测试）

**Step 1: 清理**
1. `rg` 检查是否仍存在旧的 ChatGPT textarea 假设、旧 selector、旧 message type 名称；确认无引用后删除或重命名为单一真源。
2. 检查 item-mention 目录是否有未被引用的文件/导出（避免未来维护误导）。

**Step 2: 验证**
- Run: `npm --prefix webclipper run compile`
- Run: `npm --prefix webclipper run test`
- Run: `npm --prefix webclipper run lint`
- Run: `npm --prefix webclipper run format`
- Run: `npm --prefix webclipper run gate`
- Expected: 全部通过；若有历史已知阻塞，必须在本 task 的 note 中明确归因。

**Step 3: 原子提交**
- `chore: task20 - 清理dollar mention旧代码并完成验证链收口`

---

## 阶段审计

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环。
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令
