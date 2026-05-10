# 想法：把 `$ mention` 扩展到更多 AI Chat 平台

## 背景

我们现在已经在以下平台跑通了 `$ mention`（搜索已保存条目并以 Markdown 插入输入框）：

- ChatGPT
- Notion AI

下一步希望把相同的 `$ mention` 交互扩展到其它我们已支持“采集/保存 AI 对话”的平台（见 `SUPPORTED_AI_CHAT_SITES`），让用户在不同 AI 平台聊天时都能引用自己保存过的条目。

## 目标

- 在支持的 AI Chat 站点里，输入 `$` 会在当前输入框附近弹出一个小型候选窗口。
- 在 `$` 后继续输入会过滤候选（带 debounce）。
- 键盘交互：
  - `ArrowUp/ArrowDown` 上下移动高亮
  - `Tab` / `Enter` 选中当前高亮并插入 Markdown（必须阻止站点自身的 “发送消息” 行为）
  - `Escape` 关闭候选框，且不删除用户已输入的文本
- 鼠标交互：
  - 点击候选项插入 Markdown
- 插入内容必须复用现有“复制为 Markdown”的单一真源（不得自己拼新格式）。
- 不做截断：插入完整 Markdown payload。
- 遵守 Settings 总开关：关闭时应完全不生效（不挂监听、不渲染 UI、不拦截按键）。
- 保守触发：只在支持的 AI chat host 上启用，不能劫持其它网站普通输入框。

## 非目标

- 不做“按站点细粒度开关矩阵”（只保留全局开关）。
- 不支持任意网站 textarea/contenteditable 的 `$ mention`（仅 AI chat 平台）。
- 不重写候选 UI（只扩展 editor adapter 与 host 路由）。

## 关键决策与约束

- 平台差异通过 `EditorAdapter` 适配（detect/range/replace/focus）。
- 必须用 `$web-access`（CDP）对每个站点做真实页面验收，确认：
  - composer 的真实类型（`textarea` / `contenteditable` / shadow DOM / iframe）
  - 最小稳定 selector/信号（避免误匹配站点其它输入框）
  - `Tab/Enter` 插入不会触发“发送消息”
- 每个平台至少有 1 个自动化测试覆盖 “打开 + 插入”（smoke 或 unit）。

## 范围内目标平台

来自 `SUPPORTED_AI_CHAT_SITES`，排除已支持的 ChatGPT 与 Notion AI：

- Claude（`claude.ai`）
- Gemini（`gemini.google.com`）
- Google AI Studio（`aistudio.google.com`、`makersuite.google.com`）
- DeepSeek（`chat.deepseek.com`）
- Kimi（`kimi.moonshot.cn`、`kimi.com`）
- 豆包（`doubao.com`）
- 元宝（`yuanbao.tencent.com`）
- Poe（`poe.com`）
- z.ai（`chat.z.ai`）

## 验收标准（Checklist）

- 每个目标站点：
  - 在 composer 中输入 `$` 能打开候选窗口
  - 输入 `$xxx` 能过滤候选
  - `ArrowUp/ArrowDown` 能移动高亮
  - `Tab` 和 `Enter` 能把 `$...` 这段替换成 Markdown，并保持输入框聚焦
  - `Escape` 能关闭候选，且保留用户输入内容不变
- Settings 的提示文案不再误导（不能只写 ChatGPT/Notion）。
- `npm --prefix webclipper run gate` 全绿。

## 重要例外（必须显式记录）

若某些站点的 composer 位于：

- **跨域 iframe**（content script 无法访问）
- **closed shadow root**（无法 query 到真实可编辑节点）

则该站点可能在技术上无法做到“稳定插入”。遇到这种情况必须：

- 在 web-access 调研记录中写明证据与结论
- 保持该 host 上 `$ mention` 不激活（避免半成品拦截按键/误弹窗）
- 计划里将该站点标记为 “暂不支持（原因：xxx）”，而不是硬拖到最后才发现做不了
