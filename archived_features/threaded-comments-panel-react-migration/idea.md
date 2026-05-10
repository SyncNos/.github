# 背景 / 触发

- 右侧评论侧边栏（以及同源的 inpage comments overlay）当前在用户发送一条 comment / reply 之后，光标会回到顶部的 `Write a comment…` 输入框，导致用户无法顺滑地继续在同一 thread 下连续回复。
- 该行为的根因是 comments panel 的 threads 渲染策略：每次 refresh 都会整块销毁并重建 threads DOM，导致 thread 内的 `Reply…` textarea 节点必然丢失，从而 focus 无法保留。
- 本 feature 选择**破坏性重构**：将 `@ui/comments` 的 threaded comments panel 从手写 DOM 渲染迁移到 React（keyed 渲染），并删除旧实现，根治 focus 丢失问题，同时保持现有交互能力不回退。

# 核心需求（原始需求精炼）

1. 发送 root comment（顶部 `Write a comment…`）成功后：
   - 光标必须自动落在**新建 thread 对应的 `Reply…` 输入框**中。
   - 视图必须滚动到该 `Reply…` 输入框可见（用户不需要手动滚动找光标落点）。
2. 发送 reply（thread 内 `Reply…`）成功后：
   - 光标必须继续停留在**同一 thread 的 `Reply…` 输入框**中（便于连续回复）。
   - 视图保持该 `Reply…` 输入框可见（必要时滚动到可视区域）。
3. 该行为在两个宿主中一致：
   - app 右侧评论侧边栏（`webclipper/src/ui/conversations/ArticleCommentsSection.tsx`）
   - inpage overlay（`webclipper/src/ui/inpage/inpage-comments-panel-shadow.ts`）
4. 保持现有功能不回退（同等能力迁移到 React 实现）：
   - open/close、busy/disabled、quote 区、root composer、thread 列表、reply composer
   - delete 二次确认（按钮内联二次确认，不使用 alert/confirm）
   - sidebar variant 的 locate（点击 quote 定位到页面高亮；失败提示）
   - header 的 Chat with AI 菜单（如启用）
   - comment-level Chat with AI 菜单（如启用）
   - sidebar resize、dockPage（inpage overlay）等现有结构能力保持
   - stopShortcutKeyPropagation（shadow 内阻止站点快捷键干扰输入）

# 默认值与兼容策略

- 默认启用新实现：本次为破坏性重构，迁移完成后不保留 legacy DOM 实现的开关或双轨逻辑。
- 对消息契约与存储：
  - 不改变 `COMMENTS_MESSAGE_TYPES.ADD_ARTICLE_COMMENT` 等消息协议语义；
  - 允许在 UI 内部将 `addRoot` / `onSave` 的返回值升级为结构化结果（例如返回 `createdRootId`），用于精确 focus/scroll。
- 对已存在数据：不做迁移；仅 UI 渲染方式变化。

# 非目标（明确不做什么）

- 不重做 i18n/文案体系；除非迁移导致现有 `t('...')` key 无法复用，否则不新增/调整 i18n 字段。
- 不引入新的评论排序策略或新的数据字段。
- 不在本 feature 中优化性能到极致（如虚拟列表）；仅保证 React keyed 渲染与交互正确性。

# 验收标准（可检查）

- [ ] 在 app 右侧评论侧边栏中：
  - [ ] 发 root comment 成功后，自动滚动并 focus 到新 thread 的 `Reply…`。
  - [ ] 发 reply 成功后，仍 focus 在同 thread 的 `Reply…`。
- [ ] 在 inpage overlay 中同样满足上述两条。
- [ ] refresh（comment add/delete 后的 `setComments` 重渲染）不会导致 reply 输入框丢焦点回到顶部 composer。
- [ ] delete 二次确认、locate、Chat with AI 菜单等现有能力在新实现中可用且不回退。
- [ ] 最小验证通过：`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`。

