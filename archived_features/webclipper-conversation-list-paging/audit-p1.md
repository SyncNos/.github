# Audit P1 - webclipper-conversation-list-paging

## 审计范围

- 覆盖 `P1-T1` ~ `P1-T6`：
  - 分页域模型
  - schema 索引与派生键迁移
  - storage-idb 分页/精确查找/summary-facets
  - message contracts/background handlers/repo API
  - 存储与路由定向测试

## 进入审计前置条件

- `todo.toml` 中 P1 tasks 全部 `completed`。
- 每个 task 至少有一次对应验证记录。

## 审计检查清单

1. 索引与迁移
- 是否存在稳定分页索引（含 tie-break 维度）。
- 历史数据派生键是否成功回填。
- 升级失败策略是否可解释且不会悄悄破坏一致性。

2. 分页正确性
- 排序是否稳定（`lastCapturedAt desc` + `id desc`）。
- 翻页是否无重复/无漏项。
- `hasMore` 是否在整页/尾页/空页边界正确。

3. 过滤与摘要口径
- summary/facets 是否与 query 一致。
- source/site 过滤是否不依赖每页全表扫描。

4. 精确查找能力
- `source + conversationKey` 与 `id` 查询是否精确。
- miss 场景是否返回可处理空结果（非崩溃）。

5. 协议一致性
- message contracts 命名是否与既有风格一致。
- handlers 入参校验和错误结构是否统一。
- `GET_CONVERSATIONS` / `listConversations` / `getConversations` 是否按计划删除或标记为阻断待删（不允许静默保留）。

## 风险重点

- 索引或迁移遗漏导致分页顺序异常。
- 派生键未维护导致筛选失真。
- by-loc/by-id 能打开但上下文字段不完整。
- 旧全量接口残留导致主链路意外回退。

## 审计记录（执行时填写）

- Findings:
  - [ ] 无阻断问题
  - [x] 存在阻断问题（已修复并复验通过）
- 阻断问题列表：
  1. 发现旧全量接口残留：`getConversations/listConversations` 仍在 `storage-idb/storage/background-storage` 内部可达。
  2. 该残留与“破坏性移除旧全量接口”约束冲突，存在后续误用风险。
- 修复提交：
  - `19d49e3e`（refactor: audit p1 - 移除残留全量列表接口）

- 审计结论：
  - 已移除残留旧接口并将相关存储测试改为分页 API 口径。
  - 复验命令通过，P1 通过审计门禁。

## 回归命令（审计后）

- `npm --prefix webclipper run compile`
- `npm --prefix webclipper run test -- tests/storage/conversations-pagination.test.ts tests/services/conversations-pagination-handlers.test.ts`
