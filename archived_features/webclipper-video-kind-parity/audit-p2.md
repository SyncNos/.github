# Audit P2 - webclipper-video-kind-parity

- 审计方式：本地只读审查 + 修复闭环（本轮未使用 subagent）
- 审计范围：`plan-p2.md`
- feature 目录：`.github/features/webclipper-video-kind-parity/`
- 粒度：`phase`

## 任务看板（P2）

- [x] P2-T1 Add new IDB comments store + migration from article_comments (`e958e406`)
- [x] P2-T2 Switch comments storage implementation to new store (keep compatibility) (`4041e4e0`)
- [x] P2-T3 Remove/lock down legacy article_comments write paths (`318d2ced`)

## 发现项

### 发现 F-01（High）

- 严重级别：`High`
- 状态：`Resolved`
- 摘要：Backup export/import 仍只读写 `article_comments` store；P2 已把新增评论写入 `comments` store，导致新评论不会进入备份，也无法从备份恢复。
- 影响：用户执行 Backup 导出后，comments 数据不完整；恢复后 comments 丢失。
- 修复方向：
  - 导出：从 `comments` store（优先）生成 `assets/article-comments/index.json`（短期沿用旧文件名/manifest 字段，保持兼容）
  - 导入：把 `assets/article-comments/index.json` 导入到 `comments` store（优先）；仅当 `comments` store 不存在时才回退 `article_comments`（但默认不再写 legacy）
- 修复提交：`2c7d3b5f`

## 验证日志

- `rtk npm --prefix webclipper run compile` -> `PASS`（P2-T3 后）
- `rtk npm --prefix webclipper run test` -> `PASS`（修复 F-01 后）

## Gate（是否允许进入下一阶段）

- 结论：`Go`
- 理由：P2 的 comments store 切换已覆盖读取/写入/attach+migrate 兼容路径，且备份导入导出已适配新 `comments` store，不会丢数据。
