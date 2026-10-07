# 请求防重放（专项）

所有动态接口（`/api/*`、`/loser/*`）都需要携带一次性请求头 `X-Neko-Nonce`；同一个请求重复发送
（重放）会被拒绝。

> 相关文档：[主 API 文档](README.md) 的「防重放 nonce（所有动态接口）」「3. 获取音乐文件」。

---

## 1. 概述

- 每个受保护请求都需要一个**尚未使用过**的 nonce，通过请求头发送，用后即废。
- nonce 分**读**、**写**两类：`GET` 用读类别，`POST` / `PUT` / `PATCH` / `DELETE` 用写类别，不能混用。
- nonce 默认 **120 秒**有效，过期作废，需要重新领取。
- nonce 与领取它的客户端 IP 绑定，换 IP 使用会被拒绝。
- 消费只能成功一次：重放同一个请求会返回 `409`。

---

## 2. 领取 nonce

**端点:** `GET /api/replay/nonce`

**认证:** 无需登录

**查询参数:**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `read` | int | `16` | 需要领取的读类别 nonce 数量；`0` 表示不领，上限 `64` |
| `write` | int | `16` | 需要领取的写类别 nonce 数量；`0` 表示不领，上限 `64` |

**响应示例（成功）:**

```json
{
  "success": true,
  "message": "",
  "data": {
    "nonces": {
      "read":  ["3f2a9c1d5e7b04a8c2d6f1093b5e7a4c", "9c01d7b3f52a48e6b0c9d1f7a3e5820b"],
      "write": ["a71b3e95c40d728f16b9a3e5c7082d41", "de4420b7c9813fa56d0e2b4c7a91f385"]
    },
    "expiresIn": 120
  }
}
```

**响应示例（失败）:**

```json
{
  "success": false,
  "message": "服务繁忙，请稍后重试"
}
```

**状态码:**

| 状态码 | 说明 |
|--------|------|
| 200 | 领取成功 |
| 503 | 暂时无法签发 nonce，请稍后重试 |

---

## 3. 携带 nonce 发起请求

**请求头:**

```
X-Neko-Nonce: <从领取接口得到的 nonce>
```

**示例:**

```text
GET /api/music/file/1
X-Neko-Nonce: 3f2a9c1d5e7b04a8c2d6f1093b5e7a4c
```

**校验结果:**

| 情况 | 结果 |
|------|------|
| nonce 有效且未被使用 | 正常处理 |
| 缺少 `X-Neko-Nonce` | `409` + `X-Neko-Replay-Status: missing` |
| nonce 已被使用 / 已过期 / 类别不符 / IP 不符 | `409` + `X-Neko-Replay-Status: invalid` |
| 暂时无法校验 nonce | 不阻塞请求，客户端无需特殊处理 |

**响应示例（失败）:**

```json
{
  "success": false,
  "message": "请求已失效，请刷新后重试"
}
```

**客户端处理建议:** 收到 `409` 且 `X-Neko-Replay-Status` 为 `missing` / `invalid` 时，重新领取一个
nonce 并重试一次即可（被拒的请求不会被执行，重试不会产生重复副作用）。

---

## 4. 无需 nonce 的接口

| 路径 / 类型 | 说明 |
|-------------|------|
| `/api/replay/nonce` | nonce 领取接口本身 |
| `/api/music/latest`、`/api/music/ranking` | 公开数据，重复获取无副作用 |
| `/api/music/cover/*` | 封面图片，由浏览器原生图片请求加载，无法附带请求头 |
| `/api/user/avatar/*` | 用户头像，同上 |
| `/api/user/qrlogin/status` | 扫码登录状态流（SSE 长连接，浏览器会自动重连） |
| `/loser/*/pull` | 歌单导入进度流（SSE） |
| `/api/payment/zpay/notify` | 支付平台服务器回调，无法附带自定义请求头 |
| `multipart/form-data` 请求 | 文件上传 |
| `OPTIONS`、`HEAD` | 预检 / 元信息请求 |

---

## 5. 客户端接入

### Web

网页端已内置自动处理：业务代码照常发起 `fetch` 请求即可，无需手动传 nonce。

### PC / Android

按以下步骤接入即可，最低要求是**每个受保护请求使用一个未使用过的 nonce**：

1. 启动时（或池内数量不足时）调用 `GET /api/replay/nonce`，缓存 `read` / `write` 两个列表；
2. 发起 `GET` 时从 `read` 取一个，发起写方法时从 `write` 取一个，放进 `X-Neko-Nonce` 头；
3. 收到 `409`（`missing` / `invalid`）时重新领取一个再重试一次；
4. 白名单接口与 `multipart` 上传无需携带。

> **兼容提示**：`GET /api/music/file/{id}` 已由 `302` 重定向改为 `200` + JSON `data.url`
> （详见主文档「3. 获取音乐文件」），客户端需同步适配。该接口需要 `X-Neko-Nonce`。

---

## 6. 状态码汇总

| 状态码 | 说明 |
|--------|------|
| 200 | 正常处理 |
| 409 | 缺少 / 重放 / 过期 / 类别或 IP 不符的 nonce，可通过 `X-Neko-Replay-Status` 区分 |
| 503 | nonce 领取暂时不可用，请稍后重试 |
