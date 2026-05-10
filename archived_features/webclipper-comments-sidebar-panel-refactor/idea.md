# WebClipper：右侧 Comments Sidebar（Threaded Comments Panel）彻底拆分重构

## 背景 / 触发

- 当前右侧 comments sidebar 的核心 UI 实现集中在单文件 `webclipper/src/services/comments/threaded-comments-panel.ts`（体积过大、职责混杂）。
- 该文件同时承担：Shadow DOM 挂载、样式注入、dockPage 顶开页面、宽度拖拽与持久化、Chat-with 菜单、评论渲染与交互、定位高亮、快捷键拦截、删除二次确认等多种职责，导致：
  - 迭代成本高（改一处容易牵动多处）
  - 难以复用/测试（大量 DOM 副作用掺杂在业务逻辑里）
  - 分层语义不清（UI 代码落在 `services/**` 下，且依赖 `@ui/styles/*?raw` / `@i18n`）

本 feature 目标是**彻底重构 + 拆分**，把右侧 comments sidebar 的实现收敛到清晰的模块边界，同时严格遵循 WebClipper 的 TS 分层规范与 import 约束。

## 核心需求（原始需求精炼）

1. **彻底拆分**右侧 comments sidebar（threaded comments panel）的实现：按职责拆成多个小模块，避免巨型文件。
2. **分层归位**：把 UI 挂载/Shadow DOM/DOM 事件等 UI 实现从 `services/**` 迁移到 `ui/**`（遵循依赖方向 `ui -> services`）。
3. **不引入臃肿**：重构以“拆分 + 收敛”为主，不新增多余抽象层、不引入框架、不引入无必要的通用工具。
4. **严格 TS 规范**：
   - 尽量消除 `any`；使用 `unknown + type guards`；将必要的 `as any` 集中在边界层。
   - 避免跨层 import：`ui/**` 不得 import `platform/**`；`services/**` 不得 import `ui/**`。
   - 保持导出 API 清晰、命名稳定（必要变更必须在 plan 中列出迁移策略）。
5. **行为不变（默认）**：用户可见行为（打开/关闭语义、dockPage、resize、Chat-with、locate、快捷键处理等）保持一致；仅做结构重构与可维护性提升。

## 默认值与兼容策略

- 默认不改 UI 文案与 i18n key，不改 CSS class/attribute contract（除非为拆分所必需且有明确迁移）。
- 若需要调整模块路径，优先在本仓库内一次性更新引用点，避免保留长期 shim。

## 非目标（明确不做什么）

- 不新增功能（如 iframe embed / 原生 sidepanel / per-tab companion 等）。
- 不改现有数据结构与存储协议（comments storage、message contracts、DB schema 等不在本 feature 范围）。
- 不引入 React 重写（保持现有 vanilla DOM + Shadow DOM 实现）。

## 验收标准（可检查）

- `mountThreadedCommentsPanel(...)` 仍可被 `app` 与 `inpage` 正常使用；行为保持一致。
- 原巨型文件被拆分为多个职责明确的小模块（单文件体积显著下降；无“上千行单文件”）。
- 编译与静态检查通过：
  - `npm --prefix webclipper run lint`
  - `npm --prefix webclipper run compile`
  - `npm --prefix webclipper run test`

