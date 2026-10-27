# 客户端身份与 Client ID 接入

除公开接口和自举接口外，`/api/*`、`/loser/*` 的动态请求都需要证明具体客户端应用与安装实例的身份。
仅携带 `X-Neko-Client` 或公开的 `client_id` 不能代表请求可信：客户端还必须持有本实例私钥、短期令牌，
并对每个请求签名。

> 本文说明客户端必须实现的公开契约。审核标准、具体限流阈值、内部存储和风控判定属于服务端实现，
> **不对外公开且会随时调整**。请勿依赖文档未列出的行为。

---

## 1. 申请与分配 Client ID

### 1.1 官方客户端

官方 Android、PC、Web 及手表客户端由服务端预先登记，无需开发者单独申请。客户端升级时应继续使用
既有 `client_id`，不得改用其他平台的 ID。

### 1.2 第三方客户端

第三方客户端目前通过人工审核接入，**没有公开的自助申请 API**。请发送邮件至
`support@nekocore.cn`，主题注明「NekoMusic Client ID 申请」。或者前往官方qq群，并提供：

| 信息 | 说明 |
|------|------|
| 应用名称 | 用户可见名称 |
| 归属方 | 个人或组织名称 |
| 目标平台 | 例如 `web`、`android`、`androidwatch`、`pc`；一个 Client ID 只绑定一个平台 |
| 应用主页 / 仓库 | 用于确认应用来源与用途 |
| 预计接入方式 | Web 请提供实际 Origin；Android 请提供包名与签名证书摘要（如适用） |
| 客户端类型 | 公开客户端或机密客户端，说明密钥将保存在哪里 |
| 联系方式 | 便于审核、轮换凭据和吊销实例 |

审核通过后，管理员会分配：

- `client_id`：应用公开标识，可以随客户端发布，但不能跨平台使用，也不等同于访问凭据。
- `client_secret`：仅机密客户端分配。只在首次创建或轮换时展示一次，不得提交到公开仓库、
  写入日志或下发给普通用户。浏览器、公开分发的桌面 / 移动二进制无法可靠保守该值，应采用公开客户端
  加每实例私钥签名的方案。

同一应用的每个安装实例都应有独立的 EC 私钥、`instance_id` 和短期令牌；不要在所有安装之间复制私钥。

### 1.3 平台绑定

申请时确定的平台会与后续所有环节校验：

- `GET /api/client/challenge` 的 `platform`；
- `X-Neko-Client: <platform>+<version>` 中的平台；
- 注册 / 续签请求体中的 `platform`；
- 每次签名请求里的平台字段。

以上平台必须完全一致。把 Android 的 Client ID 用于 PC、Web 或其它平台会收到 `401`。

---

## 2. 实例密钥与编码约定

每个安装实例首次启动时生成一对 **EC P-256** 密钥：

- 私钥只保存在本机安全存储中（例如 Android Keystore、WebCrypto、系统凭据库），不得上传。
- 公钥按 **SPKI DER** 编码后使用无填充 base64url 表示，放入注册请求的 `public_key`。
- 签名算法为 **SHA256withECDSA**，签名结果按 DER 编码后再使用无填充 base64url 表示。
- WebCrypto 默认产生的 P1363（`r || s`，64 字节）签名需先转换为 DER，再提交给服务端。
- `instance_id` 使用 UUID；一个实例 ID 只能绑定一个 Client ID、平台和公钥。
- `timestamp` 使用 Unix 秒。客户端应保持系统时钟同步；时钟偏差过大时身份校验会失败。
- 所有签名串均按 UTF-8 编码，字段之间的换行必须是单个 `\n`，最后一行之后不再追加换行。

---

## 3. 获取一次性挑战

注册实例和续签令牌前都必须先获取挑战。挑战一次有效，并绑定 Client ID、平台、来源与当前请求的
`User-Agent`；换 IP、换 UA 或重复使用都会失败。

**端点:** `GET /api/client/challenge`

**认证:** 无需用户登录，无需实例令牌，也无需 `X-Neko-Nonce`。

**请求头:**

```text
User-Agent: <客户端 User-Agent>
X-Neko-Client: <platform>+<version>
```

**查询参数:**

| 参数 | 必填 | 说明 |
|------|------|------|
| `client_id` | 是 | 已分配的 Client ID |
| `platform` | 是 | Client ID 绑定的平台，小写 |
| `purpose` | 否 | `register`（默认）或 `token`，必须与实际用途一致 |

**响应示例（成功）:**

```json
{
  "success": true,
  "message": "",
  "data": {
    "challenge": "<一次性挑战>",
    "expiresIn": 300,
    "purpose": "register",
    "protocol": "neko-client-v1"
  }
}
```

