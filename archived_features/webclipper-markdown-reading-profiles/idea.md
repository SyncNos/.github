# WebClipper：Markdown 阅读风格协议化（Medium-first）

## 背景 / 触发

- 当前 Markdown 渲染偏“紧凑技术文档”观感，用户反馈为“字体不美观、阅读不舒服”。
- 用户已确认目标是三种阅读风格都要做：`Medium-like`、`Notion-like`、`Book-like`；但第一步先上线 `Medium-like`。
- 这次要求“协议驱动开发”：先把风格能力沉淀成稳定协议，再逐 phase 扩展，避免继续在 `ChatMessageBubble` 写散落硬编码 class。

## 核心需求（原始需求精炼）

1. 建立 Markdown 阅读风格协议（profile protocol），统一管理：`profile id`、默认值、回退策略、样式映射。
2. 第一阶段仅实现 `Medium-like`，并明确替换当前“小字紧凑”排版基线（字体、字号、行高、段距、标题层级、代码块、引用、图片说明）。
3. 第二阶段在同一协议下实现 `Notion-like` 与 `Book-like`，不得复制渲染入口。
4. 第三阶段接入设置页与持久化，支持运行时切换，脏值/未知值统一回退 `medium`。
5. 全过程不改 markdown 语义解析链（继续沿用 markdown-it plugin-first），仅改阅读样式层。

## 阅读体验硬指标（协议验收基线）

> 这些是“体验是否变好”的硬约束，不是可选建议。

- 行长（measure）：目标 `55-75ch`，硬上限不超过 `80` 字符；CJK 场景不超过 `40` 字符。
- 正文字体节奏：行高不低于 `1.5`，并允许在可访问性测试中提升到 `1.5em` 行高、`2em` 段距而不截断。
- 对比度：正文/链接在 light 与 dark 下都满足可读对比（普通文本至少 4.5:1；大文本至少 3:1）。
- 重排：在 `320 CSS px` 宽度下不出现正文双向滚动（`table`、`pre/code` 可局部横向滚动）。
- 断行策略：长链接、长 token 不得撑破容器（`overflow-wrap` 相关策略必须覆盖）。

## 默认值与兼容策略

- 默认风格：`medium`（新装/升级均一致）。
- 存储键缺失、异常或未知值：统一回退 `medium`。
- 升级策略：无需离线迁移脚本，采用运行时 normalize。
- 协议预留：稳定 id 为 `medium | notion | book`；后续新增 profile 必须先扩协议与测试，再改 UI。

## 非目标（明确不做什么）

- 不替换 markdown 渲染引擎，不从 `markdown-it` 迁移到其他 AST 流程。
- 不改会话数据结构、采集逻辑、同步逻辑（Notion/Obsidian）与导出格式。
- 不新增独立主题系统，仍遵循 `prefers-color-scheme`。
- 不引入“视觉好看但不可验证”的主观标准；必须有可执行验收项。

## 验收标准（可检查）

- 存在单一协议真源与 resolver，UI 不再散落硬编码 profile 分支。
- `Medium-like` 上线后，可在 `ChatMessageBubble` 明显改善正文阅读节奏（字号、行高、段距、行长）。
- 三种 profile 的 id/回退/preset 完整性均有单测覆盖。
- 设置页切换与持久化稳定，popup/app 重开后保持一致。
- 验证链至少包含：`npm --prefix webclipper run compile`、目标测试、`npm --prefix webclipper run build`。

## 参考依据（用于 plan 审查，不作为实现依赖）

- WCAG：1.4.3 Contrast Minimum、1.4.10 Reflow、1.4.12 Text Spacing、1.4.8 Visual Presentation
- Baymard（2022）：长文可读行长与 `70ch` 量级建议
- MDN：`line-height`、`overflow-wrap`、`prefers-reduced-motion`
- Carbon Typography：token 化排版体系（type tokens）
- `github-markdown-css` / `tailwindcss-typography`：成熟 markdown 排版基线
