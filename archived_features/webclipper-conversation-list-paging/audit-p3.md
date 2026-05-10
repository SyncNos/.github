# Audit P3 - webclipper-conversation-list-paging

## 审计范围

- 覆盖 `P3-T1` ~ `P3-T6`：
  - AppShell loc 精确打开链路
  - Insight 跳转去全量依赖
  - 详情头/评论侧栏“目标未加载”兼容
  - deep-link/locate/窄屏桥接回归
  - 文档同步与最终验证归因

## 进入审计前置条件

- `todo.toml` 中 P3 tasks 全部 `completed`。
- compile/test/build 结果可追溯（命令 + 摘要 + 归因）。

## 审计检查清单

1. deep-link 契约
- `loc` 编码兼容是否保持。
- 目标未加载时是否可精确打开并最终定位。
- `processedLocRef` 相关逻辑是否不会误吞外部 loc。

2. Insight 联动
- Top conversations 跳转是否不依赖全量 `items`。
- 点击行为在 app/popup/narrow 下是否一致。

3. 上下文完整性
- 目标未加载时，详情标题/URL/动作是否仍可用。
- 评论侧栏上下文是否稳定。

4. 窄屏桥接
- `pending-open` 一次性消费语义是否保持。
- list/detail 往返 scroll restore 是否正常。

5. 文档与验证
- `webclipper/AGENTS.md` 与 deepwiki 是否同步到新语义。
- 已知阻塞是否标注“是否由本 feature 引入”。

6. 破坏性切换收口
- `rg` 残留检查是否通过（无 `GET_CONVERSATIONS/listConversations/getConversations` 主链路调用）。
- 是否不存在“为兼容保留旧接口”的运行时分支。

## 风险重点

- loc 可打开但列表不定位。
- Insight 跳转偶发失效（缺字段或状态竞态）。
- 文档口径落后于实现。
- 旧全量接口未清干净，导致后续维护出现双轨逻辑。

## 审计记录（执行时填写）

- Findings:
  - [x] 无阻断问题
  - [ ] 存在阻断问题（需列出）
- 阻断问题列表：
  1. 无
- 修复提交：
  - `82db4bc9`（task15）
  - `6d7ed1bc`（task16）
  - `0aa87453`（task17）
  - `f9ce355b`（task18）
  - `6fde8bb0`（task19）
- 最终验证摘要：
  - compile: `npm --prefix webclipper run compile` 通过。
  - test: `npm --prefix webclipper run test` 通过（`108` files / `438` tests 全绿）；存在少量 `act(...)` 警告与 `--localstorage-file` warning，不影响结果。
  - build: `npm --prefix webclipper run build` 通过（WXT chrome-mv3 产物生成成功）。
  - known blockers: 无。
  - 残留检查：`rg -n "listConversations\\(|GET_CONVERSATIONS|getConversations\\(" webclipper/src webclipper/tests` 无命中（exit code 1）。

## 回归命令（审计后）

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test`
- `npm --prefix webclipper run build`
