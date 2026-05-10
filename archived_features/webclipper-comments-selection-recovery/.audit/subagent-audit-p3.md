现在我完成了 P3 只读审计的信息收集。根据计划、源码、测试与文档的交叉分析，输出以下审计报告：

---

## P3 只读审计报告：webclipper-comments-selection-recovery

### **OVERALL STATUS: FAIL** ⚠️

P3 两个 task 形式上已完成（提交、编译、测试通过），但存在**计划与实现不对齐、扩展策略缺陷、风险识别未闭环**的关键问题。

---

## Findings

### **C1 (HIGH) — Site Adapter 扩展设计缺失**
- **文件**: [policy/audit mismatch]
- **问题**: 
  - `audit-p3.md` 的"重点风险"明确指出需要审查 "site adapter 扩展位设计质量" 与 "adapter 注册策略与 host gate 规则一致性"
  - 实现代码完全没有 adapter registry / site adapter framework
  - recovery-host-gate.ts 中的 `DEFAULT_ALLOWED_HOSTS = Object.freeze(['www.dedao.cn', 'dedao.cn'])` 是纯硬编码

- **修复建议**: 
  - 澄清 P3 非目标：是否故意排除 site adapter 框架（应在 [plan-p3.md](plan-p3.md) 的 "Non-goals" 中明确声明） 
  - 或补齐设计：定义 `SelectionRecoverySiteAdapter` 接口与 registry 模式，虽不实现具体 adapter，但留出扩展点以满足 audit 预期

---

### **C2 (MEDIUM) — 审计计划与测试清单不闭环**
- **文件**: [audit-p3.md#L27-L31](audit-p3.md#L27-L31)
- **问题**:
  - 回归验证清单列出 `tests/unit/comment-selection-site-adapter-registry.test.ts`
  - 但此文件不存在，对应功能也未实现
  - 表明 audit 模板与实施脱节——或审计清单是从某个草稿复制未更新

- **修复建议**:
  - 更新 [audit-p3.md](audit-p3.md#L27-L31) 的回归验证清单，移除不存在的测试文件
  - 补齐实际执行过的测试：应包含 `comment-selection-coordinate-recorder.test.ts`（该文件存在但未在 audit 清单中）

---

### **C3 (MEDIUM) — Host Gate 缺乏扩展机制**
- **文件**: [recovery-host-gate.ts](webclipper/src/services/comments/selection/recovery-host-gate.ts#L1)
- **问题**:
  - [plan-p3.md](plan-p3.md#L27) 承诺"后续扩展策略：先新增白名单与回归，再评估是否需要站点专属 adapter"
  - 但 `DEFAULT_ALLOWED_HOSTS` 硬编码，没有配置读取或注册 API
  - 对下一个站点的支持仅能通过改源代码 + 重新编译
  - 与"稳定而非扩张"的思想相悖——稳定应该包含"低成本扩展"

- **修复建议**:
  - 至少提供 `createSelectionRecoveryHostGate({ allowlist?: string[] })` 的参数化构造
  - 考虑在后续阶段从配置或 runtime settings 读取白名单（不必在 P3 实现，但代码注释应标记未来扩展点）

---

### **C4 (LOW) — 文档重复与事实一致性**
- **文件**: [deepwiki/modules/comments.md](../.github/deepwiki/modules/comments.md#L142-L146) vs [compatibility-checklist.md](compatibility-checklist.md#L7-L30)
- **问题**:
  - deepwiki 中 "inpage 选区恢复策略" 与 compatibility-checklist 中"浏览器能力矩阵"部分重复描述
  - deepwiki 给出的细节（如原因码内容）多于 compatibility-checklist，产生冗余
  - AGENTS.md 与两个 deepwiki 模块（comments.md + webclipper.md）间的叙述风格、准确度不统一

- **修复建议**:
  - 在 [deepwiki/modules/comments.md](../.github/deepwiki/modules/comments.md) 中去掉实现细节，只保留高层链接与行为边界
  - 把实现细节（浏览器 API 名称、原因码枚举、降级路径）收敛到 [compatibility-checklist.md](compatibility-checklist.md) 作为唯一事实源
  - 审视 AGENTS.md 中的相关段落，确保与 deepwiki 内容不冲突

---

### **C5 (LOW) — 浏览器兜底测试边界覆盖不完整**
- **文件**: [comment-selection-recovery-browser-fallback.test.ts](webclipper/tests/unit/comment-selection-recovery-browser-fallback.test.ts)
- **问题**:
  - 测试覆盖 ✅ caretRangeFromPoint + caretPositionFromPoint 兼用（preferring test）
  - 测试覆盖 ✅ caretRangeFromPoint unavailable 时回退
  - 测试覆盖 ✅ caretRangeFromPoint throw 时回退
  - 缺失 ❌ "caretPositionFromPoint 也返回 null 或 invalid offsetNode" 的边界（虽然 resolver 中有 `!caret?.offsetNode` 的检查，但测试未验证该路径）

- **修复建议**:
  - 补充测试用例：`{offsetNode: null, offset: 2}` 返回时的降级路径；`caretPositionFromPoint` 返回非 null 但 offsetNode 不在 root 内的场景

---

## 建议验证命令

```bash
# 1. 类型检查 + 编译
npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run compile

# 2. P3 相关单元测试（按 compatibility-checklist 的完整列表）
npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run test -- \
  tests/unit/inpage-selection-recovery-fallback.test.ts \
  tests/unit/comment-selection-recovery-resolver.test.ts \
  tests/unit/comment-selection-recovery-browser-fallback.test.ts \
  tests/unit/comment-selection-coordinate-recorder.test.ts

# 3. 生产构建验证
npm --prefix /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper run build

# 4. 核对问题 C1：验证是否存在 site adapter 相关代码
rg -i "adapter|registry" \
  /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper/src/services/comments/selection/

# 5. 核对是否有遗漏的测试（audit-p3.md 清单中的所有文件是否存在）
ls -la /Users/chii_magnus/Github_OpenSource/SyncNos/webclipper/tests/unit/{inpage-selection-recovery-fallback,comment-selection-recovery-resolver,comment-selection-recovery-browser-fallback,comment-selection-coordinate-recorder}.test.ts
```

---

## 风险观察

| 等级 | 观察 | 影响 |
|------|------|------|
| 🔴 High | 审计计划（P3 risk）与实现完全不对齐；**未提交修复就声称问题已闭环** | 后续审计信用度下降；P3 风险可能在生产中复现 |
| 🟡 Medium | Host gate 无配置扩展机制 + 缺乏 adapter registry 框架 | 下一个白名单站点的支持成本高（需改源代码 + 回归全套测试） |
| 🟡 Medium | 文档重复 + 事实源不唯一 | 后续维护时改一处会遗漏另一处；新增站点时文档改动成本高 |
| 🟢 Low | 浏览器兜底测试有边界覆盖缺口 | 在罕见浏览器行为下可能出现降级路径未被测试的情况 |

---

## 建议下一步行动

1. **立即**（Blocking）：
   - 确认 P3 是否故意不做 site adapter registry（需在代码注释或 plan 中明确）
   - 更新 [audit-p3.md](audit-p3.md) 的验证清单，移除、补齐或新增实际执行的测试

2. **短期**（Before merge/release）：
   - 为 host gate 补齐参数化构造，便于单测扩展
   - 补充 browser fallback 的 null offsetNode 边界测试

3. **中期**（For next phase or feature）：
   - 设计 site adapter registry 接口（虽 P3 不实现具体 adapter，但框架应在 P4 或 feature branch 中就绪）
   - 统一文档事实源，deprecate 重复段落