# Audit P3 - threaded-comments-panel-react-migration

> Phase 审计闭环：先记录发现，再修复问题，再跑本 phase 验证命令。

## Findings

### 任务看板（来自 `todo.toml`）

- `P3-T1` 手工回归清单（app+inpage）
- `P3-T2` focus/scroll 最小可测单元
- `P3-T3` 分层边界自检
- `P3-T4` 最终 compile/test/build + 冒烟

### 发现 F-01

- 任务：`P3-T1`
- 严重级别：`High`
- 状态：`Resolved`
- 位置：`.github/features/threaded-comments-panel-react-migration/plan-p3.md:37`
- 摘要：P3-T1 要求对 `.github/features/...` 下的手工清单做 `git commit`，与当前协作约定（feature 目录作为本地执行证据链，默认不入库）冲突。
- 风险：执行时把临时计划/证据文件误提交进主仓库，引发噪音与后续漂移维护负担。
- 预期修复：在 `plan-p3.md` 中把 P3-T1 的“原子提交”步骤改为“落盘即可（不提交）”，并在 P3-T4 的 `git status` 预期里明确这些文件允许未提交。
- 验证：`git status --porcelain`
- 解决证据：已更新 `plan-p3.md`（明确 P3-T1 不做 `git commit`）。

### 发现 F-02

- 任务：`P3-T2`
- 严重级别：`Medium`
- 状态：`Resolved`
- 位置：`.github/features/threaded-comments-panel-react-migration/plan-p3.md:55`
- 摘要：focus-rules 的测试设计需要覆盖 Shadow DOM 下的“焦点在 panel 内”的判定，但计划当前描述仍停留在“activeElement 不在 panel 内”。
- 风险：测试用例可能写成直接读取 DOM 的 brittle 测试，或遗漏 shadowRoot 情况，导致关键规则回归无人兜底。
- 预期修复：在 `plan-p3.md` 明确：focus-rules 纯函数应接受 `hasFocusWithinPanel`（或等价信号）作为输入，Shadow DOM 的判定由 UI 层实现并单独做少量集成覆盖/手工覆盖。
- 验证：`npm --prefix webclipper run test`
- 解决证据：已更新 `plan-p3.md`（把规则改为 `hasFocusWithinPanel` 输入，避免 DOM brittle 测试）。

## Fixes

- `plan-p3.md` 已补齐：
  - P3-T1 不提交 feature 证据文件的约束
  - focus-rules 纯函数输入形状（`hasFocusWithinPanel`）与 Shadow DOM 注意点
  - 边界自检补充 `@platform/` 扫描

## Verification

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 全通过。

---

## Round 2 (post P3 completion)

Subagent evidence: `.github/features/threaded-comments-panel-react-migration/.audit/subagent-audit-p3.md`

### 结论

- Verdict: `PASS`
- Findings: `No material findings`

## Round 2 Verification

Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 全通过。
