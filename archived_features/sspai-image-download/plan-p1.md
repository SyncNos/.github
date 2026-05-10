# Phase 1 Plan: 通用防盗链图片下载基础设施

## Goal

- 添加必要的权限和基础设施，使扩展能够修改特定域名请求的 Referer header
- 创建**通用**的防盗链图片下载辅助模块（支持多网站配置、Firefox 兼容）
- 扩展 `imageSanitizer` 类型支持去除查询参数

## Non-goals

- 不修改其他网站的图片下载逻辑
- 不实现完整的防盗链网站通用框架（留扩展点即可，当前只做少数派配置）

## 设计原则

**不是"少数派专用"，而是"通用防盗链下载服务 + 少数派配置"**

未来添加新网站（微信公众号、知乎等）只需在配置表中加一行，不需要改核心逻辑。

---

## P1-T1 - 添加 declarativeNetRequest 权限到 wxt.config.ts

### 涉及文件

- `webclipper/wxt.config.ts`

### 实现步骤

1. 在 `manifest.permissions` 数组中添加 `'declarativeNetRequestWithHostAccess'`
2. 在 `manifest.host_permissions` 数组中添加可能需要防盗链的域名 pattern：
   - `'https://cdnfile.sspai.com/*'`（当前需求）
   - 注意：这里只添加实际需要的域名，不要过度授权

### 验证

- 运行 `npm --prefix webclipper run build` 确认产物正常
- 检查生成的 `.output/chrome-mv3/manifest.json` 和 `.output/firefox-mv3/manifest.json` 包含新权限

### 提交要求

```
chore: P1-T1 - 添加 declarativeNetRequest 权限支持防盗链图片下载
```

---

## P1-T2 - 创建通用 image-download-proxy 图片下载辅助模块

### 涉及文件

- 新建 `webclipper/src/platform/webext/image-download-proxy.ts`

> 注意：测试文件在 P2-T2 中单独创建，本 task 只负责实现。

### 设计目标

提供一个**通用**的防盗链图片下载服务，支持：
1. 基于域名映射的自动 Referer 注入
2. Firefox 兼容（DNR 不可用时降级为普通 fetch）
3. 未来添加新网站只需在配置表中加一行

### 核心 API

```typescript
/**
 * 智能下载图片，自动处理防盗链 Referer
 * 
 * - 如果 URL 匹配防盗链规则，自动注入正确的 Referer
 * - Firefox 降级为普通 fetch（可能失败，但不会崩溃）
 * 
 * @example
 * const result = await downloadImageSmart({
 *   url: 'https://cdnfile.sspai.com/xxx.jpg',
 *   maxBytes: 2_000_000,
 * });
 */
export async function downloadImageSmart(input: {
  url: string;
  maxBytes?: number;
}): Promise<{ ok: true; blob: Blob; byteSize: number; contentType: string } | { ok: false; reason: string }>;
```

### 实现步骤

#### 1. 防盗链网站 Referer 映射表

```typescript
/**
 * 防盗链网站的 Referer 映射
 * Key: CDN 域名（exact match）
 * Value: 需要注入的 Referer
 * 
 * 未来添加新网站只需在此加一行
 */
const ANTI_HOTLINK_REFERER_MAP: Record<string, string> = {
  'cdnfile.sspai.com': 'https://sspai.com/',
  // 未来可扩展：
  // 'mmbiz.qpic.cn': 'https://mp.weixin.qq.com/',  // 微信公众号
  // 'picx.zhimg.com': 'https://www.zhihu.com/',    // 知乎
};

/**
 * 检查 URL 是否需要防盗链处理，返回对应的 Referer
 */
function getAntiHotlinkReferer(url: string): string | null {
  try {
    const { hostname } = new URL(url);
    return ANTI_HOTLINK_REFERER_MAP[hostname] || null;
  } catch {
    return null;
  }
}
```

#### 2. Feature-detect DNR 支持

```typescript
function isDnrSupported(): boolean {
  return !!(
    (globalThis as any).chrome?.declarativeNetRequest?.updateSessionRules
  );
}
```

#### 3. DNR 规则注册/清理（Firefox 兼容）

