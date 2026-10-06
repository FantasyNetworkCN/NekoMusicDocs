# 防爬与客户端识别

本文档描述 Neko歌姬计划后端对 JSON 接口（`/api/*`）的**防爬与客户端区分拦截**策略：
如何拦截已知爬虫/安全扫描器，以及如何区分「未知 / 小众网站爬虫」与「真实浏览器、应用内 WebView、原生客户端」。
所有接口的通用响应契约见主文档 [README.md](README.md)。

> 目标：主要拦截**未知网站、小众网站**的爬虫与探测工具——这类 UA 往往不含常见关键词（`bot`/`crawler`），
> 甚至只伪造一个 `Mozilla/5.0` 前缀。仅靠关键词黑名单会大面积漏网，因此引入「浏览器完整性」正向校验。

---

## 1. 拦截模型（两段式）

对 `/api/*` 的每个请求，按下列顺序判定：

| 顺序 | 判定 | 命中动作 |
| --- | --- | --- |
| 0 | 路径为 `/api/payment/zpay/notify` | **直接放行**（支付平台回调，豁免防爬与限流） |
| 0.5 | **User-Agent 为空** | **直接放行**（NekoMusic PC 等 Qt 桌面端默认不发送 UA，交 IP 限流兜底；有 UA 的未知爬虫仍走下方拦截） |
| 1 | UA 命中**已知爬虫 / 无头 / 命令行 / 安全扫描器**关键词 | **直出 SEO**（GET/HEAD → 200 HTML；其它方法 → 403） |
| 2 | UA 命中**放行名单**（`network.allow_client_user_agents`） | 放行 |
| 3 | UA 为**内置原生客户端**（Android okhttp/Dalvik、PC NekoMusic/Qt/Electron、播放器 libmpv/VLC/FFmpeg 等） | 放行 |
| 4 | UA **结构像真浏览器** 且请求带**浏览器特征头** | 放行 |
| 5 | 其余（未知爬虫、残缺/仅伪造 UA、扫描器） | **直出 SEO**（GET/HEAD → 200 HTML；其它方法 → 403） |

第 2~5 步属于**浏览器完整性区分拦截**，可用 `network.browser_integrity_enabled=false` 关闭（关闭后退化为仅第 1 步黑名单）。

> **「直出 SEO」含义**：爬虫访问 `/api/*` 时不再直接 403，也**不做 302 跳转**，而是 `GET`/`HEAD`
> **直接返回 `200` + 对应 SEO 页的服务端 HTML**（服务端内部 forward，URL 不变），让抓取器一次拿到可索引内容；
> `POST`/`PUT` 等没有对应 SEO 页的方法仍返回 `403`。
> 页面路由（首页 / 下载 / 关于 / 隐私 / 排行榜 / 最新 / 搜索、`/detail/{id}`、`/playlist/{id}`）
> 同样使用本判定：**爬虫（含伪造浏览器 UA 但缺特征头者）返回 SEO HTML，真浏览器返回前端 SPA**。

### 1.1 已知爬虫 / 扫描器关键词（第 1 步）

覆盖搜索引擎、AI 抓取、链接预览、SEO 分析、命令行客户端、无头浏览器、监控归档，以及常见安全扫描器
（`sqlmap`、`nikto`、`nmap`、`masscan`、`zgrab`、`nuclei`、`wpscan`、`gobuster`、`ffuf`、`feroxbuster`、
`acunetix`、`nessus`、`openvas`、`burpsuite`、`zaproxy`、`whatweb`、`w3af`、`arachni`、`skipfish`、
`jaeles`、`commix`、`dalfox`、`wapiti`、`netsparker`、`sslscan`、`sslyze`、`testssl` 等）。

### 1.2 浏览器结构判定（第 4 步其一）

UA 必须同时满足：

- 含 `Mozilla/`；
- 含浏览器内核标记之一：`AppleWebKit/`（Chrome / Edge / Opera / Safari / iOS·Android WebView）、
  `Gecko/`（Firefox）、`Trident/`（旧 Edge / IE11）。

> 注意：iOS `WKWebView`（微信、QQ、支付宝等应用内浏览器）的 UA **常省略 `Safari/`、`Version/`**，
> 因此本校验**不强制** Safari 版本串，只要求存在内核标记，避免误伤应用内浏览器。

