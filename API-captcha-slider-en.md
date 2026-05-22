# Slider captcha & registration email verification APIs

These endpoints support the **sign-up** flow: load a slider puzzle, verify it to obtain a short-lived pass token, then send the email verification code. Base URL is the same as the main doc (e.g. `https://music.cnmsb.xin`).

**Auth:** No login required; no `Authorization` header.

---

## Recommended client flow

1. Call **`GET /api/captcha/slider`** to obtain `captchaToken`, images, and layout fields.
2. After the user aligns the puzzle piece, call **`POST /api/captcha/slider/verify`** with `captchaToken` and `captchaOffsetX` (piece left X in background coordinates, integer pixels).
3. On success, use **`captchaPassToken`** from the response body and immediately call **`POST /api/user/send-verification`** with `email`, `username` (optional), and **`captchaPassToken`**.
4. User completes registration with **`POST /api/user/register`** using the code from email only (register does **not** require captcha fields).

**Rules:**

- Each **`captchaToken`** is consumed by a single `verify` call (success or failure).
- Each **`captchaPassToken`** is single-use for `send-verification` and expires after a short TTL.
- Horizontal tolerance is **±5 px** vs server secret.

**TTL (implementation):**

| Item | Notes |
|------|--------|
| Challenge `captchaToken` | ~**3 minutes** from issue; removed on verify |
| `captchaPassToken` | ~**2 minutes**; must be sent with `send-verification` once |

---

## `GET /api/captcha/slider`

Returns JSON:

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

---

## `POST /api/captcha/slider/verify`

**Headers:** `Content-Type: application/json`

**Body:**

```json
{
  "captchaToken": "string",
  "captchaOffsetX": 0
}
```

**Success:**

```json
{
  "success": true,
  "message": "验证通过喵",
  "data": { "captchaPassToken": "string" }
}
```

**Failure:** `success: false`, `data: null`, `message` describes missing fields, mismatch, or expired challenge.

---

## `POST /api/user/send-verification`

**Headers:** `Content-Type: application/json`

**Body:**

```json
{
  "email": "user@example.com",
  "username": "User",
  "captchaPassToken": "string"
}
```

| Field | Required | Notes |
|-------|----------|--------|
| `email` | Yes | Recipient |
| `username` | No | Defaults may apply server-side |
| `captchaPassToken` | Yes | From successful `slider/verify` |

Without a valid pass token, the server responds with `success: false` and a message such as asking to complete the slider first, or that verification expired/was already used.

---

## `POST /api/user/register`

Does **not** accept slider fields; only account fields and `verificationCode` from email. See main [README-EN.md](README-EN.md) user section.

---

## Difference from reset-password code

**`POST /api/user/send-reset-code`** does **not** require `captchaPassToken`.
