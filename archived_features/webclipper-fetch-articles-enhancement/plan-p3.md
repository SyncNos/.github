# Plan P3 - webclipper-fetch-articles-enhancement

**Goal:** 收敛旧链路与重复逻辑，让 fetch articles 的实现“主干唯一、路径清晰”，并把对标 obsidian-clipper 的差异与验证清单沉淀到 deepwiki。

**Non-goals:** 本 phase 不做用户侧 UI 改造；不新增 i18n 文案（除非必要且另行确认）。

**Approach:** 清理 Readability 注入/旧 markdown 转换的残留与重复逻辑；在 deepwiki 补齐“文章抓取架构 + 验证清单 + 对标差异（Defuddle/Turndown）”。

**Acceptance:**
- fetch articles 主干链路唯一且清晰（Defuddle/Readability/fallback 的顺序与职责明确）。
- 有一份可复用的手动验证清单覆盖 Discourse/普通网页/已知站点 spec（公众号、小红书、Bilibili opus）。
- `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build` 通过。

---

<a id="p3-t1"></a>
## P3-T1 收敛/清理旧extract路径（Readability注入与重复逻辑）

**Files:**
- Modify: `webclipper/src/collectors/web/article-fetch.ts`
- Modify: `webclipper/src/collectors/web/article-extract/engine.ts`
- (Optional) Modify: `webclipper/src/vendor/readability.js`（仅在确认无需注入时才考虑移除引用，不改 vendor 内容本体）

**Step 1: 实现功能**
- 背景：如果 P2 引入 Defuddle 后成为主干抽取路径，`fetchActiveTabArticle()` 里对 Readability 的注入应重新评估：
  - 若 Readability 仅作为兜底，考虑延后注入或按需注入（减少不必要开销/失败噪音）
  - 收敛重复的“等待/抽取/转换”逻辑，确保主干路径只有一套
- 在 P1/P2 已执行“及时清理旧代码”的前提下，P3 只做最终扫尾：
  - 用 `rg`/TypeScript 编译确认旧实现无残留引用
  - 若仍有残留，再就地删除（但不作为主要工作项拖延到 P3）
- 保持：不改 UI；仅做内部结构与调用顺序的清理。

**Step 2: 验证**
Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test`

Expected: 编译与测试通过；Readability 注入失败的噪音与对功能的影响被收敛（仅作为兜底）。

**Step 3: 原子提交**
Run: `git add webclipper/src/collectors/web/article-fetch.ts webclipper/src/collectors/web/article-extract/engine.ts`

Run: `git commit -m "refactor: P3-T1 - 收敛fetch articles旧extract路径与Readability注入"`

---

<a id="p3-t2"></a>
## P3-T2 更新deepwiki：新fetch articles架构、验证清单、对标差异

**Files:**
- Modify: `.github/deepwiki/modules/webclipper.md`

**Step 1: 实现功能**
- 在 deepwiki 中补齐“fetch articles 新架构（通用抽取 + 通用转换）”章节，并附手动验证 checklist（仅供开发者）：
  - Discourse（linux.do）topic：首贴正文完整、图片/链接/列表结构可读
  - 普通网页：Readability path 正常
  - 已知 spec：WeChat share media、小红书 note、Bilibili opus 不回归
- 记录对标差异：
  - obsidian-clipper 的 Defuddle 抽取与 Turndown 规则点（以及它“不做 Discourse 特适配”的事实）
  - SyncNos 当前实现的差距与下一步（若有）

**Step 2: 验证**
Run: `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`

Expected: 文档变更不影响编译/测试/构建；并且 deepwiki 作为仓库文档应随代码一起入库。

**Step 3: 原子提交**
Run: `git add .github/deepwiki/modules/webclipper.md`

Run: `git commit -m "docs: P3-T2 - 补齐fetch articles验证清单与对标差异"`

---

## Phase Audit

- Audit file: `audit-p3.md`
- Rule: 完成本 phase 全部 tasks 后，`executing-plans` 必须自动进入该文件的审计闭环
- Flow:
  1. 先记录发现
  2. 再修复问题
  3. 再运行本 phase 验证命令（至少 `npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`）
