## 动机

SyncNos 的底色是「把信息收集到 Notion」——读书标注、得到笔记、微信文章、AI 对话已经走通。**视频是一直缺的一条网**——尤其是讲解类、访谈类、技术分享类视频，字幕是其中信息密度最高、最易索引和后续处理的形态。补上这一段，「信息获取」的拼图才算闭合。

## 要什么（P1）

用户在浏览器里看 YouTube / B 站视频时，SyncNos WebClipper 把这个视频的 **字幕文本 + 元信息** 采集到 Notion。

- 字幕文本：带时间戳，完整。
- 元信息：标题、作者/UP 主、时长、封面、源 URL。

## 需求边界

| 维度 | 约束 |
| --- | --- |
| 平台 | 仅 YouTube + Bilibili |
| 运行环境 | WebClipper 浏览器扩展（用户本地） |
| 触发方式 | 用户主动采集（点按钮），不自动后台抓取 |
| 交付物 | 字幕文本 + 元信息，写入 SyncNos-Web Articles 数据库 |
| 无字幕 | 跳过；只存元信息。P2 再议 ASR |

## 不做（划界）

- 本地语音识别 / 无字幕视频的 ASR
- YouTube / B 站以外的平台（小红书、抖音、TikTok… P3+）
- 字幕翻译 / 摘要 / AI 加工（这是 Notion 端或后续应用的事）
- 视频本体下载
- 绕过用户的被动采集（打开网页就偺下来）

## 红线

- **不部署本地 ASR 模型**（违背 WebClipper 轻量定位）
- **不维护个人 cookie 池**（不用开发者自己的账号代理用户请求，封号风险不值得）
- **不依赖第三方付费 API**（savesubs 之类的外包）
- **不逆向 / 不跟签名**（PoToken、WBI 签名这些融化的冰面，不站）

## 成功标准

- 用户看完一个带字幕的视频、点一下采集，Notion 里能查到完整字幕 + 元信息
- 采集行为不引入用户的平台账号风险（不偷用登录态、不伪造请求）
- 实现不依赖任何会「过期」的平台签名 / 内部接口：即使 YouTube 或 B 站明天改接口，我们也不用救火

---

## 🔧 技术实现参考（精简版，2026-04-18）

> 核心思路一句话：**我们不发请求、只做观众** —— MAIN world content script 注入，hook 页面自己的 `fetch` / `XHR`，在页面已认证的前提下拿到字幕响应。所有和平台的反爬对抗（PoToken / WBI 签名 / Cookie）全部外包给用户的浏览器。
> 

### 核心技术 1 · MAIN world content script 拦截请求

这是整套方案的技术支点，没这一步其他都不成立。

- MV3 下 `chrome.webRequest.onBeforeRequest` **无法读 response body**，只能拿 URL/headers/statusCode → 这条路不通。
- **正解**：`manifest.json` 里声明 `"world": "MAIN"` + `"run_at": "document_start"` 的 content script，在页面脚本之前注入，重写 `globalThis.fetch` 和 `XMLHttpRequest.prototype.{open,send}`，拿到响应后通过 `window.postMessage` 传回 ISOLATED world → `chrome.runtime.sendMessage` 传到 background。
- 代价：**用户必须先在播放器里开启字幕**，字幕请求才会发出、才会被拦到。UI 上要明确提示。

**参考**：