> ⚠️ **重要**：Chrome DNR 的 `urlFilter` 使用 filter list syntax，不是精确字符串匹配。
> - `urlFilter: 'https://example.com/'` 匹配 URL 中包含该子串的所有请求
> - `urlFilter: '|https://example.com/'` 匹配以该字符串开头的 URL
> - 我们需要使用 `|` 前缀匹配域名开头（包括 query string）

```typescript
async function registerRefererRule(ruleId: number, url: string, referer: string): Promise<void> {
  const isChrome = (globalThis as any).chrome?.runtime?.id != null;
  
  // 提取域名用于 urlFilter
  let domainPrefix: string;
  try {
    const { origin } = new URL(url);
    domainPrefix = `|${origin}/`;  // 例如: '|https://cdnfile.sspai.com/'
  } catch {
    domainPrefix = `|${url}`;  // fallback: 使用完整 URL
  }
  
  const condition: chrome.declarativeNetRequest.RuleCondition = {
    urlFilter: domainPrefix,  // ⚠️ filter list syntax: 匹配该域名下所有 URL
    resourceTypes: ['xmlhttprequest'],
  };
  
  // initiatorDomains 仅 Chrome 120+ 支持，Firefox 不支持
  if (isChrome && chrome.runtime?.id) {
    (condition as any).initiatorDomains = [chrome.runtime.id];
  }
  
  await chrome.declarativeNetRequest.updateSessionRules({
    addRules: [{
      id: ruleId,  // ⚠️ 必须是正整数！
      priority: 1,
      action: {
        type: 'modifyHeaders',
        requestHeaders: [{
          header: 'Referer',
          operation: 'set',
          value: referer,
        }],
      },
      condition,
    }],
  });
}

async function removeRefererRule(ruleId: number): Promise<void> {
  await chrome.declarativeNetRequest.updateSessionRules({
    removeRuleIds: [ruleId],
  });
}
```

> 💡 **性能说明**：当前设计每次请求注册/清理独立规则。如果一篇文章有 N 张防盗链图片，会执行 N 次注册/清理。由于 `updateSessionRules` 是异步的，性能影响可接受。未来可优化为"一次注册域名规则，所有图片共享"。

#### 4. 普通下载（公共逻辑）

> 注意：`parseContentType` 函数需要实现。可以复制 `image-inline.ts` 中的同名函数，或提取为公共工具函数。

```typescript
function parseContentType(value: unknown): string {
  const raw = String(value || '').trim();
  if (!raw) return '';
  return raw.split(';')[0]!.trim().toLowerCase();
}

async function downloadWithPlainFetch(url: string, maxBytes: number) {
  try {
    const res = await fetch(url, {
      method: 'GET',
      credentials: 'include',
      redirect: 'follow',
    });
    if (!res.ok) return { ok: false, reason: 'http' } as const;

    const contentType = res.headers.get('content-type') || '';
    if (!contentType.startsWith('image/')) {
      return { ok: false, reason: 'non_image' } as const;
    }

    const blob = await res.blob();
    const byteSize = blob.size || 0;
    if (!byteSize) return { ok: false, reason: 'empty' } as const;
    if (byteSize > maxBytes) return { ok: false, reason: 'too_large' } as const;

    return { ok: true, blob, byteSize, contentType } as const;
  } catch (_e) {
    return { ok: false, reason: 'fetch' } as const;
  }
}
```

#### 5. 主函数：智能下载

