# webclipper-comments-selection-recovery · compatibility checklist

> 范围：仅评论侧栏选区恢复链路（inpage root composer），不扩展到 comments 之外模块。

## 1) 浏览器能力矩阵

| 浏览器 | `caretRangeFromPoint` | `caretPositionFromPoint` | 预期链路 |
| --- | --- | --- | --- |
| Chrome (MV3) | ✅ | 可能有 | native -> caretRange -> caretPosition |
| Edge (MV3) | ✅ | 可能有 | native -> caretRange -> caretPosition |
| Firefox | ❌/不稳定 | ✅ | native -> caretPosition |
| 其它/受限内核 | ❌ | ❌ | native only（`point-failed` 降级） |

## 2) 站点与门控

| 场景 | 预期 |
| --- | --- |
| host = `www.dedao.cn` / `dedao.cn` | native 为空时允许坐标反推 |
| 非白名单 host | 不触发坐标反推，保持旧行为 |
| host 白名单配置过宽（如 `cn`） | 应被门控归一化过滤 |

## 3) 交互方向与文本恢复

- [ ] 左 -> 右拖拽可恢复 `quoteText`
- [ ] 右 -> 左拖拽可恢复 `quoteText`
- [ ] 跨段落拖拽可恢复 `quoteText`
- [ ] root composer 自动触发（`trigger: auto`）可附加引用
- [ ] root composer 手动触发（`trigger: button`）可附加引用
- [ ] reply 输入框不会触发附加/清空引用

## 4) 失败降级语义

| reasonCode | 触发条件 | 预期行为 |
| --- | --- | --- |
| `native-empty` | native 为空且 host 未命中白名单 | 返回空引用，不抛异常 |
| `point-failed` | point API 不可用/返回空/异常 | 返回空引用，不抛异常 |
| `outside-root` | 重建范围落在 root 外 | 返回空引用，不抛异常 |
| `root-null` | 未能提供有效 root | 返回空引用，不抛异常 |
| `locator-null` | quote 可得但 locator 构建失败 | 保存 `quoteText`，`locator = null` |

## 5) 运行时诊断

- [ ] 默认无调试噪声（控制台不输出 recovery 诊断）
- [ ] 设置 `globalThis.__SYNCNOS_DEBUG_SELECTION_RECOVERY__ = true` 后可看到 reasonCode 诊断

## 6) 回归命令

- [ ] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`
- [ ] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/inpage-selection-recovery-fallback.test.ts tests/unit/comment-selection-recovery-resolver.test.ts tests/unit/comment-selection-recovery-browser-fallback.test.ts tests/unit/comment-selection-coordinate-recorder.test.ts`
- [ ] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build`
