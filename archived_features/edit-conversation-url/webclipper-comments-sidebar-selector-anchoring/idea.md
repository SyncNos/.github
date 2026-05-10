# 背景 / 触发

- 我们希望给插件的“右侧评论侧边栏”（包括 inpage 评论侧边栏、以及 app 中的右侧评论侧边栏）的评论条目增加“点击定位”能力。
- 之前讨论过用 “scrollTop 百分比（progress）”定位，但该方案在页面高度变化（懒加载/无限滚动/插入内容）时会漂移。
- 本 feature 决定直接采用成熟的 **Selector Anchoring** 方案：按 W3C Web Annotation Selectors 的思路（TextQuoteSelector/TextPositionSelector）记录选区锚点，并在点击时把 selector 还原为 DOM Range 再滚动定位。

## 核心需求（原始需求精炼）

1. 仅对 **右侧侧边栏（sidebar variant）** 生效：
   - inpage：overlay 右侧 sidebar（dock page 的那个）
   - app：宽屏右侧 comments sidebar
   - 窄屏 inline/embedded 评论区不做定位
2. 点击侧边栏里的 **根评论**（root comment，非 reply）：
   - 若该根评论当初是“带引用/选区”（quoteText 非空）创建的，则触发定位
   - reply 点击不触发定位（方案 2）
3. 定位行为：
   - smooth 滚动（不是瞬间跳转）
   - 不需要高亮
4. 环境边界：
   - 只保证“同环境”定位：inpage 写的 locator 仅用于 inpage；app 写的 locator 仅用于 app
   - 跨环境点击同一条评论不需要定位（给轻提示即可）
5. 失败处理：
   - 无 locator / 还原 Range 失败 / 无法滚动时：给轻提示（轻量 toast/inline 提示）

## 默认值与兼容策略

- 新增字段为可选（optional），历史评论默认没有 locator：
  - 点击时不定位，走轻提示。
- locator 的写入只发生在“根评论 + quoteText 非空”的保存路径中：
  - reply 永不写入 locator
  - quoteText 为空的根评论也不写入 locator（用户明确：不要）
- locator 结构允许后续扩展（例如 refinedBy / 多策略 / 站点特化），但 P1 先做最小闭环。

## 非目标（明确不做什么）

- 不做跨环境定位（inpage <-> app）。
- 不为 embedded/narrow inline 评论区实现定位。
- 不做定位后的高亮/闪烁。
- 不承诺对“页面文本被彻底替换/虚拟列表未渲染”的 100% 成功率；失败时给轻提示即可。

## 验收标准（可检查）

- inpage sidebar：对带 quoteText 的根评论，点击后页面 smooth 滚动到对应文本附近；reply 点击不滚动。
- app 右侧 sidebar：对带 quoteText 的根评论，点击后详情内容区（route-scroll）smooth 滚动到对应文本附近；reply 点击不滚动。
- 对无 locator 的历史评论：点击后不滚动，并展示轻提示。
- 对跨环境 locator：点击后不滚动，并展示轻提示。
- 不改变既有语义：
  - sidebar open-only（open 不负责关闭；关闭只走 header collapse/close 按钮）
  - delete/reply/save 交互不受影响

