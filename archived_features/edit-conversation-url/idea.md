# Edit Conversation URL (Detail Header)

## 背景 / 触发

用户希望对部分捕获到的 URL 进行手动修正（常见是去掉多余后缀/跟踪参数），并且入口必须在 **detail view 的 header**（原本显示 `source · conversationKey` 的那一行）。

当前文章评论（comments sidebar / embedded）按 `canonicalUrl`（http/https + 去掉 `#hash`）作为主 key 存储与查询；如果 URL 被手动修改但评论不迁移，会造成“评论看起来消失”的体验问题。

## 核心需求（Goal）

1. 在 detail header 第二行显示 **URL link 文本**（替换 `source · conversationKey`）。
2. **点击 URL 文本进入编辑态**（而不是打开链接）。
3. 编辑保存后，写回并持久化到该 conversation 的 **原始 `url` 字段**（不修改 `conversationKey`）。
4. 对 `article`：URL 变化时需要将评论从旧 `canonicalUrl` **迁移/合并**到新 `canonicalUrl`，保证评论不丢。
5. 提供一个 `清理参数` 按钮：对输入框当前值调用 `cleanTrackingParamsUrl()`（基于现有 tracking-param-cleaner），回填到输入框但不自动保存。
6. 若新 URL 的 canonical 形式已存在于另一条 `article` conversation：保存前必须提示用户；用户确认后继续，并执行“合并评论到新 canonicalUrl”。

## 非目标（Non-goals）

- 本功能的“手动编辑 URL”流程不主动重写 `conversationKey`（避免引入按 key 更新的复杂度与误更新风险）。
- 不把点击 URL 的默认行为改成“打开链接”；打开链接仍走现有 Open actions（P2 可选提供小按钮）。
- 不做复杂自动重写/跳转跟随（只做最小 canonical 化 + 可选的一键清理参数）。
- 不对全仓库 URL normalize 逻辑做大范围去重重构（仅为本功能新增/复用必要能力）。

## 已知行为 / 风险（需在实现与审计中关注）

1. `UPSERT_CONVERSATION` 的 storage 合并逻辑对 `sourceType` **不会 fallback 到 existing**：如果 UI 侧只发送 `source+conversationKey+url`，会把 `article` 错写成 `chat`。因此更新 URL 时必须携带当前 conversation 的 `sourceType`。
2. `UPSERT_CONVERSATION` 会在未提供 `lastCapturedAt` 时把它更新为 `now`；如果我们只是编辑 URL，通常不希望把列表排序/“最近捕获时间”意外改掉。因此更新 URL 时建议携带当前 conversation 的 `lastCapturedAt` 来保持稳定。
3. `article` 捕获流程会用 URL 生成 `conversationKey`。即使本功能不主动改 `conversationKey`，后续若重新抓取同一 URL，现有 upsert/去重逻辑可能会基于 URL 匹配并更新该条记录的 `conversationKey`（并触发 `sync_mappings` key 迁移）。本 feature 不阻断该行为，但需要在 audit 中确认不会造成数据丢失（尤其是 notion mapping）。

## 兼容与默认策略

- canonical 化规则（保存前）：
  - `trim`
  - 仅允许 `http://` / `https://`
  - 移除 `#hash`（`url.hash = ''`）
- `清理参数`：使用 `cleanTrackingParamsUrl(input)` 作为 best-effort；若清理失败则保持原输入不变并提示错误。

## 验收标准（Acceptance）

- Header 第二行显示当前 conversation 的 URL（无 URL 时显示占位提示）。
- 点击 URL 文本可进入编辑态；`Enter` 保存，`Esc` 取消。
- 保存成功后：
  - `conversation.url` 更新并在 UI 立刻反映
  - `article` 评论在新 URL 下可见（旧 URL 下不再分裂出另一套评论线程）
- 当新 URL 与另一条 `article` 的 canonicalUrl 冲突：
  - 必须出现明确提示（confirm）
  - 用户确认后评论合并；取消则不产生任何落库修改
