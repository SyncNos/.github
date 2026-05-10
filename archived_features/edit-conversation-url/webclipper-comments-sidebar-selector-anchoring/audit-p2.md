# Audit P2 - webclipper-comments-sidebar-selector-anchoring

## Checklist

- [ ] 动态加载/内部滚动容器下定位成功率是否可接受
- [ ] app 侧是否存在误匹配（sidebar 内/重复文本）
- [ ] 失败提示是否对齐（inpage/app）
- [ ] hostile CSS 回归是否通过（不被页面样式污染、点击区域不出问题）

## Manual Regression (Draft)

### Common

- Root comment（带引用）点击：应 smooth 滚动到引用所在位置；不需要高亮。
- Reply 点击：不触发定位。
- 无 locator 的历史根评论点击：不滚动，显示轻提示“无法定位”（自动消失、不抢焦点）。
- 窄屏（`md` 以下）：不要求定位（通常无右侧 sidebar）。

### Inpage (content script sidebar)

**ChatGPT（内部滚动容器）**
- 打开任意长对话页面，找到一段在“可滚动内容区”内的文本（避免 header/footer）。
- 鼠标选中一段唯一短句 -> 打开/聚焦 inpage comments sidebar -> 发送根评论（带引用）。
- 点击该根评论的 quote 或 comment body：
  - 预期：滚动发生在对话内容区（而不是页面整体），并能把目标文本滚到视野中间附近。
  - 若第一次失败：允许一次轻量重试后成功；最终失败则提示“无法定位”。

**Notion（复杂 DOM）**
- 打开一篇较长 Notion 页面，选中正文中的一段唯一文本 -> 发送根评论（带引用）。
- 点击定位：应滚到对应文本附近；失败提示不影响输入/回复。

**Medium（长文）**
- 打开长文，选中正文某段 -> 发送根评论（带引用）-> 点击定位：应滚到对应段落附近。

### App (SyncNos WebClipper app)

- 打开 app 宽屏布局（右侧能出现 comments sidebar）。
- 在详情页消息区选中一段文本 -> 点击评论按钮 -> 发送根评论（带引用）。
- 点击该根评论定位：
  - 预期：只滚动消息内容区（不滚到 sidebar 内的相同文本），且 smooth。

## Notes

- 自动化验证通过：
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper test`
  - `npm --prefix webclipper run build`
- 以上 Checklist 的站点级手动回归：待执行（按本文件步骤逐项勾选）。
- 飞书/类似动态加载站点若仍失败：优先确认是否为“内部滚动容器/虚拟列表”导致目标文本未渲染；已追加一次滚动容器识别 + 重试（commit `b7278c1b`），建议在该站点复测 A/B 任务链路。
