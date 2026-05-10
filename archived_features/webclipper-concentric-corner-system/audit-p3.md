# Audit P3 - webclipper-concentric-corner-system

> 记录 P3 的审计发现与修复闭环（由 executing-plans 在 P3 完成后自动进入）。

## Scope

- Phase: `P3`（`P3-T1` ~ `P3-T3`）
- Baseline commits:
  - `d927d827`（`P3-T1`）
  - `4c72b3bd`（`P3-T2`）
  - `89a43c49`（`P3-T3`）

## Read-only checks

1. 边缘样式圆角收敛：
   - `rg -n "border-radius:\s*[0-9]" webclipper/src/ui/styles/tooltip.css webclipper/src/ui/styles/inpage-tip.css webclipper/src/ui/comments/locate.ts`
2. markdown 内容块圆角收敛：
   - `rg -n "tw-rounded-\[|border-radius" webclipper/src/ui/shared/ChatMessageBubble.tsx`
3. 全局硬编码扫描：
   - `rg -n "border-radius:\s*[0-9]|tw-rounded-\[" webclipper/src/ui webclipper/src/entrypoints`
4. 核心门禁验证链：
   - `npm --prefix webclipper run compile`
   - `npm --prefix webclipper run test -- tests/smoke/inpage-comments-sidebar-toggle.test.ts tests/unit/chat-message-bubble.test.ts tests/smoke/inpage-tip-speech-bubble.test.ts`
   - `npm --prefix webclipper run build`
5. 全量回归现状（非阻塞）：
   - `npm --prefix webclipper run test`
   - 若失败，记录失败用例并标注是否与圆角改动相关

## Findings

- 无阻塞发现。
- `tooltip.css` / `inpage-tip.css` / `locate.ts` 未检出固定 px 的 `border-radius`。
- `ChatMessageBubble.tsx` 的气泡与 markdown 内联块圆角已收敛到 `--radius-chip` / `--radius-inline`。
- 全局扫描剩余项均可解释：
  - `tw-rounded-[var(--radius-*)]`（token 驱动，符合规范）
  - `webclipper/src/ui/styles/inpage-comments-panel.css` 的 `border-radius: 0`（reset 白名单）
  - `webclipper/src/ui/example.html`（示例页白名单）
- 核心门禁通过：`compile + targeted tests + build`。
- 全量 `npm test` 非阻塞失败共 8 条，集中在 4 个 smoke 文件，报错主因仍为 `MutationObserver is not defined` / tooltip 事件兼容：
  - `tests/smoke/popup-shell-header-actions.test.ts`（3）
  - `tests/smoke/app-shell-comments-sidebar.test.ts`（2）
  - `tests/smoke/app-shell-narrow-header-actions.test.ts`（2）
  - `tests/smoke/app-shell-sidebar-collapse.test.ts`（1）
  结论：与本次圆角替换无直接逻辑耦合，属于既有测试环境问题基线。

## Decision

- `P3` 审计通过，本 feature 可交付。
