# 背景 / 触发

- 当前 `app.html` 存在宽屏三栏与窄屏单栏两套交互；popup 基本等同窄屏两步流（`list -> detail`）。
- 文章评论在窄屏仍以内嵌 detail 顶部的方式展示，导致与 app 宽屏右侧 comments sidebar 语义不一致。
- 现有窄屏 header 由 `PopupShell`/`AppShell` 外层 `DetailNavigationHeader` 承载，而 `ConversationDetailPane` 本身也具备 header 能力，存在双轨风险。

本 feature 目标是：复用现有 comments sidebar 能力，把评论并入窄屏三步流（`list -> detail -> comments`），并在 app 中新增 medium 档位策略（默认 comments 关闭）。

## 核心需求（原始需求精炼）

1. **复用 comments 面板能力，不重写评论业务逻辑**  
   必须复用 `ArticleCommentsSection` sidebar 模式与现有 session/controller 能力，而不是新造一套 comments UI/状态机。

2. **窄屏统一三步流**  
   popup 与 app 窄屏都改为 `list -> detail -> comments`。  
   其中 `comments` 的入口来自 detail header 的 comments 按钮；close/collapse 必须返回 detail。

3. **移除窄屏内嵌 comments**  
   `ConversationDetailPane` 的窄屏内嵌 comments（`md:tw-hidden` embedded 版本）必须下线，避免同屏双 comments 入口。  
   窄屏的 comments 入口只保留在详情顶部的 header action 区，不能放到正文内容里。

4. **app 增加 medium 布局策略（仅 app）**  
   medium 下：comments 关闭时两栏，打开后三栏。  
   medium 默认关闭 comments，不继承 wide 的“已打开”状态。

5. **wide 行为不回归**  
   app wide 继续沿用当前 comments 持久化键与自动打开逻辑；不能被 medium 状态污染。

6. **测试基线同步更新**  
   现有 smoke（尤其 popup/app 窄屏 header 与 Escape 路由）会因结构调整受影响，计划内必须显式覆盖并修正。

## 默认值与兼容策略

- 档位默认按 Tailwind 断点：
  - `narrow < 768 (md)`
  - `medium = 768..1279`
  - `wide >= 1280 (xl)`
- popup 仅实现窄屏三步流，不引入 medium 语义。
- wide 继续使用 `webclipper_app_comments_sidebar_collapsed`。
- medium 不读写 wide 持久化键，刷新后始终默认关闭。
- comments 数据层保持本地注释语义，不扩大到 Notion/Obsidian 同步。

## 非目标（明确不做什么）

- 不重做 threaded comments panel UI 结构、样式与交互模型。
- 不新增 settings 开关。
- 不改变 detail header action 的 provider 语义（open/tools 分槽不改）。
- 不引入新的 comments 存储 schema。

## 验收标准（可检查）

- popup：
  - 窄屏可走 `list -> detail -> comments`，且 comments 入口位于详情顶部 header action 区，comments close 后回 detail。
  - detail 不再出现窄屏内嵌 comments。
- app：
  - 窄屏三步流成立，close/back/escape 回退链路正确。
  - medium 默认两栏；点击 comments 后三栏；关闭后回两栏。
  - wide 的 comments 自动打开 + 持久化行为与当前一致。
- 回归：
  - `compile/test/build` 全部通过。
  - 受影响 smoke（popup/app 窄屏 header、escape、app comments sidebar）通过。