**会被判定为「非浏览器」的典型 UA**（即使含 `Mozilla/`）：

- `Mozilla/5.0`（裸前缀）
- `Mozilla/5.0 (X11; Linux x86_64)`（无内核标记）
- `Mozilla/5.0 (compatible; AcmeIndex/1.0)`（未知爬虫）
- `Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)`（无内核标记）

### 1.3 浏览器特征头（第 4 步其二）

请求必须带 `Accept`，且带 `Accept-Language` 或任一 `Sec-Fetch-*`（`Sec-Fetch-Mode` / `Sec-Fetch-Site` / `Sec-Fetch-Dest`）。
真实浏览器（含媒体 `<audio>` 请求，通常带 `Accept: */*` + `Accept-Language` + `Sec-Fetch-Dest: audio`）恒满足。

> 仅伪造浏览器 UA、但不带浏览器头的脚本会被第 4 步拒绝。
> 说明：请求头可被有能力的攻击者一并伪造，本策略用于显著提高门槛、拦截绝大多数未知/小众爬虫；对高度定制的爬虫属于持续对抗。

### 1.4 内置原生客户端白名单（第 3 步）

UA 含下列标记之一即放行：`okhttp`、`dalvik`、`libmpv`、`mpv/`、`vlc`（前缀）、`ffmpeg`、`ffprobe`、
`android`、`electron`、`qt`（前缀）、`qts`、`qtwebengine`、`nekomusic`（本站 PC 桌面端，含封面 UA `NekoMusic Qt`）。

---

## 2. 命中响应

**直出 SEO（GET / HEAD）** —— 返回 `200 OK` + 对应 SEO 页 HTML（`Content-Type: text/html;charset=utf-8`），
不做 302 跳转；同一 URL 对爬虫与浏览器表现不同，因此固定带 `Vary: User-Agent` 与 `Cache-Control: private, no-store`：

| 被访问的 `/api` 路径 | 直出的 SEO 页面 |
| --- | --- |
| `/api/music/ranking` | `/ranking` |
| `/api/music/latest` | `/latest` |
| `/api/music/search` | `/search` |
| `/api/music/info/{id}`、`cover/{id}`、`file/{id}`、`lyrics/{id}` | `/detail/{id}` |
| 其它 | `/`（首页） |

直出的 HTML 自带 `<link rel="canonical">`（例如 `/api/music/info/42` 的 canonical 指向 `/detail/42`），
因此与正式页面不构成重复内容，搜索引擎会把权重归到正式页。

**非 GET / HEAD 方法** —— 没有对应 SEO 页面，返回 **403**，响应体与全局错误契约一致：

```json
{
  "success": false,
  "message": "请求已拒绝",
  "data": null
}
```

响应头固定为 `Cache-Control: private, no-store`。

---

## 3. 配置项

见 `backend/src/main/resources/config.yml` 的 `network` 段：

```yaml
network:
  # 可信客户端 IP 头（影响评论归属地与 IP 限流；详见主文档与部署说明）
  #   auto        —— 默认。自适应多家 CDN：依次尝试 CF-Connecting-IP / True-Client-IP /
  #                  Ali-CDN-Real-IP / X-Edge-Real-IP / Fastly-Client-IP / X-Client-IP /
  #                  X-Azure-ClientIP，再退到 X-Forwarded-For（自动摘掉本机 Nginx 追加的回源那一跳）、
  #                  X-Real-IP，最后退到 socket 对端。换 CDN 不用改配置、不用维护回源 IP 段。
  #   X-Real-IP   —— 只信该头（Nginx 用 proxy_set_header 覆盖写入，客户端无法伪造）。
  #   direct / 留空 —— 忽略一切转发头，只用 socket 对端，最严格。
  trusted_client_ip_header: auto

  # 仅 auto 模式生效：额外登记「优先尝试」的自定义客户端 IP 头名（内置清单已覆盖常见 CDN，
  # 换了清单之外的新 CDN 时在此加一行即可，无需改代码或发版）
  extra_client_ip_headers: []

  # 总开关：关闭后 /api 不再做任何防爬拦截
  crawler_protection_enabled: true

  # 浏览器完整性区分拦截（拦截未知/小众爬虫与安全扫描器）
  # false = 仅保留关键词黑名单
  browser_integrity_enabled: true

  # 额外放行的客户端 UA 子串（大小写不敏感），用于登记内置白名单之外的第三方客户端
  allow_client_user_agents: []
```

