# Discourse topic URL 归一化 + OP DOM 抓取

## 背景 / 触发

- 当前 WebClipper 在 Discourse 站点（如 linux.do）上，会把 `/t/slug/topic-id/N` 视为不同文章 URL。
- 用户在同一 topic 的不同楼层（`/1`、`/20`）切换时，会出现：
  - article conversation 重复
  - inpage 评论侧栏会话被重置
  - 非首楼触发抓取时，可能抓到回复内容而不是楼主 OP
- 本次已明确关键策略（pivot）：
  - 只用 DOM，不发额外请求
  - 必要时允许跳转到 `/1` 获取 OP
  - 获取后停留在 `/1`，不自动跳回原楼层
  - `/1` 超时找不到 OP 时严格失败，不降级抓当前楼层

## 核心需求（原始需求精炼）

1. 对 Discourse topic URL 做归一化：`/t/slug/topic-id/N` 统一映射到 `/t/slug/topic-id`（canonical）。
2. 文章抓取始终只保留 OP（首帖），不抓回复作为文章正文。
3. 评论侧栏在同一 topic 的不同楼层 URL 间保持同一会话，不因 `/N` 变化重置。
4. 用户仍可对页面上任意回复执行“选中引用 + 评论”互动。
5. 方案适用于通用 Discourse 站点，不绑定单一域名。

## 默认值与兼容策略

- 默认启用 Discourse topic canonical 规则（对非 Discourse URL 无影响）。
- 升级后对历史数据采用“运行时兼容 + 惰性收敛”：
  - 会话与评论查询优先使用 canonical 规则命中
  - 不强制首启全量迁移，避免阻塞用户路径
- 抓取策略默认顺序：
  1. 当前页尝试定位 OP DOM
  2. 失败且当前不是 `/1` 时跳转 `/1`
  3. 在超时窗口内等待 DOM 稳定并重试
  4. 仍失败则报错终止

## 非目标（明确不做什么）

- 不引入 Discourse JSON/API 拉取。
- 不实现“跳到 `/1` 后再自动回原楼层”。
- 不做站点专用 hardcode（仅用通用 Discourse 结构特征）。
- 不改变现有“选中引用 + 评论”的交互语义。

## 验收标准（可检查）

- 在 Discourse topic 中，`/t/slug/topic-id` 与 `/t/slug/topic-id/20` 命中同一 canonical URL。
- 同一 topic 不再创建重复 article conversation。
- inpage 评论侧栏在同一 topic 的楼层切换后会话保持稳定（上下文键不变）。
- 从 `/20` 触发抓取时，最终文章正文来自 OP。
- 当无法在规定时间内定位 OP 时，抓取明确失败并给出可理解提示。