- [Chrome Extensions ·](https://developer.chrome.com/docs/extensions/reference/api/scripting#type-ExecutionWorld) [scripting.world](http://scripting.world) [MAIN 官方文档](https://developer.chrome.com/docs/extensions/reference/api/scripting#type-ExecutionWorld)
- [rxliuli/intercept-network-requests-in-chrome-extension](https://github.com/rxliuli/intercept-network-requests-in-chrome-extension) —— MAIN world hook fetch/XHR 的最小可用实现

### 核心技术 2 · YouTube 字幕采集

**拦截目标**：`https://www.youtube.com/api/timedtext?...&fmt=vtt`（或 xml/json3 格式）

**配套数据**：页面里的 `window.ytInitialPlayerResponse.captions.playerCaptionsTracklistRenderer.captionTracks[]` —— 含每条轨道的 `languageCode / name / baseUrl / kind`（`kind="asr"` 表示自动生成）。

**格式解析**：VTT / XML / json3 三选一，统一归一成 `[{start, duration, text}]`。

**参考**：

- [algolia/youtube-captions-scraper](https://github.com/algolia/youtube-captions-scraper) —— `ytInitialPlayerResponse` → `baseUrl` 的解析器，VTT/XML parse 逻辑可直接复用（**只抄解析，不抄发请求那半**）
- [DEV · Download YouTube subs in a few lines](https://dev.to/wimdenherder/download-the-subs-with-a-few-lines-of-code-2k0) —— `ytInitialPlayerResponse` 字段结构参考
- [DEV · Chrome Extension transcript exporter](https://dev.to/namurema/how-i-built-a-youtube-transcript-exporter-chrome-extension-26l8) —— 扩展架构参考

### 核心技术 3 · Bilibili 字幕采集

**拦截目标**（按优先级）：

1. `//i0.hdslb.com/bfs/subtitle/*.json` —— 字幕本体，JSON 结构 `{body: [{from, to, content}]}`，带 token 直接可读。**这个是首选拦截点**，因为它就是字幕内容本身。
2. `api.bilibili.com/x/player/wbi/v2?aid=...&cid=...` —— 返回 `data.subtitle.subtitles[]`，里面的 `subtitle_url` 指向上面那个 JSON。

**关键事实**：B 站字幕覆盖率（尤其 AI 自动字幕）显著低于 YouTube，匿名态下常常返回空 `subtitles` 数组 —— UI 要诚实告诉用户"此视频无字幕"，不做伪造。

**参考**：

- [bilibili-API-collect · video/info 字幕字段](https://github.com/pskdje/bilibili-API-collect/blob/main/docs/video/info.md) —— 社区维护的接口文档
- [2dgirlismywaifu/bilibili_json2SRT](https://github.com/2dgirlismywaifu/bilibili_json2SRT) —— 字幕 JSON → SRT 转换逻辑
- [Greasyfork · B站字幕导出助手 568469](https://greasyfork.org/en/scripts/568469-b%E7%AB%99%E5%AD%97%E5%B9%95%E5%AF%BC%E5%87%BA%E5%8A%A9%E6%89%8B-%E6%82%AC%E6%B5%AE%E7%AA%97%E7%89%88/code) —— 油猴版"拦截字幕 JSON 请求"的最小实现，思路对齐

### 兜底技术 · DOM 抓字幕面板

拦截失败时使用，零网络风险，但时间戳精度降到秒级、UI 改版会破。

- **YouTube**：模拟点击 `aria-label="Show transcript"`，读 `#segments-container` 下的 `ytd-transcript-segment-renderer`（每个含时间戳 + 文本）。
- **Bilibili**：读 `.bpx-player-subtitle-panel-text` 逐条。
- **不可行**：`video.textTracks[i].cues` —— YouTube / B 站都是自绘字幕，不挂在原生 `<video>` 上。

**参考**：

- [Adam's Blog · YouTube transcript copy-paste script](http://www.adambourg.com/javascript/youtube/web-scraping/ai/2025/05/20/script_to_copy_paste_youtube_transcripts.html)

### 元信息抽取

字幕采集失败也要跑，零依赖。

- **YouTube**：优先 `window.ytInitialPlayerResponse.videoDetails` → `title / author / lengthSeconds / thumbnail.thumbnails[]`；退到 `<script type="application/ld+json">` 里的 [schema.org](http://schema.org) [VideoObject](https://schema.org/VideoObject)；最后 `<meta property="og:*">`。
- **Bilibili**：优先 `window.__INITIAL_STATE__.videoData` → `aid / bvid / cid / title / owner.name / duration / pic`；退到 `og:*`。

**参考**：

- [yt-dlp · bilibili extractor](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/bilibili.py) —— `__INITIAL_STATE__` 字段路径参考

---

## ✅ 推荐技术方案（给 WebClipper 实现）

**三层降级**：

1. **Layer A · 被动拦截（首选）** — MAIN world content script 在 `document_start` 注入，hook `fetch` + `XHR.prototype.{open,send}`。缓存所有响应到一个以 URL 为 key 的 Map。
    - YouTube：匹配 `/api/timedtext`，直接拿 VTT/XML 文本解析。
    - Bilibili：匹配 `api.bilibili.com/x/player/wbi/v2`（抠 `data.subtitle.subtitles[].subtitle_url`）**和** `i0.hdslb.com/bfs/subtitle/*.json`（直接就是字幕 JSON）。
    - 点采集按钮时，优先从缓存里取。
    - **前提提示**：如果缓存为空，UI 明确说"请先在播放器里开启字幕，再点采集"。
2. **Layer B · DOM 兜底** — 模拟点击 YouTube "Show transcript" / B 站字幕面板，从 DOM 抠文本。精度降到秒级，但零网络风险。
3. **Layer C · 纯元信息** — 字幕完全不可得时，只写入标题 / 作者 / 时长 / 封面 / 源 URL。诚实地告诉用户"此视频无字幕"。

**元信息始终跑** —— 从 `ytInitialPlayerResponse.videoDetails` / `window.__INITIAL_STATE__.videoData` 读，失败回退 `ld+json` → `og:*`。

**写入目标**：[SyncNos-Web Articles](https://www.notion.so/335be9d6386a8153a5ccc9afbc994620?pvs=21)，字段建议：`Title / Source URL / Author / Duration / Thumbnail / Captured At / Transcript(正文) / Transcript Source(A/B/C) / Has Timestamps(bool)`。

**明确不做**（硬红线）：

- ❌ 不实现 / 不调用 BotGuard PoToken 生成逻辑
- ❌ 不实现 WBI 签名逻辑
- ❌ 不要求用户填 Cookie（依赖浏览器自身登录态）
- ❌ 不做本地 ASR（P2 再议）
- ❌ 不做批量 / 后台 / 其他平台（P1 只管 YouTube + Bilibili）

**为什么这套能活** —— 因为我们把所有"需要和平台的反爬对抗"的部分全部外包给**用户的浏览器**。浏览器能播放字幕，我们就能采集字幕；浏览器播放不了，我们也老实说播放不了。这是扩展形态最健康的姿态，也是 2026 年唯一还没翻车的姿态。