`expiresIn` 以服务端本次返回值为准，不要写死。挑战无论注册成功与否，使用一次后都会失效。

**状态码:**

| 状态码 | 含义 |
|--------|------|
| `200` | 挑战签发成功 |
| `401` | Client ID、平台、UA 或自报客户端不匹配，或应用已停用 |
| `429` | 请求过于频繁，按 `Retry-After` 退避 |
| `503` | 暂时无法签发挑战，稍后重试 |

---

## 4. 注册安装实例

**端点:** `POST /api/client/register`

**认证:** 无需用户登录；不需要实例令牌，也不需要 `X-Neko-Nonce`。

**请求头:**

```text
Content-Type: application/json
User-Agent: <客户端 User-Agent>
X-Neko-Client: <platform>+<version>
```

机密客户端还需携带：

```text
X-Neko-Client-Secret: <client_secret>
```

**请求体字段:**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `client_id` | string | 是 | 已分配的 Client ID |
| `platform` | string | 是 | 与 Client ID、`X-Neko-Client` 一致 |
| `instance_id` | string | 是 | 新安装实例的 UUID |
| `public_key` | string | 是 | EC P-256 公钥 SPKI DER 的 base64url |
| `challenge` | string | 是 | 上一步 `purpose=register` 的挑战 |
| `timestamp` | number | 是 | 当前 Unix 秒 |
| `signature` | string | 是 | 注册签名串的 SHA256withECDSA DER 签名，base64url |
| `origin` | string | 否 | Web 客户端应填写实际 Origin；与请求头 `Origin` 同时存在时必须一致 |
| `attestation.chain` | string[] | 条件必填 | 仅当该 Client ID 被要求提供平台证明时填写 |

**请求体示例:**

```json
{
  "client_id": "nkc_pub_example",
  "platform": "web",
  "instance_id": "550e8400-e29b-41d4-a716-446655440000",
  "public_key": "<SPKI DER base64url>",
  "challenge": "<一次性挑战>",
  "timestamp": 1790000000,
  "signature": "<DER signature base64url>",
  "origin": "https://client.example.com"
}
```

**注册签名串（逐字节一致）:**

```text
neko-client-register-v1
<client_id>
<platform>
<instance_id>
<public_key>
<challenge>
<timestamp>
```

**响应示例（成功）:**

```json
{
  "success": true,
  "message": "",
  "data": {
    "instance_id": "550e8400-e29b-41d4-a716-446655440000",
    "token": "<实例短期令牌>",
    "expires_at": 1790604800,
    "attestation": "none",
    "protocol": "neko-client-v1"
  }
}
```

- `token` 只用于客户端身份校验，不是用户登录 Token；需要用户数据的接口仍要另带 `Authorization`。
- `expires_at` 为 Unix 秒。到期前应使用续签接口换新令牌，不要重新注册同一实例。
- 同一 `instance_id` 重复注册只有在 Client ID、平台和公钥均一致时才可能成功。

**状态码:**

| 状态码 | 含义 |
|--------|------|
| `200` | 注册成功 |
| `401` | 参数、平台、挑战、签名、来源或机密客户端 secret 校验失败，或实例已被吊销 |
| `503` | 客户端身份服务暂时不可用 |

---

## 5. 续签实例令牌

**端点:** `POST /api/client/token`

**认证:** 无需用户登录；使用实例私钥证明身份，不需要 `X-Neko-Nonce`。

**请求头:**

```text
Content-Type: application/json
User-Agent: <客户端 User-Agent>
X-Neko-Client: <platform>+<version>
```

**请求体字段:**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `client_id` | string | 是 | 与注册时一致 |
| `platform` | string | 是 | 与注册时一致 |
| `instance_id` | string | 是 | 已注册实例 UUID |
| `challenge` | string | 是 | `purpose=token` 的挑战 |
| `timestamp` | number | 是 | 当前 Unix 秒 |
| `signature` | string | 是 | 续签签名串的 SHA256withECDSA DER 签名，base64url |

**续签签名串（逐字节一致）:**

```text
neko-client-token-v1
<client_id>
<platform>
<instance_id>
<challenge>
<timestamp>
```

**响应示例（成功）:**

```json
{
  "success": true,
  "message": "",
  "data": {
    "token": "<新的实例短期令牌>",
    "expires_at": 1790604800,
    "protocol": "neko-client-v1"
  }
}
```

**状态码:**

| 状态码 | 含义 |
|--------|------|
| `200` | 续签成功 |
| `401` | 实例不存在 / 已吊销、平台不符、挑战无效、时间戳或签名错误 |
| `503` | 客户端身份服务暂时不可用 |

---

## 6. 为每个动态请求签名

除第 7 节的豁免接口外，每个 `/api/*`、`/loser/*` 请求都必须同时携带以下请求头：