```typescript
export async function downloadImageSmart(input: { url: string; maxBytes?: number }) {
  const safeUrl = String(input.url || '').trim();
  const maxBytes = Number(input.maxBytes) || 2_000_000;
  
  if (!safeUrl) {
    return { ok: false, reason: 'invalid_input' } as const;
  }
  
  // 检查是否需要防盗链处理
  const referer = getAntiHotlinkReferer(safeUrl);
  
  // 不需要防盗链处理，直接普通下载
  if (!referer) {
    return downloadWithPlainFetch(safeUrl, maxBytes);
  }
  
  // 需要防盗链处理
  // Firefox 降级：DNR 不可用时使用普通 fetch
  if (!isDnrSupported()) {
    console.warn('[image-download-proxy] DNR not supported (Firefox?), falling back to plain fetch', { 
      url: safeUrl,
      expectedReferer: referer,
    });
    return downloadWithPlainFetch(safeUrl, maxBytes);
  }
  
  // 注册临时 DNR 规则注入 Referer
  const ruleId = Math.floor(Math.random() * 1000000) + 1;  // ⚠️ 正整数！
  
  try {
    await registerRefererRule(ruleId, safeUrl, referer);
    return await downloadWithPlainFetch(safeUrl, maxBytes);
  } finally {
    // 无论成功/失败都清理规则
    removeRefererRule(ruleId).catch((e) => {
      console.warn('[image-download-proxy] cleanup rule failed', { ruleId, error: String(e) });
    });
  }
}
```

#### 6. 并发安全说明

每次调用 `downloadImageSmart` 都会生成**唯一的 ruleId**（随机正整数），因此多个并发下载不会互相干扰。规则生命周期严格绑定到单次下载调用（`try/finally` 清理）。

### 验证

- 运行 `npm --prefix webclipper run compile` 确认类型正确
- 手动测试：调用 `downloadImageSmart` 下载少数派图片，确认成功

### 提交要求

```
feat: P1-T2 - 创建通用 image-download-proxy 防盗链图片下载辅助模块
```

---

## P1-T3 - 扩展 imageSanitizer 类型支持 stripQuerySuffix

### 涉及文件

- 修改 `webclipper/src/collectors/web/article-fetch-sites/site-spec.ts`（添加类型）
- 修改 `webclipper/src/collectors/web/article-extract/url.ts`（实现 sanitizer 逻辑）

### 实现步骤

#### 1. 扩展 `ArticleFetchImageSanitizer` 类型

在 `site-spec.ts` 中添加 `'stripQuerySuffix'` 选项：

```typescript
export type ArticleFetchImageSanitizer = 'none' | 'stripAtSuffix' | 'stripBangSuffix' | 'stripQuerySuffix';
```

#### 2. 实现 `stripQuerySuffix` 逻辑

在 `url.ts` 的 `sanitizeSiteImageUrl` 函数中添加分支：

```typescript
if (sanitizer === 'stripQuerySuffix') {
  try {
    const url = new URL(resolved);
    url.search = '';  // 清空所有查询参数
    return url.toString();
  } catch (_e) {
    // fallback: 简单字符串截断
    const qAt = resolved.indexOf('?');
    return qAt > -1 ? resolved.slice(0, qAt) : resolved;
  }
}
```

### 验证

- 运行 `npm --prefix webclipper run compile` 确认类型正确
- 单元测试验证：`sanitizeSiteImageUrl('https://example.com/img.jpg?foo=bar', 'https://example.com', 'stripQuerySuffix')` 返回 `'https://example.com/img.jpg'`

### 提交要求

```
feat: P1-T3 - 扩展 imageSanitizer 支持 stripQuerySuffix
```

---

## Phase 1 审计要求（audit-p1.md）

Phase 1 完成后，需要审计：

1. **权限最小化**：确认只添加了必要的权限，没有过度授权
2. **通用性**：确认 `ANTI_HOTLINK_REFERER_MAP` 设计支持未来扩展，新网站只需加配置
3. **规则安全性**：确认 DNR 规则仅作用于配置的域名，不影响其他网站
4. **错误处理**：确认 DNR API 调用失败时有合理的降级策略（Firefox 不崩溃）
5. **代码风格**：确认新文件符合项目 TypeScript 规范和分层约定
6. **ruleId 类型**：确认所有 `ruleId` 都是正整数，不是字符串或小数
7. **Firefox 兼容**：确认 Firefox MV3 构建不报错（DNR 降级路径正确）
8. **DNR urlFilter 格式**：确认使用 `|${origin}/` filter list syntax，不是完整 URL 精确匹配
9. **chrome.runtime.id 处理**：确认 `initiatorDomains` 仅在 `chrome.runtime.id` 存在时设置
10. **性能注释**：确认代码中有多图并发场景的性能说明注释
