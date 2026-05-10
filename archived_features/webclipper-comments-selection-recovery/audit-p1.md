# Audit P1 - webclipper-comments-selection-recovery

## Scope

- P1 新增的选区恢复契约、host gate 与坐标记录器
- 坐标反推解析器在基础单测维度的正确性

## 重点风险

- host 后缀匹配过宽导致误命中
- 坐标记录窗口过长导致引用到过期轨迹
- 反向拖拽处理不完整导致截取文本错误

## 审计记录（执行期填写）

- Finding A1 (Medium) — `recovery-host-gate.ts` 的 allowlist 归一化在结构上可能接受过宽后缀（例如仅 `cn`），存在后续扩展时误命中风险。
- Finding A2 (Low) — `coordinate-recorder.ts` 在无法读取 viewport 时会静默降级为无效轨迹；行为可接受，但调试可观测性偏弱。
- Finding A3 (Low) — `selection-recovery-resolver.ts` 对反向拖拽边界比较异常采用静默降级，当前正确但缺少诊断信息。

## 修复记录（执行期填写）

- Fix A1: 已修复。`recovery-host-gate.ts` 新增 `isScopedDomain()` 校验，allowlist 仅接受具备域名层级的 host；并补充 `comment-selection-host-gate.test.ts` 覆盖“过宽后缀被丢弃”。
- Fix A2: 本阶段不修复代码，维持静默降级语义（避免主流程噪声）；诊断增强延后至 P3。
- Fix A3: 本阶段不修复代码，维持当前“异常即降级为空”安全路径；诊断增强延后至 P3。

## 回归验证

- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile`
- [x] `npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- tests/unit/comment-selection-host-gate.test.ts tests/unit/comment-selection-recovery-resolver.test.ts tests/unit/comment-selection-coordinate-recorder.test.ts`
