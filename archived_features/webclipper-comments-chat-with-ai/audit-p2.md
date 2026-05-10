# Audit P2 - webclipper-comments-chat-with-ai

## Findings

- ✅ comment 级 Chat with 已升级为“复制成功后自动跳转平台”。
- ✅ inpage 侧 header/comment 两条链路均已接入 background 打开能力，`window.open` 仅在 runtime 不可用时降级。
- ✅ background 新增 `chatwithOpenPlatformTab` 路由，基于 `platformId` + settings 解析，拒绝禁用/非法平台。
- ✅ message contract 与 re-export 已同步，避免类型漂移。

## Fixes

- `ad20780a`：P2-T1，新增 open-port 抽象，comment action 复制后自动跳转。
- `613f8007`：P2-T2，新增 chatwith background handler + message contract，同步 inpage 接入与烟测。

## Verification

Run:
- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/unit/chatwith-comment-actions.test.ts tests/smoke/background-router-chatwith-open-platform.test.ts tests/smoke/inpage-comments-sidebar-toggle.test.ts`

Result:
- compile 通过。
- 3 个测试文件通过，共 10 个测试通过。

