# Audit P2 - webclipper-markdown-reading-profiles

> 记录 P2 的审计发现与修复闭环（由 executing-plans 在 P2 完成后自动进入）。

## Scope

- Phase: `P2`（`P2-T1` ~ `P2-T3`）
- Baseline commits:
  - `762c5019`（P2-T1）
  - `75ab1a66`（P2-T2）
  - `0c5fc4c1`（P2-T3）

## Read-only checks

1. Profile 完整性检查：
   - `rg -n "'medium'|'notion'|'book'|MarkdownReadingProfile|preset" webclipper/src/services/protocols webclipper/src/ui/shared`
2. 三风格可读性策略检查：
   - `rg -n "line-height|measure|max-w-|overflow-wrap|blockquote|pre|table|image-link" webclipper/src/ui/shared webclipper/src/ui/styles`
3. 矩阵测试检查：
   - `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/smoke/markdown-reading-profiles-matrix.test.ts`
4. 视觉风险抽查（按 profile 记录证据）：
   - 标题层级是否可区分
   - blockquote 与正文是否有足够层次
   - pre/table 是否局部横滚且不撑破正文
   - 图片说明链接是否可读且可换行

## Findings

1. Read-only checks 结果：未发现阻塞缺陷。
2. 已覆盖高风险项：
   - `notion` 紧凑排版：`fontSize/measure/lineHeight` 断言覆盖，且保持 `lineHeight >= 1.5`。
   - `book` 长文排版：serif + CJK fallback、更宽 measure 与更松段落节奏有断言覆盖。
   - 三风格矩阵：`profile x bubbleRole` + light/dark token scaffold 覆盖到 `tests/smoke/markdown-reading-profiles-matrix.test.ts`。
3. 审计验证：
   - `npm --prefix webclipper run test -- tests/unit/markdown-reading-profiles.test.ts tests/smoke/markdown-reading-profiles-matrix.test.ts` 通过。
4. 残余检查项（后续 phase 终验执行）：
   - 与设置链路打通后的运行时实测（切换 profile 后即时生效与重启持久化）。

## Decision

- Pass（可进入 P3）。
