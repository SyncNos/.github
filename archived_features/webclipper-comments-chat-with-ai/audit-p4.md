# Audit P4 - webclipper-comments-chat-with-ai

## Findings

- ✅ 已建立 `tabGroups` 能力封装与特性检测，Firefox 构建可通过并自动降级。
- ✅ background 已支持同文章同平台 AI tab 复用（`platformId::articleKey`）与分组流程。
- ✅ inpage 与 app 评论 ChatWith 已接入分组消息通道；失败时可回退到 Phase 2 打开能力。
- ✅ 全量回归（`npm run test`）通过，未引入新失败；`act(...)` 警告为既有噪声。

## Fixes

- `cb0258e3`：P4-T1，引入 `tab-groups` 封装、`tabsMove` 能力、`tabGroups` 权限与基础单测。
- `e89fa0f6`：P4-T2，新增 tabgroup store/runner、background grouped message、unit+smoke 覆盖。
- `0843358b`：P4-T3，inpage/app 接入 grouped open/focus 与降级回退，补充 comment action 断言。

## Verification

Run:
- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test`
- `npm --prefix webclipper run build`
- `npm --prefix webclipper run build:firefox`
- `npm --prefix webclipper run check`

Result:
- compile 通过。
- test 通过（`Test Files 133 passed (133)`）。
- build / build:firefox / check 全部通过。