| 请求头 | 说明 |
|--------|------|
| `User-Agent` | 非空；应与注册时保持同一客户端类型 |
| `X-Neko-Client` | `<platform>+<version>` |
| `X-Neko-Client-Id` | 应用 Client ID |
| `X-Neko-Client-Instance` | 安装实例 UUID |
| `X-Neko-Client-Token` | 注册 / 续签得到的实例令牌 |
| `X-Neko-Client-Timestamp` | 当前 Unix 秒 |
| `X-Neko-Client-Signature` | 本请求签名串的 SHA256withECDSA DER 签名，base64url |
| `X-Neko-Nonce` | 按《请求防重放》领取的一次性 nonce |

如需访问用户数据，同时携带 `Authorization: <用户 Token>`。实例令牌与用户 Token 缺一不可，
二者含义不同。

### 6.1 计算请求体哈希 `body_hash`

- JSON 或其它原始请求体：对实际发送的原始字节做 SHA-256，结果为小写十六进制。
- 无请求体的 `GET` / `HEAD` / `DELETE` 等：使用空字节的 SHA-256，即
  `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`。
- `multipart/form-data` 上传：固定使用 `-`，签名不绑定上传体。

### 6.2 构造请求签名串

```text
neko-client-v1
<METHOD>
<PATH>
<RAW_QUERY>
<client_id>
<instance_id>
<platform>
<timestamp>
<body_hash>
<nonce>
```

- `METHOD` 必须大写。
- `PATH` 是不含查询串的请求路径，例如 `/api/music/search`。
- `RAW_QUERY` 是原样查询串，不含开头的 `?`；没有查询串时留空。参数顺序与百分号编码都必须与
  实际发送的 URL 完全一致。
- `timestamp` 必须与 `X-Neko-Client-Timestamp` 完全相同。
- `nonce` 必须与 `X-Neko-Nonce` 完全相同；应先领取 nonce，再构造签名并发送请求。
- 每一行都以单个 `\n` 分隔，最后一行之后不要再加换行。

**示例（仅示意字段顺序）:**

```text
neko-client-v1
GET
/api/music/search
q=%E6%99%B4%E5%A4%A9&page=1
nkc_pub_example
550e8400-e29b-41d4-a716-446655440000
web
1790000000
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
<nonce>
```

签名失败、令牌过期、实例吊销、平台不符或时钟偏差过大时，服务端返回 `401`：

```json
{
  "success": false,
  "message": "客户端身份无效，请重启或更新客户端",
  "data": null
}
```

此时响应头会带 `X-Neko-Client-Status: invalid` 与 `WWW-Authenticate: NekoClient`。客户端应：

1. 比对本地 `expires_at`，令牌临近到期时先调用 `POST /api/client/token` 续签；
2. 仍失败时重新完成注册流程，确认 Client ID、平台、实例 ID 与私钥匹配；
3. 不要用同一个时间戳 / 签名盲重试；重试必须重新签名。

收到 `409` 表示 nonce 相关失败，应按《请求防重放》重新换题、领取 nonce、重新签名后重试一次。

---

## 7. 无需实例身份的接口

以下接口不校验实例令牌与请求签名：

| 路径 / 类型 | 说明 |
|-------------|------|
| `/api/client/challenge`、`/api/client/register`、`/api/client/token` | 实例身份自举 |
| `/api/replay/challenge`、`/api/replay/nonce` | 防重放 nonce 自举 |
| `/api/music/latest`、`/api/music/ranking` | 公开且允许缓存的音乐列表 |
| `/api/music/cover/*`、`/api/user/avatar/*` | 浏览器原生图片请求，无法附加请求头 |
| `/api/user/qrlogin/status`、`/api/user/notifications/stream` | SSE 长连接 |
| `/loser/*/pull` | 歌单导入进度 SSE |
| `/api/payment/zpay/notify` | 支付平台服务器回调 |
| `OPTIONS`、`HEAD` | 预检 / 元信息请求 |
| 静态资源、媒体文件、安装包 | 不属于动态接口，缓存与下载语义不变 |

豁免接口仍受其它安全策略约束；例如公开接口仍可能需要用户登录，SSE 仍需要有效的用户 Token。

---

## 8. 状态码汇总

| 状态码 | 客户端处理 |
|--------|------------|
| `200` | 按响应体 `success` 处理业务结果 |
| `401` | 客户端身份或登录态无效；先检查 `X-Neko-Client-Status`，再按第 6 节续签或重新注册 |
| `409` | nonce 缺失 / 过期 / 重放；重新领取 nonce 并重新签名后重试一次 |
| `413` | 请求体超过接口限制 |
| `429` | 请求过于频繁；按 `Retry-After` 退避 |
| `503` | 身份或 nonce 服务暂时不可用；指数退避后重试 |
