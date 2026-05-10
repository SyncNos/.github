# 背景 / 触发

- 触发：用户反馈 **AI chats auto save** 在跨设备场景下会“只保存网页端新输入输出”，而不会把打开页面时已存在的历史消息自动补齐到本地。
  - 典型复现：手机端先聊一段 -> 网页端打开同一对话 -> 开启 auto save + 手动开启“Fetch AI chat”能力 -> 本地只出现网页端新增消息，打开时页面上已有的历史缺失。
- 现状根因（已确认）：autosave 增量引擎在首次 capture 时只建立 baseline，不写入；仅对非常短的窗口（`<=6`）做 seed；并且只在最后 200 条窗口中工作（`MAX_WINDOW_MESSAGES=200`）。
- 目标：在不引入“重进对话重复追加爆炸”的前提下，让 autosave 能在打开对话时自动补齐缺失历史（至少覆盖最近 200 条窗口内的缺口），从根本上解决跨设备缺口问题。

## 核心需求（原始需求精炼）

1. 当用户开启 `ai_chat_auto_save_enabled` 且在支持的 AI chat 站点打开某个会话时：
   - 插件应自动把“页面当前可抓取到的历史（窗口内）但本地缺失的消息”补齐到本地 IndexedDB。
   - 补齐需要 append-only：不删除、不替换本地已有消息。
2. 补齐逻辑需要稳健：
   - 如果无法找到可靠 overlap 锚点（本地 tail 与页面 window 完全对不上），必须采取安全策略：不写入，仅 console warn；同时后续增量保存仍应继续可用。
   - 但如果 **本地完全为空**（conversation 尚未创建或无任何消息），应允许把 page window（<=200）作为“首次落盘”直接 append-only 写入一次，否则无法解决“首次打开即缺失历史”的根因场景。
   - 支持“窗口变化触发的有限重试”：当页面 window 发生变化（例如用户滚动导致加载更多内容）时，可以再尝试 backfill；有节流与尝试上限。
3. 现状窗口即可：
   - v1 复用现有 autosave 的 `MAX_WINDOW_MESSAGES=200`，只保证在最近 200 条窗口范围内的自动补齐。
4. 用户体验：
   - backfill 过程无需 UI 提示；日志输出到页面 console 即可（info/warn）。

## 默认值与兼容策略

- 新安装默认值：不新增设置键；沿用 `ai_chat_auto_save_enabled`（默认 `true`）与现有支持站点列表（默认排除 `googleaistudio`）。
- 已有用户升级：
  - 行为会改变：开启 auto save 后，打开对话会尝试自动补齐缺失历史（窗口内）。
  - 保持 append-only：不会删除本地已有消息；手动 “Save / Fetch AI chat” 仍是全量 snapshot 的最终兜底方式。
- 兼容性边界：
  - 若站点/collector 只暴露尾部窗口或消息结构不稳定导致 overlap 无法建立，则 backfill 会安全跳过并提示用户手动保存。

## 非目标（明确不做什么）

- 不新增 UI 开关/opt-out（v1 直接随 `ai_chat_auto_save_enabled` 生效）。
- 不改国际化文案（i18n）。
- 不把 autosave 改回 destructive snapshot；不做自动删除/对齐清理。
- 不承诺“全量历史补齐”（v1 仅最近 200 条窗口）；不做跨页面网络请求去抓取更多历史。

## 验收标准（可检查）

1. **跨设备缺口可自动补齐（窗口内）**
   - 准备：手机端在某会话中新增若干条消息；桌面端本地尚未包含这些消息。
   - 操作：桌面端打开该会话页面，保持 `ai_chat_auto_save_enabled=true`。
   - 结果：在无需继续输入的情况下，本地会话消息会自动增加（补齐到包含手机端新增的消息，至少在页面可抓取的最后 200 条窗口内），并在 console 输出一次 backfill info。
2. **安全策略 A 生效**
   - 当本地 tail 与页面 window 无法找到可靠 overlap 时：不发生写入（可通过断言未触发 `syncConversationMessages` 写入或本地消息数不变），console 输出 warn；之后在网页中继续对话产生新消息时，增量保存仍能把新消息 append 到本地。
3. **有限重试**
   - 在首次 backfill 失败（无 overlap）后，用户滚动/页面加载更多消息导致 window 变化时，会在节流/上限内再次尝试；成功一次后停止重试。
4. **构建验证**
   - `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。
