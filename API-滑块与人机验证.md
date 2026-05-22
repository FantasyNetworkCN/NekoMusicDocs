# 滑块人机验证与注册邮箱验证码 API

本文说明**注册场景**下：获取滑块挑战、校验换取通行令牌、发送邮箱验证码的接口。所有路径均相对部署根 URL（与主文档一致，例如 `https://music.cnmsb.xin`）。

**认证：** 以下接口均**无需**登录，无需 `Authorization` 头。

---

## 整体流程（客户端建议顺序）

1. 用户填写邮箱（及可选用户名）后，在界面中打开人机验证（例如弹窗）。
2. 调用 **`GET /api/captcha/slider`** 拉取当次挑战（背景图、滑块图、`captchaToken` 等）。
3. 用户拖动滑块对齐缺口后，在**松手**时调用 **`POST /api/captcha/slider/verify`**，提交本次的 `captchaToken` 与拼图水平位移 `captchaOffsetX`（像素，与背景图坐标系一致）。
4. 若校验成功，响应体中的 **`captchaPassToken`** 为短时一次性令牌；紧接着调用 **`POST /api/user/send-verification`**，在 JSON 中带上 **`captchaPassToken`**（及 `email`、`username`），服务端校验通过后才发邮件。
5. 用户收到邮件验证码后，再调用 **`POST /api/user/register`** 完成注册（**注册接口不要求**滑块参数，仅校验邮箱验证码等业务规则）。

**注意：**

- 每个 **`captchaToken`** 在调用一次 `slider/verify` 后即作废（无论对错）。
- 每个 **`captchaPassToken`** 仅可使用一次，用于一次 `send-verification`；过期或重复使用会失败。
- 服务端拼图水平容差为 **±5 像素**（与实现一致）。

**时效（实现常量，供联调参考）：**

| 对象 | 说明 |
|------|------|
| 挑战 `captchaToken` | 签发后约 **3 分钟**内有效；`verify` 会消费该 token |
| `captchaPassToken` | 校验通过后签发，约 **2 分钟**内须用于 `send-verification`，且一次性 |

---

## 1. 获取滑块挑战

**方法/路径：** `GET /api/captcha/slider`

**请求头：** 无强制要求；浏览器跨域时服务端已返回 `Access-Control-Allow-Origin: *`。

**请求体：** 无。

**成功响应** `200`，`Content-Type: application/json;charset=UTF-8`：

```json
{
  "success": true,
  "message": "ok",
  "data": {
    "captchaToken": "string",
    "bgImage": "data:image/png;base64,...",
    "sliderImage": "data:image/png;base64,...",
    "puzzleY": 12,
    "bgWidth": 300,
    "bgHeight": 180,
    "sliderWidth": 52,
    "sliderHeight": 52
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `captchaToken` | string | 本次挑战唯一标识，提交校验时原样传入 |
| `bgImage` | string | 背景图 Data URL（PNG） |
| `sliderImage` | string | 滑块拼图块 Data URL（PNG） |
| `puzzleY` | number | 拼图块在背景上的 **Y** 偏移（像素） |
| `bgWidth` / `bgHeight` | number | 背景图尺寸（当前实现固定为 300×180） |
| `sliderWidth` / `sliderHeight` | number | 拼图块尺寸（当前实现固定为 52×52） |

**失败示例：**

```json
{
  "success": false,
  "message": "生成验证码失败喵",
  "data": null
}
```

---

## 2. 校验滑块位移（换取 `captchaPassToken`）

**方法/路径：** `POST /api/captcha/slider/verify`

**请求头：**

```
Content-Type: application/json
```

**请求体（JSON）：**

```json
{
  "captchaToken": "string",
  "captchaOffsetX": 0
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `captchaToken` | string | 是 | 来自 `GET /api/captcha/slider` 的 `data.captchaToken` |
| `captchaOffsetX` | number | 是 | 用户对齐后，拼图块左边缘相对背景的 **X** 像素（整数） |

**成功响应：**

```json
{
  "success": true,
  "message": "验证通过喵",
  "data": {
    "captchaPassToken": "string"
  }
}
```

**失败响应（示例）：**

```json
{
  "success": false,
  "message": "拼图位置不正确或已失效，请换一张重试喵",
  "data": null
}
```

其他常见 `message`：`缺少 captchaToken 喵`、`缺少 captchaOffsetX 喵`、`请求体不能为空喵`、`服务器内部错误喵`。

---

## 3. 发送注册用邮箱验证码（消费 `captchaPassToken`）

**方法/路径：** `POST /api/user/send-verification`

**请求头：**

```
Content-Type: application/json
```

**请求体（JSON）：**

```json
{
  "email": "user@example.com",
  "username": "用户",
  "captchaPassToken": "string"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | 是 | 接收验证码的邮箱 |
| `username` | string | 否 | 邮件模板等用途，缺省服务端可按 `"用户"` 处理 |
| `captchaPassToken` | string | 是 | 来自 `POST /api/captcha/slider/verify` 成功响应的 `data.captchaPassToken` |

**成功响应：**

```json
{
  "success": true,
  "message": "验证码已发送至您的邮箱",
  "data": null
}
```

**失败响应（示例）：**

| 场景 | `success` | 说明 |
|------|-------------|------|
| 未带或空 `captchaPassToken` | false | `message` 如：`请先完成安全验证（滑动拼图）` |
| `captchaPassToken` 无效、过期或已使用 | false | `message` 如：`安全验证已失效或已使用，请重新滑动验证` |
| 邮箱格式错误 / 非白名单域名等 | false | 以服务端返回 `message` 为准 |
| 发送频率限制 | false | HTTP **429**，响应头 `Retry-After` 为建议等待秒数；JSON `data` 中含 `retryAfterSec`（整数秒） |

---

## 4. 用户注册（不要求滑块）

**方法/路径：** `POST /api/user/register`

注册**不再**提交滑块相关字段；仅需用户名、密码、邮箱与**邮件里的验证码**。详见主文档 [用户相关 API](README.md#用户相关-api) 中「用户注册」一节。

---

## 5. 跨域与 OPTIONS

上述接口在 Servlet 中设置了 `Access-Control-Allow-Origin: *` 等头。若浏览器发起预检 `OPTIONS`，需保证网关/容器对 OPTIONS 返回成功并与业务 CORS 策略一致（与主站其它公开 API 相同）。

---

## 6. 与「重置密码验证码」的区别

**`POST /api/user/send-reset-code`** 用于忘记密码场景，**不要求** `captchaPassToken`（与注册发信接口不同）。实现以服务端代码为准。