IP 频率限制见 `rate_limit` 段（按 /24 聚合计数与封锁，超限返回 429 与 `Retry-After`）。

### 3.1 客户端 IP 解析（多 CDN 自适应）

统一由 `util/ClientIpResolver` 解析，评论归属地（`/api/comments` 的 `ipRegion`）、IP 限流、歌词接口、
支付水皮下发、听歌识曲等共用同一来源。

- `auto`（默认）按顺序取第一个命中的头：
  1. `network.extra_client_ip_headers` 中登记的自定义头（可选，便于适配清单外的新 CDN）；
  2. CDN 专用单值头 `CF-Connecting-IP` → `True-Client-IP` → `Ali-CDN-Real-IP` → `X-Edge-Real-IP` →
     `Fastly-Client-IP` → `X-Client-IP` → `X-Azure-ClientIP` → `CloudFront-Viewer-Address`；
  3. `X-Forwarded-For`：先摘掉本机 Nginx 用 `$proxy_add_x_forwarded_for` 追加的一格回源地址
     （即与 `X-Real-IP` 相同的那一格），再取最右一格，避免取到 CDN 回源 IP；
  4. `X-Real-IP`；
  5. socket 对端地址。
- 安全边界：`auto` 只有在 socket 对端是回环 / 内网 / IPv6 ULA 地址（说明请求确实经过本机反向代理）
  时才信任上述转发头；后端被公网直连时一律使用 socket 对端地址，杜绝伪造归属地。
- 若源站可被公网直连、需要最严格的防伪造，请把该项改为 `X-Real-IP` 或 `direct`。

---

## 4. 客户端接入建议

- **浏览器 / Web 前端**：无需任何改动（真实浏览器天然满足结构与特征头校验）。
- **应用内 WebView（微信 / QQ / 支付宝等）**：无需改动（UA 含 `AppleWebKit/`，且带浏览器特征头）。
- **Android 客户端**：使用内置白名单内的 UA（如 `okhttp`、`dalvik`）即可。
- **NekoMusic PC（Qt）**：无需改动。`ApiClient` 用 `QNetworkRequest` 默认**不发送 User-Agent**，服务端对空 UA 直接放行；封面请求 UA `NekoMusic Qt` 已在原生白名单内。
- **其它 Qt / 桌面客户端**：若 UA 为空同样放行；若使用自定义 UA，请确保包含 `qt` 等内核标记，或用 `network.allow_client_user_agents` 登记。
- **第三方客户端 / 脚本集成**：若 UA 不在内置白名单，请使用带版本号的自定义 UA，并加入
  `network.allow_client_user_agents` 登记放行，例如：

  ```yaml
  allow_client_user_agents:
    - "MyThirdPartyClient/"
  ```

---

## 5. 与 SEO 抓取的关系

搜索引擎等需要被收录的抓取器**不应读取 JSON 接口**。当它们访问 `/api/*` 时，服务端会**直接返回**
对应服务端渲染页面的 HTML（`200`，详见第 2 节映射），拿到可索引内容而不是 JSON 或 SPA 壳；
不使用 `302` 跳转，避免把抓取配额与权重消耗在跳转上。
页面路由本身也遵循同一套爬虫判定（真浏览器 → SPA，爬虫 / 伪浏览器 UA → SEO HTML）。

---

## 6. 通用安全加固（方法限制与响应头）

除防爬外，后端对所有响应统一做以下加固（`SecurityHeadersFilter` + Jetty 连接器配置）：

| 项 | 策略 |
| --- | --- |
| `TRACE` / `TRACK` 方法 | 返回 `405 Method Not Allowed`，`Allow` 列出常规方法（防 XST） |
| `Server` 响应头 | 不发送带版本号的实现标识，统一为 `NekoMusic`（避免泄露 Jetty 版本） |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `X-Frame-Options` | `SAMEORIGIN` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` |
| `Strict-Transport-Security` | 由 Nginx / CDN 在 TLS 层下发（见 `deploy/nginx.cdn.conf`），后端明文 HTTP 不下发 |
