# WebClipper：同心圆角统一（Apple 风格）

## 背景 / 触发

- 当前 WebClipper 的圆角风格未统一：按钮体系部分统一，但容器、面板、输入、tooltip、inpage comments 仍有大量硬编码半径。
- 用户明确要求达到 Apple 设备常见的“同心圆角”视觉：外层与内层半径应有固定层级关系，而不是各处随意取值。
- 本 feature 以“先统一、再扩展”为原则，先建立全局圆角 token 与分级规则，再覆盖 app / popup / inpage / shared 组件。

## 核心需求（原始需求精炼）

1. 建立 WebClipper 圆角单一真源（全局 `--radius-*` tokens），禁止新增散落硬编码半径。
2. 按同心分级统一主要 UI：主容器、卡片、控件、chip/徽标、内联代码块、圆形元素。
3. 覆盖范围包含 `app + popup + inpage + shared`，避免“局部看起来统一、整体仍分叉”。
4. 视觉目标为“同心分级”而非本轮引入 clip-path/mask squircle。
5. 改造后需能通过现有 WebClipper 验证链（compile/test/build）并保留交互行为（focus/hover/active）。

## 默认值与兼容策略

- 新旧用户统一采用同一套圆角 token（纯样式层改造，无数据迁移）。
- 默认分级策略：`outer > card > control > chip > inline`，圆形继续使用 pill（999px）。
- 历史 `tw-rounded-*` 与 `border-radius: xxpx` 逐步替换为 token；仅允许白名单场景保留 pill。

## 非目标（明确不做什么）

- 本次不引入复杂连续曲线算法（如 `clip-path` / `mask` / SVG squircle）作为默认实现。
- 不在本 feature 内改动业务逻辑、数据结构、消息协议或存储模型。
- 不新增主题切换机制，继续跟随 `prefers-color-scheme`。

## 验收标准（可检查）

- `tokens.css` 中存在完整圆角 token 集合，并在 app/popup/shadow 侧生效。
- 主要 UI 路径（Conversations、Settings、Popup、Inpage Comments、Menu/Tooltip）不再依赖散落硬编码半径。
- 通过扫描命令能定位剩余硬编码半径并收敛到白名单（主要是 pill）。
- `npm --prefix webclipper run compile`、`npm --prefix webclipper run test`、`npm --prefix webclipper run build` 可完成（若存在既有阻塞，需在执行记录注明与本 feature 的相关性）。
