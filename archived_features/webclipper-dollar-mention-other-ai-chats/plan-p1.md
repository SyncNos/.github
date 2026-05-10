# 计划 P1：基础建设

## 目标

把“新增一个站点的 `$ mention` 支持”变成低风险、低成本的重复动作：

- 用 registry 标准化 host -> adapter 路由
- 提供可复用的 textarea/contenteditable 基础适配工具
- 修正 Settings 的提示文案（不再只写 ChatGPT/Notion）
- 建立可复制的 smoke test 模板，避免每个站点都手写一大坨

## P1-T1：web-access 站点调研：整理每个平台的 composer 类型/selector/按键语义与可达性

### 为什么

现在 plan 里默认“全部站点都能做成”，但现实里经常会遇到：

- composer 在跨域 iframe（content script 访问不到）
- composer 在 closed shadow root（query 不到真实 editable）
- `Enter` 默认发送消息，`Tab` 默认切换焦点，必须严谨拦截并且只在候选框打开时拦截

如果不先调研，会在实现阶段频繁返工。

### 产出物（必须写入证据链）

在本 feature 目录下新增并维护一个调研记录文件（本地证据，不入 git）：

- `.github/features/webclipper-dollar-mention-other-ai-chats/.audit/site-survey.md`

每个站点至少记录：

- URL（落到哪个页面才是聊天 composer）
- composer 类型：`textarea` / `contenteditable` / shadow DOM / iframe
- 最小稳定 selector（优先 activeElement + 语义属性，其次 querySelector）
- 插入策略（execCommand / Range / setRangeText 等）
- 键盘语义风险：`Enter` 是否发送，`Tab` 是否夺焦点，是否需要 `stopImmediatePropagation`
- 是否可稳定支持：`yes/no`，若 `no` 必须写原因与证据

### 验证

- 调研记录写完即可（无代码变更）。

### 提交

- 无需提交（这是纯本地证据链文件）。

## P1-T2：重构 item-mention adapter 路由为 registry（可按 host 扩展）

### 为什么

当前 controller 里硬编码了 ChatGPT/Notion 的 host 判断。继续往里加会越来越乱，且难以做到“某站点不可支持时完全不激活”。

### 范围 / 文件

- `webclipper/src/services/integrations/item-mention/content/mention-controller.ts`
- New:
  - (Optional) `webclipper/src/services/integrations/item-mention/content/editor-registry.ts`
- Tests:
  - `webclipper/tests/unit/**` (adapter-level tests)
  - `webclipper/tests/smoke/**` (controller-level tests when URL host matches)

### 实施步骤

1. 引入 registry：`{ id, matchHost, detectContext?, adapter }[]`。
2. 把 host/context 的选择逻辑从 controller 中抽出来，统一走 registry。
3. 保持现有 ChatGPT/Notion 行为不变（不允许回归）。
4. 允许 registry 显式返回 “不支持/不激活”（用于 P1 调研中判定不可支持的站点）。

### 验收

- 现有 ChatGPT/Notion 相关测试保持全绿。
- `mention-controller` 在非支持 host 上不会启动（`start()` 返回 `null` 或不挂监听）。

### 验证

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -t item\\ mention`

### 提交

- `refactor: P1-T2 - 抽象 item mention adapter registry`

## P1-T3：补齐 textarea 通用适配工具（用于新平台复用替换/光标/事件）

### 为什么

很多站点最终落到 `textarea`（或表现像 textarea）。我们希望把“读光标/替换区间/恢复光标/触发 input”做成可复用工具，避免每站重复写 bug。

### 范围 / 文件

- New:
  - `webclipper/src/services/integrations/item-mention/content/editor-textarea.ts`
- Tests:
  - `webclipper/tests/unit/**`（至少覆盖 range/replace 的边界）

### 实施步骤

1. 实现：
   - `detectActiveTextarea()`（优先 activeElement，必要时允许站点传入 selector 兜底）
   - `getSelectionRange()`（`selectionStart/End`）
   - `replaceRange()`（优先 `setRangeText`，其次手动拼接 + 还原 selection）
   - 统一派发 bubbled `input` 事件
2. 明确 contract：只有在 session open 且按键被我们消费时才 `preventDefault/stopPropagation`。

### 验收

- Unit tests demonstrate:
  - range 读写正确
  - 替换后光标在正确位置
  - 输入事件被触发（供站点更新 UI）

### 验证

- `npm --prefix webclipper run test -t textarea`

### 提交

- `feat: P1-T3 - 新增 textarea 通用 editor adapter`

## P1-T4：补齐 contenteditable 通用适配工具（用于非 ChatGPT/Notion 的富文本输入框）

### 为什么

部分站点会使用 `contenteditable`（但不是 ChatGPT 的 ProseMirror）。我们需要一个“通用、保守”的 contenteditable 适配工具作为基座，站点 adapter 只负责 detect。

### 范围 / 文件

- New:
  - `webclipper/src/services/integrations/item-mention/content/editor-contenteditable-generic.ts`
- Tests:
  - `webclipper/tests/unit/**`

### 实施步骤

1. 复用现有 ChatGPT/Notion 的 offset 计算思路，但实现为通用版：
   - 在 root 内计算 selection offsets
   - `replaceRange` 优先尝试 `execCommand('insertText')`（更接近“像用户输入”），失败再 fallback
   - 统一派发 bubbled `input`
2. 明确：ChatGPT 仍走自己的 ProseMirror 适配器，不能被 generic 覆盖。

### 验收

- 单测证明：在简单 contenteditable DOM 下，能正确替换并保持 focus。

### 验证

- `npm --prefix webclipper run test -t contenteditable`

### 提交

- `feat: P1-T4 - 新增 contenteditable 通用 editor adapter`

## P1-T5：更新 Settings 的 `$ mention` i18n 提示文案（覆盖更多平台）

### 为什么

当前 hint 文案会误导用户以为只支持 ChatGPT/Notion。扩展后必须改成“支持的 AI chat 站点”。

### 范围 / 文件

- `webclipper/src/ui/i18n/locales/en.ts`
- `webclipper/src/ui/i18n/locales/zh.ts`

### 实施步骤

1. `aiChatDollarMentionHint` 改为 “supported AI chat sites / 支持的 AI 聊天站点”。
2. 文案不要枚举站点列表（避免后续维护成本）。

### 验收

- 中英文 hint 都不再绑定 ChatGPT/Notion。

### 验证

- `npm --prefix webclipper run compile`

### 提交

- `i18n: P1-T5 - 更新 $ mention 提示文案（多站点）`

## P1-T6：为新增平台适配器建立 smoke test 模板与复用 helper

### 为什么

每个站点都要 “打开 + 插入” 的最小测试，但不能无限复制粘贴现有 smoke test，否则维护成本爆炸。

### 范围 / 文件

- 新增 helper（示例路径，按实际调整）：
  - `webclipper/tests/helpers/item-mention-test-helpers.ts`
- 允许重构：
  - `webclipper/tests/smoke/item-mention-chatgpt.test.ts`
  - `webclipper/tests/smoke/item-mention-notionai.test.ts`

### 验收

- 新站点 smoke test 只需要：
  - jsdom url host
  - composer fixture html + “可见” stub
  - 断言 search 被调用、insert 被写回 editor

### 验证

- `npm --prefix webclipper run test -t item\\ mention`

### 提交

- `test: P1-T6 - 抽取 item mention 测试 helper`

## Audit P1

完成 P1 后，更新 `audit-p1.md`：

- API/设计正确性
- 测试覆盖缺口
- contenteditable/textarea 编辑器之间的潜在回归风险
