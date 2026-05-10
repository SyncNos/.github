# 背景 / 触发

- 反馈：WebClipper 的 `fetch articles` 在 Discourse（例如 linux.do）上抓取正文不完整；典型例子：`https://linux.do/t/topic/1635410`。
- 现象：抓取结果只包含 onebox/少量文本，缺失首贴正文的大段内容（列表/段落/图片等）。
- 对标：`/Users/chii_magnus/Github_OpenSource/obsidian-clipper-main` 在同类页面上可以抓到更完整的内容。它主要走“通用抽取（Defuddle）+ 通用 HTML→Markdown（Turndown + 规则）”，并没有看到 Discourse 的专用适配；因此我们也应把 `fetch articles` 做成 **通用、可扩展、可回归验证**的一条主干链路。

# 核心需求（原始需求精炼）

1. 把 `fetch articles` 重构为“**通用抽取 + 通用转换**”主干链路：从当前页面 DOM 得到稳定的 `contentHtml` + 元信息，然后转换成结构保真的 `contentMarkdown`。
2. Discourse（linux.do）必须作为强约束用例：`https://linux.do/t/topic/1635410` 能稳定抓到首贴正文，不再被 onebox/内嵌 `<article>` 结构截断。
3. 目标是“通用优先”：不为 Discourse 单独写大量站点适配；但 **必须保留** 现有已落地的站点特化能力（例如公众号、 小红书、Bilibili opus），并在重构过程中纳入回归测试矩阵。
4. 可回归：新增单测/冒烟测试覆盖关键结构（onebox、details、blockquote、code、table 等），避免“修一次坏一次”。
5. 性能与体积可控：引入新依赖（例如 Defuddle/Turndown）要有明确的 bundle-size 与运行时开销评估，不在 P1 就一口气上所有规则。
6. **及时清理旧代码**：每个迁移 task 完成后应同步移除已不再需要的旧实现/旁路 API，避免“新旧共存拖到最后统一清理”导致维护成本与回归风险指数上升。

# 现有“站点单独适配 / 特化逻辑”清单（重构必须保留）

> 这里的“适配”指：目前 `fetch articles` 链路中，已经存在的、明确针对某些站点结构/行为做的抽取或稳定性增强。

1. **Discourse（linux.do 等）**
   - URL 归一化：`canonicalizeArticleUrl()` 会将 `/t/<slug>/<topicId>/<postNo>` 归一到 `/t/<slug>/<topicId>`，并清空 search。
   - OP-only 抽取：content 侧 `extractDiscourseOpOnly()` 会优先定位 `post-number=1` 的首贴并读取 `.cooked`。
   - /1 回退导航：background 侧在抓取非 1 楼并缺失 OP 的情况下，会跳转到 `/1` 再抓取（不涉及 `.json`）。
2. **微信公众号（mp.weixin.qq.com）**
   - `#js_content` 优先作为正文根节点；并对 root 做可见性修正 + 移除噪音节点（a11y/like tips）。
   - Share media 图集增强：识别 `.share_content_page + #img_swiper_content`，抽取 swiper 图片并追加为 image blocks（避免 table）。
   - Hydration wait：等待 swiper 图片挂载完成后再抽取，降低空图/缺图概率。
3. **小红书 Note（xiaohongshu.com）**
   - Site spec 抽取：基于 `#noteContainer` 的结构化 selector（作者/时间/正文/图片）。
   - Hydration wait：等待 note 容器文本/图片达到阈值再抽取，降低“只抓到壳子”的概率。
4. **Bilibili opus（bilibili.com/opus）**
   - Site spec 抽取：基于 `.bili-opus-view` 的结构化 selector。
   - 图片 URL sanitizer：去掉图片 URL 的 `@858w_...` 后缀，提升可用性。

# 默认值与兼容策略

- 默认策略：对用户无 UI/设置变化；仅提升抓取稳定性与完整性。
- 兼容：不主动迁移存量已保存的 conversation；对同一 Discourse topic 的 URL 归一化策略（canonical url）以“未来新抓取不重复”为目标，不强制合并历史记录。
- 兜底策略：本 feature 不启用 Discourse topic `.json` 兜底（避免引入额外网络请求与跨域不确定性）；优先把 DOM 抽取与 HTML→Markdown 做到“像 Obsidian 一样通用且稳”。
- Markdown 兼容：以 Turndown 为唯一 HTML→Markdown 主干；当 Turndown 产出空时，仅回退到 `textContent`（纯文本），不再保留旧的手写 HTML→Markdown 转换器兜底。

# 非目标（明确不做什么）

- 本次不做：为所有网站引入复杂的“全站通用 extractor 大重写”。
- 本次不做：新增设置开关/复杂 UI（除非后续发现必须暴露给用户）。
- 本次不做：国际化文案调整（除非确实需要新增错误提示，且再单独评估）。
- 本次不做：对 Discourse 做“逐站点 CSS selector 适配”扩散（只允许极少量、明确可复用的通用修正）。

# 验收标准（可检查）

- 在 Chrome MV3 开发模式下，对 `https://linux.do/t/topic/1635410` 执行 `fetch article`：
  - 抓取到的 `article_body` 文本长度显著大于当前（例如 > 2,000 chars），包含首贴中的多段文本/列表项关键字。
  - Markdown 中包含列表（`- ...` 或 `1. ...`）与外链（`[text](url)`）等基础结构，而不仅仅是一段 onebox 摘要。
- 现有站点特化能力不回归（至少保持现有测试通过）：
  - WeChat share media 图集依旧输出为 image blocks（不是 table）
  - 小红书 note 在 hydrated DOM 上仍能抓到正文与图片
  - Bilibili opus 仍能抓到图片与正文且去掉 `@...` 后缀
- 相关单测/冒烟测试通过：`npm --prefix webclipper run compile && npm --prefix webclipper run test && npm --prefix webclipper run build`。
- 对标实现参考被落实为可执行的实现 task（至少包含：通用抽取策略、HTML 清洗、Turndown 风格转换与规则集、Discourse onebox 回归用例）。
