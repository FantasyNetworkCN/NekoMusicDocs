# Neko歌姬计划 API 文档

### 使用本 API 需遵守本项目 LICENSE 协议，必须开源并保留 Neko歌姬计划 署名及源码链接！

#### 更新时间 2026年5月30日
## 概述

Neko歌姬计划提供完整的 RESTful API，支持音乐搜索、播放、用户认证、收藏、横屏分享视频生成等功能。所有 API 都基于 HTTP/HTTPS 协议，使用 JSON 格式进行数据交换。

**基础 URL:** `https://music.cnmsb.xin`

## 目录

- [认证说明](#认证说明)
- [用户相关 API](#用户相关-api)
- [滑块人机验证与注册邮箱验证码（专项）](API-滑块与人机验证.md)
- [歌单相关 API](#歌单相关-api)
- [歌手相关 API](#歌手相关-api)
- [VIP 与价目 API](#vip-与价目-api)
- [音乐相关 API](#音乐相关-api)
- [听歌识曲 API](#听歌识曲-api)
- [分享视频渲染 API](#分享视频渲染-api)
- [错误码说明](#错误码说明)

---

## 认证说明

### 用户认证

所有需要用户登录的 API 都需要在请求头中包含用户 Token：

```
Authorization: <token>
```

Token 在用户登录时生成并返回给客户端。

**Token 有效期:** 30 天

---

## 用户相关 API

### 1. 用户注册

**端点:** `POST /api/user/register`

**请求头:**
```
Content-Type: application/json
```

**请求体:**
```json
{
  "username": "string",      // 用户名 (必填)
  "password": "string",      // 密码 (必填)
  "email": "string",         // 邮箱 (必填)
  "verificationCode": "string"  // 邮箱验证码 (必填)
}
```

**响应示例:**
```json
{
  "success": true,
  "message": "注册成功",
  "data": {
    "user": {
      "id": 1,
      "username": "用户名",
      "email": "email@example.com",
      "createdAt": "2024-01-01T00:00:00"
    },
    "token": "64位十六进制字符串"
  }
}
```

### 2. 用户登录

**端点:** `POST /api/user/login`

**请求头:**
```
Content-Type: application/json
```

**请求体:**
```json
{
  "username": "string",  // 邮箱
  "password": "string"   // 密码
}
```

**响应示例:**
```json
{
  "success": true,
  "message": "登录成功",
  "data": {
    "user": {
      "id": 1,
      "username": "用户名",
      "email": "email@example.com",
      "createdAt": "2024-01-01T00:00:00",
      "isVip": false,
      "vipExpiresAt": null
    },
    "token": "64位十六进制字符串"
  }
}
```

### 3. 发送邮箱验证码

用于**注册**前向邮箱发送数字验证码。须先完成滑块校验并取得 `captchaPassToken`（详见专项文档）。

**专项文档（推荐）：** [API-滑块与人机验证.md](API-滑块与人机验证.md)（含 `GET /api/captcha/slider`、`POST /api/captcha/slider/verify` 与本接口的完整字段与流程说明）

**端点:** `POST /api/user/send-verification`

**请求头:**
```
Content-Type: application/json
```

**请求体:**
```json
{
  "email": "string",
  "username": "string",
  "captchaPassToken": "string"
}
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `email` | 是 | 接收验证码的邮箱 |
| `username` | 否 | 展示名等，缺省可由服务端按「用户」处理 |
| `captchaPassToken` | 是 | 调用 `POST /api/captcha/slider/verify` 成功后返回的一次性通行令牌 |

**响应示例:**
```json
{
  "success": true,
  "message": "验证码已发送至您的邮箱",
  "data": null
}
```

未通过人机验证或令牌无效时，`success` 为 `false`，`message` 会说明原因（如须先完成滑动拼图、令牌已失效等）。发信频率过高时可能返回 HTTP `429`（若服务端启用限流）。

### 4. 发送重置密码验证码

**端点:** `POST /api/user/send-reset-code`

**请求头:**
```
Content-Type: application/json
```

**请求体:**
```json
{
  "email": "string"  // 注册邮箱（必填）
}
```

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "验证码已发送至您的邮箱",
  "data": null
}
```

**说明:**

- 若邮箱未注册：`success` 为 `false`，`message` 为「该邮箱未注册」，**不会**发送验证码。
- 验证码逻辑与注册验证码相同，通过邮件发送。

### 5. 重置密码

**端点:** `POST /api/user/reset-password`

**请求头:**
```
Content-Type: application/json
```

**请求体:**
```json
{
  "email": "string",       // 邮箱（必填）
  "code": "string",        // 邮件验证码（必填）
  "newPassword": "string"  // 新密码（必填，6–30 位）
}
```

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "密码重置成功，请使用新密码登录",
  "data": null
}
```

### 6. 获取用户头像

**端点:** `GET /api/user/avatar/{userId}`

**路径参数:**
- `userId`: 用户 ID

**响应:** 图片文件 (PNG/JPG)

### 7. 获取收藏列表

**端点:** `GET /api/user/favorites`

**请求头:**
```
Authorization: <token>
```

**响应示例:**
```json
{
  "success": true,
  "favorites": [
    {
      "id": 1,
      "title": "歌曲标题",
      "artist": "艺术家",
      "album": "专辑",
      "duration": 180,
      "filename": "song.mp3"
    }
  ]
}
```

### 8. 添加收藏

**端点:** `POST /api/user/favorites`

**请求头:**
```
Authorization: <token>
Content-Type: application/json
```

**请求体:**

音乐 ID 二选一或可同时使用（会合并后按顺序 **去重** 再依次收藏）：

- `musicId`：单个音乐 ID（与旧版客户端兼容）
- `musicIds`：整数数组，一次收藏多首

至少需在合并去重后得到 **至少一个** 音乐 ID，否则返回 `400`。

```json
{
  "musicId": 1
}
```

批量收藏示例：

```json
{
  "musicIds": [1, 2, 3]
}
```

也可同时传 `musicId` 与 `musicIds`（重复 ID 只会收藏一次）。

**响应示例（成功，单首）:**
```json
{
  "success": true,
  "addedCount": 1,
  "message": "收藏成功"
}
```

**响应示例（成功，多首）:**
```json
{
  "success": true,
  "addedCount": 3,
  "message": "已收藏 3 首音乐"
}
```

**响应示例（部分失败，HTTP 400）:**  
当部分 ID 已在收藏中（`INSERT IGNORE` 跳过）时，已成功收藏的仍会保留；响应包含未成功的 ID 列表。

```json
{
  "success": false,
  "addedCount": 1,
  "failedMusicIds": [99, 100],
  "message": "部分音乐未能收藏（已存在或收藏失败），失败数量: 2"
}
```

### 9. 删除收藏

**端点:** `DELETE /api/user/favorites/{musicId}`

**请求头:**
```
Authorization: <token>
```

**路径参数:**
- `musicId`: 音乐 ID

**响应示例:**
```json
{
  "success": true,
  "message": "取消收藏成功"
}
```

### 10. 获取收藏歌单列表

**端点:** `GET /api/user/favorite-playlists`

**请求头:**
```
Authorization: <token>
```

**响应示例:**
```json
{
  "success": true,
  "playlists": [
    {
      "id": 1,
      "name": "我的歌单",
      "description": "这是我的歌单描述",
      "musicCount": 5,
      "createdAt": 1706500800000,
      "updatedAt": 1706501100000,
      "favoriteTime": 1706501400000,
      "creator": {
        "id": 1,
        "username": "用户名"
      }
    }
  ]
}
```

**说明:**
- `createdAt`, `updatedAt`, `favoriteTime` 为 Unix 时间戳（毫秒）
- 只返回当前用户收藏的歌单列表
- 按收藏时间倒序排列

### 11. 收藏歌单

**端点:** `POST /api/user/favorite-playlists`

**请求头:**
```
Authorization: <token>
Content-Type: application/json
```

**请求体:**
```json
{
  "playlistId": 1  // 歌单 ID
}
```

**响应示例:**
```json
{
  "success": true,
  "message": "收藏歌单成功"
}
```

**响应示例（已收藏）:**
```json
{
  "success": false,
  "message": "收藏歌单失败或已收藏"
}
```

### 12. 取消收藏歌单

**端点:** `DELETE /api/user/favorite-playlists/{playlistId}`

**请求头:**
```
Authorization: <token>
```

**路径参数:**
- `playlistId`: 歌单 ID

**响应示例:**
```json
{
  "success": true,
  "message": "取消收藏歌单成功"
}
```

### 13. 获取收藏歌单内音乐

**端点:** `GET /api/user/favorite-playlists/{playlistId}`

**请求头:**
```
Authorization: <token>
```

**路径参数:**
- `playlistId`: 歌单 ID

**响应示例:**
```json
{
  "success": true,
  "music": [
    {
      "id": 1,
      "title": "歌曲标题",
      "artist": "艺术家",
      "album": "专辑",
      "duration": 180,
      "filename": "song.mp3",
      "position": 1
    }
  ]
}
```

**响应示例（未收藏）:**
```json
{
  "success": false,
  "message": "用户未收藏该歌单"
}
```

**说明:**
- 只有收藏过该歌单的用户才能查看歌单内的音乐
- 音乐按 position 字段升序排列

### 14. 获取每日推荐

**端点:** `GET /api/user/recommendations/daily`

**请求头:**
```
Authorization: <token>
```

**说明:**
- 该接口返回当前用户“今日推荐”列表。
- 推荐结果每日更新一次（UTC+8 每日 00:00）。
- 推荐列表**不会包含**用户已收藏曲目。
- 用户自建歌单、已收藏歌单中的曲目**可能入选**，但会降权排序，出现频率更低。

**响应示例:**
```json
{
  "success": true,
  "date": "2026-05-28",
  "count": 30,
  "data": [
    {
      "rank": 1,
      "musicId": 13751,
      "title": "天使ロード中…^_−☆",
      "artist": "三Z-STUDIO&HOYO-MiX",
      "album": "绝区零-天使加载中…^_−☆",
      "language": "日语",
      "tags": "二次元，日语，游戏",
      "score": 4.93,
      "source": "ai",
      "reason": "与近期收藏艺人和标签更匹配"
    }
  ]
}
```

**字段说明:**
- `date`: 推荐结果所属日期（东八区）
- `count`: 返回条数
- `source`: 推荐来源（`ai` 或 `rule`）
- `reason`: 推荐理由（用于前端展示）

### 15. 上传用户头像

**端点:** `POST /api/user/avatar/upload`

**请求头:**
```
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**请求参数:**
- `avatar`: 图片文件 (multipart/form-data)
  - 支持格式：jpg, jpeg, png, gif, webp, bmp
  - 最大文件大小：10MiB

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "头像上传成功",
  "avatarPath": "avatars/1_550e8400-e29b-41d4-a716-446655440000.jpg"
}
```

**响应示例（失败）:**
```json
{
  "error": "未授权访问"
}
```

或

```json
{
  "error": "文件大小超过10MiB限制"
}
```

### 16. 用户上传音乐

**端点:** `POST /api/user/upload`

**请求头:**
```
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**请求参数:**
- `title`: 歌曲标题（必填）
- `artist`: 歌手名称（必填）
- `language`: 语言（必填）
  - 可选值：中文、粤语、上海语、英文、日语、韩语、法语、德语、俄语、纯音乐
- `album`: 专辑名称（可选）
- `tags`: 标签（可选）
- `duration`: 音乐时长，单位秒（必填）
- `uploadUserId`: 上传用户ID（必填）
- `musicFile`: 音乐文件（必填，multipart/form-data）
  - 支持格式：MP3、FLAC、WAV
- `coverFile`: 封面图片文件（可选，multipart/form-data）
  - 支持格式：jpg, jpeg, png, gif, webp, bmp
- `lyricsFile`: 歌词文件（可选，multipart/form-data）
  - 支持格式：lrc
  - 如果不提供歌词文件，系统会自动使用默认的空歌词文件

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "上传成功，等待审核",
  "data": {
    "id": 1,
    "status": "pending",
    "createdAt": "2026-02-12T16:30:00"
  }
}
```

**响应示例（重复音乐）:**
```json
{
  "success": false,
  "message": "已有重复音乐，请检查后重新上传"
}
```

**响应示例（未授权）:**
```json
{
  "success": false,
  "message": "未授权访问"
}
```

**说明:**
- 上传的音乐会进入待审核状态（`status: "pending"`）
- 管理员审核通过后，音乐会正式添加到音乐库
- 系统会自动检查是否有重复的音乐（相同的标题、歌手、专辑）
- 如果没有上传歌词文件，系统会自动创建一个空的歌词文件（no_lrc.lrc）
- 上传的文件会保存在 `user_upload/` 目录下，文件名格式为 `music_<timestamp>.<ext>`

**注意事项:**
- 标题、歌手、语言为必填项
- 音乐文件必须提供，且必须是MP3、FLAC或WAV格式
- 音乐时长需要用户手动输入或通过前端解析后传入
- 封面和歌词文件是可选的
- 上传成功后，用户需要等待管理员审核才能在音乐库中看到上传的音乐

**前端集成示例:**
```javascript
async function uploadMusic(formData) {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch('https://music.cnmsb.xin/api/user/upload', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`
    },
    body: formData
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('音乐上传成功，等待审核');
    console.log('上传记录ID:', data.data.id);
    console.log('状态:', data.data.status);
  } else {
    console.error('音乐上传失败:', data.message);
  }
  return data;
}

// 使用示例
const formData = new FormData();
formData.append('title', '歌曲标题');
formData.append('artist', '歌手名称');
formData.append('language', '中文');
formData.append('album', '专辑名称');
formData.append('tags', '流行,华语');
formData.append('duration', 235); // 音乐时长（秒）
formData.append('uploadUserId', 0);
formData.append('musicFile', musicFileObject);
formData.append('coverFile', coverFileObject); // 可选
formData.append('lyricsFile', lyricsFileObject); // 可选

uploadMusic(formData);
```

### 17. 修改用户密码

**端点:** `POST /api/user/password/change`

**请求头:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**请求体:**
```json
{
  "oldPassword": "string",  // 原密码（必填）
  "newPassword": "string"   // 新密码（必填，长度不能少于6位）
}
```

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "密码修改成功"
}
```

**响应示例（失败）:**
```json
{
  "error": "原密码错误"
}
```

或

```json
{
  "error": "新密码长度不能少于6位"
}
```

或

```json
{
  "error": "新密码不能与原密码相同"
}
```

### 18. 获取用户上传审核通过的音乐

**端点:** `GET /api/user/uploaded-music`

**请求头:**
```
Authorization: <token>
```

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "获取用户上传审核通过的音乐列表成功",
  "userId": 123,
  "musicList": [
    {
      "id": 1,
      "title": "歌曲标题",
      "artist": "艺术家",
      "album": "专辑",
      "duration": 180,
      "language": "中文",
      "tags": "流行,华语",
      "fileFormat": "mp3",
      "createdAt": "2026-02-16T10:00:00"
    }
  ],
  "total": 1
}
```

**响应示例（未登录）:**
```json
{
  "success": false,
  "message": "用户未登录或token无效"
}
```

**说明:**
- 此 API 需要登录才能访问
- 只返回当前用户上传的、审核通过的音乐列表
- 音乐按创建时间倒序排列（最新的在前面）
- 只包含审核状态为 `approved` 的音乐
- 每首音乐包含：
  - id, title, artist, album, duration
  - language, tags, fileFormat
  - createdAt: 创建时间

**使用场景:**
- 个人中心展示用户上传的作品
- 用户查看自己的音乐库
- 统计用户上传的音乐数量

**前端集成示例:**
```javascript
async function getUserUploadedMusic() {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch('https://music.cnmsb.xin/api/user/uploaded-music', {
    method: 'GET',
    headers: {
      'Authorization': token
    }
  });
  
  const data = await response.json();
  if (data.success) {
    console.log(`获取到 ${data.total} 首审核通过的音乐:`, data.musicList);
    // data.musicList 是一个数组，包含用户上传的审核通过的音乐
    // 每首音乐包含完整的歌曲信息
  } else {
    console.error('获取用户上传音乐失败:', data.message);
  }
  return data;
}
```

---

## 歌单相关 API

- [搜索歌单](#1-搜索歌单)
- [获取歌单详情](#2-获取歌单详情)
- [获取歌单音乐列表](#3-获取歌单音乐列表)
- [创建歌单](#4-创建歌单)
- [获取歌单列表](#5-获取歌单列表)
- [更新歌单](#6-更新歌单)
- [删除歌单](#7-删除歌单)
- [添加音乐到歌单](#8-添加音乐到歌单)
- [从歌单中移除音乐](#9-从歌单中移除音乐)
- [歌单权限说明](#歌单权限说明)

### 1. 搜索歌单

**端点:** `POST /api/playlists/search`

**无需登录**

**请求头:**
```
Content-Type: application/json
```

**请求体:**
```json
{
  "query": "string"  // 搜索关键词
}
```

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "搜索成功",
  "total": 2,
  "results": [
    {
      "id": 1,
      "userId": 1,
      "name": "我的歌单",
      "description": "这是我的歌单描述",
      "musicCount": 5,
      "createdAt": "2026-01-29 12:00:00",
      "updatedAt": "2026-01-29 12:05:00",
      "firstMusicId": 1,
    },
    {
      "id": 2,
      "userId": 2,
      "name": "流行音乐",
      "description": "收藏的流行歌曲",
      "musicCount": 10,
      "createdAt": "2026-01-28 10:00:00",
      "updatedAt": "2026-01-28 10:00:00",
      "firstMusicCover": "/api/user/avatar/default"
    }
  ]
}
```

**响应示例（失败）:**
```json
{
  "success": false,
  "message": "缺少搜索关键词"
}
```

**说明:**
- 此 API **无需登录**即可访问
- 搜索关键词会匹配歌单名称和描述
- 返回结果按创建时间倒序排列
- 每个结果包含：
  - 歌单基本信息（id, name, description, musicCount, createdAt, updatedAt）
  - 第一首音乐的 ID（firstMusicId）
  - 第一首音乐的封面 URL（firstMusicCover）
  - 如果歌单没有音乐，firstMusicCover 为默认头像 `/api/user/avatar/default`

**注意事项:**
- 搜索关键词不能为空
- 搜索是模糊匹配，使用 `LIKE %keyword%`
- 使用 POST 方式，参数在请求体中传递

### 2. 获取歌单详情

**端点:** `GET /api/playlist/{id}`

**路径参数:**
- `id`: 歌单 ID

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "获取歌单详情成功",
  "playlist": {
    "id": 1,
    "userId": 1,
    "name": "我的歌单",
    "description": "这是我的歌单描述",
    "musicCount": 5,
    "createdAt": "2026-01-29 12:00:00",
    "updatedAt": "2026-01-29 12:05:00"
  }
}
```

**响应示例（歌单不存在）:**
```json
{
  "success": false,
  "message": "歌单不存在"
}
```

**说明:**
- 此 API **无需登录**即可访问
- 任何用户（包括未登录用户）都可以查看歌单的基本信息
- 返回的歌单信息包含 `userId` 字段，可以识别歌单的创建者

### 3. 创建歌单

**端点:** `POST /api/user/playlist/create`

**请求头:**
```
Authorization: <token>
Content-Type: application/json
```

**请求体:**
```json
{
  "name": "string",          // 歌单名称 (必填)
  "description": "string"   // 歌单描述 (可选)
}
```

**参数限制:**
- `name`: 长度不能超过 255 个字符
- `description`: 长度不能超过 500 个字符

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "歌单创建成功",
  "playlist": {
    "id": 1,
    "name": "我的歌单",
    "description": "这是我的歌单描述",
    "musicCount": 0,
    "createdAt": "2026-01-29 12:00:00",
    "updatedAt": "2026-01-29 12:00:00"
  }
}
```

**响应示例（失败）:**
```json
{
  "success": false,
  "message": "歌单名称不能为空"
}
```

### 4. 获取歌单列表

**端点:** `GET /api/user/playlists`

**请求头:**
```
Authorization: <token>
```

**响应示例:**
```json
{
  "success": true,
  "message": "获取歌单列表成功",
  "isVip": false,
  "vipExpiresAt": null,
  "playlists": [
    {
      "id": 1,
      "userId": 1,
      "name": "我的歌单",
      "description": "这是我的歌单描述",
      "musicCount": 5,
      "createdAt": "2026-01-29 12:00:00",
      "updatedAt": "2026-01-29 12:05:00"
    }
  ]
}
```

**说明:** 此 API 只返回当前登录用户创建的歌单列表，歌单按创建时间倒序排列。每个用户只能看到自己创建的歌单，不会看到其他用户的歌单。响应 JSON **根级**还包含当前用户的会员信息：`isVip`（布尔）、`vipExpiresAt`（会员到期时间，ISO-8601；非会员或无到期记录时为 `null`），便于客户端与登录接口字段对齐。

**注意事项:**
- 此 API 需要登录才能访问
- 只返回当前用户的歌单，不会混合其他用户的歌单
- 如果要查看其他用户的歌单，请使用 `GET /api/playlist/{id}` API
- 歌单按创建时间倒序排列（最新的在前面）

### 5. 更新歌单

**端点:** `POST /api/user/playlist/update`

**请求头:**
```
Authorization: <token>
Content-Type: application/json
```

**请求体:**
```json
{
  "id": 1,                   // 歌单 ID (必填)
  "name": "string",          // 歌单名称 (必填)
  "description": "string"   // 歌单描述 (可选，不修改则不传此字段或传null)
}
```

**参数说明:**
- `id`: 歌单 ID（必填），必须是当前用户创建的歌单
- `name`: 歌单名称（必填），长度不能超过 255 个字符
- `description`: 歌单描述（可选），长度不能超过 500 个字符
  - 如果只想修改描述，仍需传递 `name` 字段（使用当前名称）
  - 如果不修改描述，可以不传此字段或传 `null`

**使用场景:**
1. **同时修改名称和描述**：传递所有字段
2. **只修改描述**：传递 `id`、`name`（当前值）和新的 `description`
3. **只修改名称**：传递 `id` 和新的 `name`，不传 `description`

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "歌单更新成功",
  "playlist": {
    "id": 1,
    "name": "更新的歌单名称",
    "description": "更新的描述",
    "musicCount": 5,
    "createdAt": "2026-01-29 12:00:00",
    "updatedAt": "2026-01-29 12:10:00"
  }
}
```

**响应示例（权限错误）:**
```json
{
  "success": false,
  "message": "无权限修改此歌单"
}
```

**说明:** 只有歌单的创建者（user_id 匹配）才能更新歌单信息。

### 6. 删除歌单

**端点:** `POST /api/user/playlist/delete`

**请求头:**
```
Authorization: <token>
Content-Type: application/json
```

**请求体:**
```json
{
  "id": 1  // 歌单 ID (必填)
}
```

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "歌单删除成功"
}
```

**响应示例（权限错误）:**
```json
{
  "success": false,
  "message": "无权限删除此歌单"
}
```

**说明:**
- 只有歌单的创建者（user_id 匹配）才能删除歌单
- 删除歌单会级联删除 `playlist_music` 表中的所有关联记录
- 此操作不可恢复

### 7. 获取歌单音乐列表

**端点:** `GET /api/user/playlist/music/{playlistId}`

**请求头:**
```
Authorization: <token>
```

**路径参数:**
- `playlistId`: 歌单 ID

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "获取歌单音乐列表成功",
  "playlistId": 1,
  "total": 5,
  "musicList": [
    {
      "id": 1,
      "title": "歌曲标题",
      "artist": "艺术家",
      "album": "专辑",
      "duration": 180,
      "fileFormat": "mp3",
      "language": "中文",
      "position": 1,
      "addedAt": "2026-01-29 12:00:00"
    }
  ]
}
```

**响应示例（歌单不存在）:**
```json
{
  "success": false,
  "message": "歌单不存在"
}
```

**说明:**
- 音乐列表按照 `position` 字段升序排列
- 返回的音乐信息包含完整的歌曲详情和添加时间
- 此 API **无需登录**即可访问（后端已移除 token 验证）
- 任何用户（包括未登录用户）都可以查看歌单内容

### 8. 添加音乐到歌单

**端点:** `POST /api/user/playlist/music/add`

**请求头:**
```
Authorization: <token>
Content-Type: application/json
```

**请求体:**

`playlistId` 必填。音乐 ID 二选一或可同时使用（会合并后按顺序 **去重** 再依次添加）：

- `musicId`：单个音乐 ID（与旧版客户端兼容）
- `musicIds`：整数数组，一次添加多首

至少需在合并去重后得到 **至少一个** 音乐 ID，否则返回 `400`。

```json
{
  "playlistId": 1,
  "musicId": 1
}
```

批量添加示例：

```json
{
  "playlistId": 1,
  "musicIds": [1, 2, 3]
}
```

也可同时传 `musicId` 与 `musicIds`（重复 ID 只会添加一次）。

**响应示例（成功，单首）:**
```json
{
  "success": true,
  "addedCount": 1,
  "message": "音乐添加到歌单成功"
}
```

**响应示例（成功，多首）:**
```json
{
  "success": true,
  "addedCount": 3,
  "message": "已向歌单添加 3 首音乐"
}
```

**响应示例（部分失败，HTTP 400）:**  
当请求中部分 ID 已在歌单中或添加失败时，已成功添加的仍会生效；响应包含未成功的 ID 列表。

```json
{
  "success": false,
  "addedCount": 1,
  "failedMusicIds": [99, 100],
  "message": "部分音乐未能添加到歌单（已存在或添加失败），失败数量: 2"
}
```

**响应示例（全部失败）:**
```json
{
  "success": false,
  "message": "音乐添加到歌单失败或音乐已存在于歌单中"
}
```

**响应示例（权限错误）:**
```json
{
  "success": false,
  "message": "无权限修改此歌单"
}
```

**说明:**
- 只有歌单的创建者才能添加音乐到歌单
- 若传入的某个 ID 已在歌单中或添加失败，接口返回 `400`，`failedMusicIds` 列出失败的 ID；此前已成功添加的曲目不会回滚
- `musicIds` 必须是 JSON 数组，否则会返回 `400` 及相应提示
- 每首音乐会自动添加到歌单**最上面**（position = 1）；批量时按去重后的顺序逐首添加，与逐次调用单首添加效果一致（数组中**靠后的 ID** 最终在列表更靠前）
- 添加成功后会自动更新歌单的 `musicCount` 字段

### 9. 从歌单中移除音乐

**端点:** `POST /api/user/playlist/music/remove`

**请求头:**
```
Authorization: <token>
Content-Type: application/json
```

**请求体:**

`playlistId` 必填。音乐 ID 二选一或可同时使用（会合并后按顺序 **去重** 再依次移除）：

- `musicId`：单个音乐 ID（与旧版客户端兼容）
- `musicIds`：整数数组，一次移除多首

至少需在合并去重后得到 **至少一个** 音乐 ID，否则返回 `400`。

```json
{
  "playlistId": 1,
  "musicId": 1
}
```

批量移除示例：

```json
{
  "playlistId": 1,
  "musicIds": [1, 2, 3]
}
```

也可同时传 `musicId` 与 `musicIds`（重复 ID 只会移除一次）。

**响应示例（成功，单首）:**
```json
{
  "success": true,
  "removedCount": 1,
  "message": "音乐从歌单中移除成功"
}
```

**响应示例（成功，多首）:**
```json
{
  "success": true,
  "removedCount": 3,
  "message": "已从歌单中移除 3 首音乐"
}
```

**响应示例（部分失败，HTTP 400）:**  
当请求中部分 ID 不在歌单中或移除失败时，已成功移除的仍会生效；响应包含未成功的 ID 列表。

```json
{
  "success": false,
  "removedCount": 1,
  "failedMusicIds": [99, 100],
  "message": "部分音乐未能从歌单中移除（不在歌单中或移除失败），失败数量: 2"
}
```

**响应示例（权限错误）:**
```json
{
  "success": false,
  "message": "无权限修改此歌单"
}
```

**说明:**
- 只有歌单的创建者才能从歌单中移除音乐
- 若传入的某个 ID 不在歌单中或移除失败，接口返回 `400`，`failedMusicIds` 列出失败的 ID；此前已成功移除的曲目不会回滚
- `musicIds` 必须是 JSON 数组，否则会返回 `400` 及相应提示
- 移除音乐后会自动重新排序剩余音乐的 position（删除位置之后的 position - 1）；多首时按去重后的顺序逐首移除，与逐次调用单首移除效果一致
- 移除成功后会自动更新歌单的 `musicCount` 字段

---
## 歌单权限说明

### 权限规则

| 操作 | 权限要求 |
|------|---------|
| 搜索歌单 | 无需登录（任何用户都可以搜索） |
| 获取歌单详情 | 无需登录（任何用户都可以查看） |
| 查看歌单列表 | 任何登录用户（只返回自己的歌单） |
| 查看歌单内容 | 无需登录（任何用户都可以查看歌单中的音乐） |
| 创建歌单 | 任何登录用户 |
| 更新歌单 | 只有歌单的创建者 |
| 删除歌单 | 只有歌单的创建者 |
| 添加音乐到歌单 | 只有歌单的创建者 |
| 从歌单中移除音乐 | 只有歌单的创建者 |

### 权限验证

- **查看权限**：所有用户（包括未登录用户）都可以搜索歌单、查看歌单详情和歌单内容，无需额外权限验证
- **歌单列表**：登录用户只能看到自己创建的歌单，不会看到其他用户的歌单
- **修改权限**：所有修改和删除操作都会验证用户是否是歌单的创建者（通过 token 识别用户身份）
- 如果用户尝试修改或删除不属于自己的歌单，服务器会返回 `403 Forbidden` 状态码和相应的错误信息

---

## VIP 与价目 API

公开接口用于**展示**当前在售会员套餐与时长价格；会员开通请在站内「会员中心」完成。下方补充了客户端实际使用的购买下单接口，后台管理接口不在本文档公开范围。

### 1. 查询 VIP 价目表（无需登录）

**端点:** `GET /api/vip/pricing`

**响应示例:**
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "months": 0,
      "days": 7,
      "priceYuan": 2.99,
      "sortOrder": 0,
      "updatedAt": "2026-05-15T12:00:00+08:00"
    }
  ]
}
```

**字段说明:**

| 字段 | 说明 |
|------|------|
| `months` / `days` | 套餐时长（月 + 天），至少一项大于 0 |
| `priceYuan` | 价格（人民币元） |
| `sortOrder` | 展示顺序（数值越小越靠前，以服务端为准） |
| `updatedAt` | 该行最近更新时间（东八区 ISO-8601 带偏移） |

### 2. 发起 VIP 购买（需登录）

**端点:** `POST /api/vip/pay/create`

**认证:** 需要登录，`Authorization: <token>`

**请求头:**
```http
Content-Type: application/json
```

**请求体示例:**
```json
{
  "pricingId": 1,
  "payType": "alipay"
}
```

**字段说明:**

| 字段 | 说明 |
|------|------|
| `pricingId` | 价目表中的套餐 ID，必填 |
| `payType` | 支付方式，可选，支持 `alipay` / `wxpay`，默认 `alipay` |

**响应示例:**
```json
{
  "success": true,
  "data": {
    "outTradeNo": "VIP202605301234567890",
    "payurl": "https://...",
    "qrcode": "https://..."
  }
}
```

**会员状态字段（用户侧）:** 登录接口 `data.user` 与 `GET /api/user/playlists` 响应根级均含 `isVip`、`vipExpiresAt`，含义一致，便于客户端展示与刷新。

---

## 音乐相关 API

### 1. 搜索音乐

**端点:** `POST /api/music/search`

**无需登录**

**请求头:**
```
Content-Type: application/json
```

同一端点支持两种模式，**`query` 与 `items` 二选一**，不可同时提供。

#### 模式 A：模糊搜索（单关键词）

**请求体:**
```json
{
  "query": "晴天"
}
```

**说明:**
- 在曲库内对标题、歌词、歌手、专辑及拼音列模糊匹配，按相关度排序，最多返回约 50 条
- 无匹配时 `results` 为 `null`

**响应示例（有结果）:**
```json
{
  "success": true,
  "message": "搜索成功",
  "results": [
    {
      "id": 1,
      "title": "晴天",
      "artist": "周杰伦",
      "album": "叶惠美",
      "duration": 269,
      "uploadUserId": 0,
      "createdAt": "2024-01-01 12:00:00.0",
      "lrc": true
    }
  ]
}
```

**响应示例（无结果）:**
```json
{
  "success": false,
  "message": "未找到匹配的音乐",
  "results": null
}
```

#### 模式 B：批量精确搜索（歌名 + 歌手）

**请求体:**
```json
{
  "items": [
    { "title": "晴天", "artist": "周杰伦" },
    { "title": "STAY WIT ME", "artist": "TRYBEL BAND" },
    { "title": "某首仅歌名" }
  ]
}
```

| 字段 | 说明 |
|------|------|
| `items` | 必填，数组长度不限；顺序与返回的 `results` 一一对应 |
| `items[].title` | 必填，歌名（trim 后非空） |
| `items[].artist` | 可选；有则按歌名+歌手精确匹配（繁简归一） |

**匹配规则（均在曲库内完成，按条独立匹配）:**
1. **精确匹配**：有 `artist` 时歌名+歌手同时相等（繁简归一）；无 `artist` 时歌名精确且该标题下歌手唯一
2. **择优匹配**：精确未命中时，在曲库内按歌名/歌手相似度打分，取达阈值的最佳一条（支持多歌手分段、忽略括号内备注等）
3. 仍无合格结果时，该条为 `null`

**响应示例（部分命中）:**
```json
{
  "success": true,
  "message": "搜索成功（2/3 已找到）",
  "results": [
    {
      "id": 1,
      "title": "晴天",
      "artist": "周杰伦",
      "album": "叶惠美",
      "duration": 269,
      "uploadUserId": 0,
      "createdAt": "2024-01-01 12:00:00.0"
    },
    null,
    {
      "id": 42,
      "title": "STAY WIT ME",
      "artist": "TRYBEL BAND",
      "album": "未知专辑",
      "duration": 200,
      "uploadUserId": 0,
      "createdAt": "2026-05-24 10:00:00.0"
    }
  ]
}
```

**响应示例（全部未命中）:**
```json
{
  "success": false,
  "message": "未找到匹配的音乐",
  "results": [null, null]
}
```

#### 音乐对象字段（两种模式相同）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | number | 音乐 ID |
| `title` | string | 标题 |
| `artist` | string | 歌手 |
| `album` | string | 专辑 |
| `duration` | number | 时长（秒） |
| `uploadUserId` | number | 上传用户 ID，无则为 0 |
| `createdAt` | string | 入库时间 |

**资源 URL（由客户端按 `id` 拼接，响应中不再返回路径字段）:**
- 音频：`GET /api/music/file/{id}`
- 封面：`GET /api/music/cover/{id}`

**错误请求（400）:**
```json
{
  "error": "请求格式错误: query 与 items 不能同时提供"
}
```

### 2. 获取音乐信息

**端点:** `GET /api/music/info/{id}`

**路径参数:**
- `id`: 音乐 ID

**响应示例:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "歌曲标题",
    "artist": "艺术家",
    "album": "专辑",
    "duration": 180,
    "coverUrl": "/api/music/cover/1",
    "fileUrl": "/api/music/file/1",
    "lyrics": "歌词内容"
  }
}
```

### 3. 获取音乐文件

**端点:** `GET /api/music/file/{id}`

**路径参数:**
- `id`: 音乐 ID

**响应:** 音频文件 (响应标头content-type返回媒体格式，例如audio/flac)

### 4. 获取音乐封面

**端点:** `GET /api/music/cover/{id}`

**路径参数:**
- `id`: 音乐 ID

**响应:** 图片文件 (PNG/JPG)

### 5. 听歌识曲

根据一段短录音，在 NekoMusic 自有曲库中匹配歌曲。该接口完全在本站服务端完成声纹提取与匹配，**不调用网易云、Shazam 或其他第三方识曲 API，也不会将录音转发到外部服务**。

**端点:** `POST /api/music/recognize`

**认证:** 无需登录

**请求格式:** `multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `audio` | 文件 | 是 | 麦克风或本地录音文件，推荐 3–20 秒；支持 FFmpeg 可解码的音频格式 |

**curl 示例:**

```bash
curl -sS -X POST 'https://music.cnmsb.xin/api/music/recognize' \
  -F 'audio=@sample.m4a'
```

**识别成功（HTTP 200）:**

```json
{
  "success": true,
  "matched": true,
  "message": "识别成功",
  "data": {
    "id": 1,
    "title": "晴天",
    "artist": "周杰伦",
    "album": "叶惠美",
    "duration": 269,
    "language": "中文",
    "tags": "",
    "filePath": "/api/music/file/1",
    "coverFilePath": "/api/music/cover/1",
    "coverUrl": "/api/music/cover/1",
    "confidence": 0.8842,
    "matchedLandmarks": 31,
    "offsetSeconds": 42.31,
    "sampleDurationSeconds": 8.0
  }
}
```

| 字段 | 说明 |
|------|------|
| `confidence` | 匹配置信度，范围 `0–1`；仅返回达到服务端阈值的结果 |
| `matchedLandmarks` | 对齐的声纹特征数量 |
| `offsetSeconds` | 录音片段在歌曲中的估计起始位置（秒） |
| `sampleDurationSeconds` | 服务端实际解码的录音时长 |

**未匹配（HTTP 200）:**

```json
{
  "success": true,
  "matched": false,
  "message": "未在当前曲库中识别到歌曲",
  "data": null
}
```

**错误响应:**

| HTTP 状态码 | 说明 |
|-------------|------|
| `400` | 缺少 `audio`、音频损坏、录音过短或无法解码 |
| `413` | 文件超过服务端配置的大小限制（默认 8 MiB） |
| `415` | 请求不是 `multipart/form-data` |
| `429` | 当前 IP 请求过于频繁，或识曲并发已满；可查看 `Retry-After` |
| `503` | 声纹索引正在构建或识曲服务暂时不可用 |

**曲库与索引说明:**

- 只匹配本站已入库且存在于 `Music/music/{id}.*` 的歌曲，不能识别未收录的全网歌曲。
- 首次请求或索引失效后会自动构建曲库索引，指纹缓存位于 `Music/.fingerprints/`。
- 音乐上传、替换或自动入库后会使索引失效，下一次识曲请求自动重建。
- 服务端配置位于 `backend/src/main/resources/config.yml` 的 `music_recognition` 节，可调整时长、大小、并发和限流。

### 6. 获取歌词

**端点:** `GET /api/music/lyrics/{id}`

**路径参数:**
- `id`: 音乐 ID

**响应示例:**
```json
{
  "success": true,
  "message": "获取歌词成功",
  "data": "歌词内容"
}
```

**说明:**
- 此 API **无需登录**即可访问

### 6. 获取播放次数排行榜

**端点:** `GET /api/music/ranking`

**无需登录**

**查询参数:**
- `limit`: 返回数量（可选，默认为 200，最大为 200）

**响应示例:**
```json
{
  "success": true,
  "message": "获取播放次数排行榜成功",
  "data": [
    {
      "id": 1,
      "title": "歌曲标题",
      "artist": "艺术家",
      "album": "专辑",
      "duration": 180,
      "language": "中文",
      "tags": "流行",
      "playCount": 100
    }
  ]
}
```

**说明:**
- 此 API **无需登录**即可访问
- 返回按播放次数从高到低排序的音乐列表
- 只返回播放次数大于 0 的音乐
- 默认返回前 200 首，最多支持返回 200 首
- 每首音乐包含：
  - id, title, artist, album, duration
  - language: 语言
  - tags: 标签
  - playCount: 播放次数

**使用场景:**
- 首页展示热门音乐
- 音乐排行榜页面
- 推荐热门音乐给用户

**前端集成示例:**
```javascript
async function getMusicRanking(limit = 200) {
  const response = await fetch(`https://music.cnmsb.xin/api/music/ranking?limit=${limit}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log(`获取到 ${data.data.length} 首热门音乐:`, data.data);
    // data.data 是一个数组，包含按播放次数排序的音乐
  } else {
    console.error('获取排行榜失败:', data.message);
  }
  return data;
}
```

### 7. 获取最新上传音乐

**端点:** `GET /api/music/latest`

**无需登录**

**查询参数:**
- `limit`: 返回数量（可选，默认为 300，最大为 500）

**响应示例:**
```json
{
  "success": true,
  "message": "获取最新音乐成功",
  "data": [
    {
      "id": 1,
      "title": "歌曲标题",
      "artist": "艺术家",
      "album": "专辑",
      "duration": 180,
      "language": "中文",
      "tags": "流行",
      "fileFormat": "mp3",
      "createdAt": 1704067200000
    }
  ]
}
```

**说明:**
- 此 API **无需登录**即可访问
- 返回按上传时间从新到旧排序的音乐列表
- 默认返回最新 300 首音乐，最多支持返回 500 首
- 每首音乐包含：
  - id, title, artist, album, duration
  - language: 语言
  - tags: 标签
  - fileFormat: 音频文件格式（mp3/flac/wav）
  - createdAt: 创建时间（Unix 时间戳，毫秒）

**使用场景:**
- 首页展示最新上线的音乐
- 新歌速递页面
- 推荐最新上传的音乐给用户

**前端集成示例:**
```javascript
async function getLatestMusic(limit = 300) {
  const response = await fetch(`https://music.cnmsb.xin/api/music/latest?limit=${limit}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log(`获取到 ${data.data.length} 首最新音乐:`, data.data);
    // data.data 是一个数组，包含按上传时间排序的音乐
  } else {
    console.error('获取最新音乐失败:', data.message);
  }
  return data;
}
```

---

## 分享视频渲染 API

将指定音乐渲染为 **1920×1080 横屏 MP4**（封面 + 波形 + 歌名/歌手，可选平台水印）。任务**异步**执行：创建接口立即返回 `jobId`，后台自动合成；**完成后向用户注册邮箱发送 HTML 通知**，内含下载链接。

### 权限与配额

| 用户类型 | 成片时长 | 水印 | 每日次数 |
|----------|----------|------|----------|
| 非 VIP | 最长 15 秒（从 `startSec` 起） | **必须**开启 | 10 次/自然日（东八区） |
| VIP | 从 `startSec` 至歌曲结束 | 可选（默认无水印） | 不限 |

- 非 VIP 若请求 `watermarked: false`，返回 **403**「非会员须开启水印」。
- 当日免费次数用尽返回 **429**。
- 渲染队列满返回 **503**；功能关闭返回 **503**「视频生成功能未启用」。

### 1. 创建渲染任务

**端点:** `POST /api/video/render/create`

**需要登录**

**请求头:**
```
Content-Type: application/json
Authorization: <token>
```

**请求体:**
```json
{
  "musicId": 1,
  "startSec": 0,
  "watermarked": true
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| musicId | number | 是 | 音乐 ID |
| startSec | number | 否 | 裁剪起点（秒），默认 0 |
| watermarked | boolean | 否 | 是否添加平台水印；VIP 默认 `false`，非 VIP 必须为 `true` |

- 水印为平台固定样式，**不能**通过本接口上传或指定自定义图案。

**成功响应:** HTTP **202 Accepted**
```json
{
  "success": true,
  "message": "任务已创建，完成后将邮件通知并附下载链接",
  "data": {
    "jobId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "pending",
    "isVip": false,
    "durationSec": 15,
    "watermarked": true,
    "musicId": 1,
    "remainingToday": 9
  }
}
```

- `remainingToday` 仅非 VIP 返回，表示今日剩余免费次数。
- 客户端提交后无需轮询占满界面；用户可通过邮件获取下载链接。

### 2. 查询任务状态

**端点:** `GET /api/video/render/{jobId}`

**需要登录**（仅能查询本人创建的任务）

**请求头:**
```
Authorization: <token>
```

**响应示例（渲染中）:**
```json
{
  "success": true,
  "data": {
    "jobId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "processing",
    "musicId": 1,
    "durationSec": 15,
    "watermarked": true
  }
}
```

**`status` 取值:** `pending` | `processing` | `done` | `failed`

**响应示例（已完成）:**
```json
{
  "success": true,
  "data": {
    "jobId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "done",
    "musicId": 1,
    "durationSec": 15,
    "watermarked": true,
    "downloadUrl": "/api/video/render/550e8400-e29b-41d4-a716-446655440000/download"
  }
}
```

**响应示例（失败）:**
```json
{
  "success": true,
  "data": {
    "jobId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "failed",
    "musicId": 1,
    "durationSec": 15,
    "watermarked": true,
    "error": "渲染失败原因摘要"
  }
}
```

### 3. 下载成片

**端点:** `GET /api/video/render/{jobId}/download`

**无需登录**

- 任务状态须为 `done`。
- 成功时返回 `video/mp4` 文件流，`Content-Disposition: attachment`。
- 未完成返回 **409**；任务或文件不存在返回 **404**。
- `jobId` 为 UUID，邮件中的链接形如：  
  `https://music.cnmsb.xin/api/video/render/{jobId}/download`  
  在浏览器或下载工具中直接打开即可，**不需要** Authorization 或 URL 参数 token。

### 4. 邮件通知

渲染成功后，系统向该用户**注册邮箱**发送 HTML 邮件（主题：`NekoMusic - 分享视频已生成`），内容包括：

- 歌曲名、艺术家、成片时长
- 下载按钮与完整 URL（同上 `/download` 地址）
- 若成片含水印，邮件中会注明

若未收到邮件，可先查看垃圾邮件箱；任务状态为 `done` 后，也可使用「查询任务」接口返回的 `downloadUrl` 或本页所述下载地址直接保存成片。

### 前端集成示例

```javascript
// 创建任务（需登录）
async function createVideoRenderJob(musicId, startSec = 0, watermarked = true) {
  const res = await fetch('https://music.cnmsb.xin/api/video/render/create', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: localStorage.getItem('userToken')
    },
    body: JSON.stringify({ musicId, startSec, watermarked })
  });
  const data = await res.json();
  if (!res.ok || !data.success) throw new Error(data.message);
  return data.data;
}

// 下载成片（无需登录）
async function downloadVideoClip(jobId, filename = 'clip.mp4') {
  const res = await fetch(`https://music.cnmsb.xin/api/video/render/${jobId}/download`);
  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.message || `下载失败 (${res.status})`);
  }
  const blob = await res.blob();
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}
```

---

## 错误码说明

| 状态码 | 说明 |
|--------|------|
| 200 | 请求成功 |
| 202 | 任务已接受（异步处理中） |
| 400 | 请求参数错误 |
| 401 | 未授权，需要登录或 Token 无效 |
| 403 | 禁止访问，权限不足 |
| 404 | 资源不存在 |
| 409 | 资源状态冲突（如视频尚未渲染完成） |
| 429 | 请求过于频繁或超出每日配额 |
| 503 | 服务不可用（功能关闭或渲染队列已满） |
| 500 | 服务器内部错误 |

### 错误响应格式

```json
{
  "success": false,
  "message": "错误描述"
}
```

---

## 前端集成示例

### 用户登录

```javascript
async function login(username, password) {
  const response = await fetch('https://music.cnmsb.xin/api/user/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ username, password })
  });
  
  const data = await response.json();
  if (data.success) {
    localStorage.setItem('userToken', data.data.token);
    localStorage.setItem('userInfo', JSON.stringify(data.data.user));
  }
  return data;
}
```

### 搜索音乐（模糊）

```javascript
async function searchMusic(query) {
  const response = await fetch('https://music.cnmsb.xin/api/music/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ query })
  });

  const data = await response.json();
  // data.results: Music[] | null
  // 封面/音频: `/api/music/cover/${id}`, `/api/music/file/${id}`
  return data;
}
```

### 批量精确搜索音乐

```javascript
async function searchMusicBatch(items) {
  const response = await fetch('https://music.cnmsb.xin/api/music/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      items: items // [{ title: '晴天', artist: '周杰伦' }, ...]
    })
  });

  const data = await response.json();
  // data.results.length === items.length
  // data.results[i] 对应 items[i]，未找到为 null
  return data;
}
```

### 获取收藏列表

```javascript
async function getFavorites() {
  const token = localStorage.getItem('userToken');
  const response = await fetch('https://music.cnmsb.xin/api/user/favorites', {
    method: 'GET',
    headers: {
      'Authorization': token
    }
  });
  
  return await response.json();
}
```

### 上传用户头像

```javascript
async function uploadAvatar(avatarFile) {
  const token = localStorage.getItem('userToken');
  const formData = new FormData();
  formData.append('avatar', avatarFile);
  
  const response = await fetch('https://music.cnmsb.xin/api/user/avatar/upload', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`
    },
    body: formData
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('头像上传成功:', data.avatarPath);
  } else {
    console.error('头像上传失败:', data.error);
  }
  return data;
}
```

### 搜索歌单

```javascript
async function searchPlaylists(query) {
  const response = await fetch('https://music.cnmsb.xin/api/playlists/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      query: query
    })
  });

  const data = await response.json();
  if (data.success) {
    console.log(`搜索到 ${data.total} 个歌单:`, data.results);
    // data.results 是一个数组，包含匹配的歌单
    // 每个歌单包含：
    // - id, name, description, musicCount, createdAt, updatedAt
    // - firstMusicId: 第一首音乐的 ID
    // - firstMusicCover: 第一首音乐的封面 URL
  } else {
    console.error('搜索歌单失败:', data.message);
  }
  return data;
}
```

### 搜索歌手

```javascript
async function searchArtists(query) {
  const response = await fetch('https://music.cnmsb.xin/api/artists/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      query: query
    })
  });

  const data = await response.json();
  if (data.success) {
    const artist = data.artist;
    console.log(`歌手: ${artist.name}`);
    console.log(`音乐数量: ${artist.musicCount}`);
    console.log(`音乐列表:`, artist.musicList);
    // artist.musicList 是一个数组，包含该歌手的所有音乐
    // 每首音乐包含：id, title, artist, album, duration, fileFormat, language
    // 封面/音频请用 id 拼接：/api/music/cover/{id}、/api/music/file/{id}
  } else {
    console.error('搜索歌手失败:', data.message);
  }
  return data;
}
```

### 获取歌单详情

```javascript
async function getPlaylistDetail(playlistId) {
  // 无需登录即可访问
  const response = await fetch(`https://music.cnmsb.xin/api/playlist/${playlistId}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log('歌单详情:', data.playlist);
    // data.playlist 包含歌单的基本信息
    // - id: 歌单 ID
    // - userId: 创建者 ID
    // - name: 歌单名称
    // - description: 歌单描述
    // - musicCount: 音乐数量
    // - createdAt: 创建时间
    // - updatedAt: 更新时间
  } else {
    console.error('获取歌单详情失败:', data.message);
  }
  return data;
}
```

### 创建歌单

```javascript
async function createPlaylist(name, description) {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch('https://music.cnmsb.xin/api/user/playlist/create', {
    method: 'POST',
    headers: {
      'Authorization': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: name,
      description: description
    })
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('歌单创建成功:', data.playlist);
  } else {
    console.error('歌单创建失败:', data.message);
  }
  return data;
}
```

### 获取歌单列表

```javascript
async function getPlaylists() {
  const token = localStorage.getItem('userToken');

  const response = await fetch('https://music.cnmsb.xin/api/user/playlists', {
    method: 'GET',
    headers: {
      'Authorization': token
    }
  });

  const data = await response.json();
  if (data.success) {
    console.log('获取到我的歌单列表:', data.playlists);
    // data.playlists 只包含当前用户创建的歌单
    // 不会混合其他用户的歌单
  } else {
    console.error('获取歌单列表失败:', data.message);
  }
  return data;
}
```

### 更新歌单

```javascript
async function updatePlaylist(playlistId, name, description) {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch('https://music.cnmsb.xin/api/user/playlist/update', {
    method: 'POST',
    headers: {
      'Authorization': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      id: playlistId,
      name: name,
      description: description
    })
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('歌单更新成功:', data.playlist);
  } else if (data.message === '无权限修改此歌单') {
    alert('您没有权限修改此歌单');
  } else {
    console.error('歌单更新失败:', data.message);
  }
  return data;
}
```

### 删除歌单

```javascript
async function deletePlaylist(playlistId) {
  const token = localStorage.getItem('userToken');

  const response = await fetch('https://music.cnmsb.xin/api/user/playlist/delete', {
    method: 'POST',
    headers: {
      'Authorization': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      id: playlistId
    })
  });

  const data = await response.json();
  if (data.success) {
    console.log('歌单删除成功');
    // 可以在这里刷新歌单列表
  } else if (data.message === '无权限删除此歌单') {
    alert('您没有权限删除此歌单');
  } else {
    console.error('歌单删除失败:', data.message);
  }
  return data;
}
```

### 获取歌单音乐列表

```javascript
async function getPlaylistMusic(playlistId) {
  // 无需登录即可访问
  const response = await fetch(`https://music.cnmsb.xin/api/user/playlist/music/${playlistId}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log(`获取到 ${data.total} 首音乐:`, data.musicList);
    // data.musicList 是一个数组，包含歌单中的所有音乐
  } else {
    console.error('获取歌单音乐列表失败:', data.message);
  }
  return data;
}
```

### 添加音乐到歌单

`musicIds` 可为单个数字或数字数组；请求体使用 `musicIds` 数组与后端批量语义一致（也可继续只传 `musicId`）。

```javascript
async function addMusicToPlaylist(playlistId, musicIds) {
  const token = localStorage.getItem('userToken');
  const ids = Array.isArray(musicIds) ? musicIds : [musicIds];

  const response = await fetch('https://music.cnmsb.xin/api/user/playlist/music/add', {
    method: 'POST',
    headers: {
      'Authorization': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      playlistId: playlistId,
      musicIds: ids
    })
  });

  const data = await response.json();
  if (data.success) {
    console.log('添加成功', data.addedCount, data.message);
  } else if (data.failedMusicIds) {
    console.warn('部分未添加', data.failedMusicIds, data.message);
  } else {
    console.error('添加失败:', data.message);
  }
  return data;
}
```

### 从歌单中移除音乐

`musicIds` 可为单个数字或数字数组；请求体使用 `musicIds` 数组与后端批量语义一致（也可继续只传 `musicId`）。

```javascript
async function removeMusicFromPlaylist(playlistId, musicIds) {
  const token = localStorage.getItem('userToken');
  const ids = Array.isArray(musicIds) ? musicIds : [musicIds];

  const response = await fetch('https://music.cnmsb.xin/api/user/playlist/music/remove', {
    method: 'POST',
    headers: {
      'Authorization': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      playlistId: playlistId,
      musicIds: ids
    })
  });

  const data = await response.json();
  if (data.success) {
    console.log(data.message || '音乐从歌单中移除成功', data.removedCount != null ? `removedCount=${data.removedCount}` : '');
    // 可以在这里刷新歌单内容
  } else if (data.message === '无权限修改此歌单') {
    alert('您没有权限修改此歌单');
  } else if (Array.isArray(data.failedMusicIds) && data.failedMusicIds.length) {
    console.error('部分音乐未能移除', data.failedMusicIds, 'removedCount=', data.removedCount);
    alert(data.message || '部分音乐未能从歌单中移除');
  } else {
    console.error('音乐从歌单中移除失败:', data.message);
  }
  return data;
}
```

### 修改用户密码

```javascript
async function changePassword(oldPassword, newPassword) {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch('https://music.cnmsb.xin/api/user/password/change', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      oldPassword: oldPassword,
      newPassword: newPassword
    })
  });
  
  const data = await response.json();
  if (data.success) {
    alert('密码修改成功！');
  } else {
    alert('密码修改失败：' + data.error);
  }
  return data;
}
```

### 获取收藏歌单列表

```javascript
async function getFavoritePlaylists() {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch('https://music.cnmsb.xin/api/user/favorite-playlists', {
    method: 'GET',
    headers: {
      'Authorization': token
    }
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('收藏歌单列表:', data.playlists);
    // data.playlists 包含用户收藏的所有歌单
    // 每个歌单包含：
    // - id, name, description, musicCount
    // - createdAt, updatedAt, favoriteTime (Unix时间戳)
    // - creator: 创建者信息 {id, username}
  } else {
    console.error('获取收藏歌单列表失败');
  }
  return data;
}
```

### 收藏歌单

```javascript
async function favoritePlaylist(playlistId) {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch('https://music.cnmsb.xin/api/user/favorite-playlists', {
    method: 'POST',
    headers: {
      'Authorization': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      playlistId: playlistId
    })
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('收藏歌单成功');
    // 可以在这里刷新收藏列表
  } else if (data.message.includes('已收藏')) {
    alert('您已经收藏过这个歌单了');
  } else {
    console.error('收藏歌单失败:', data.message);
  }
  return data;
}
```

### 取消收藏歌单

```javascript
async function unfavoritePlaylist(playlistId) {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch(`https://music.cnmsb.xin/api/user/favorite-playlists/${playlistId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': token
    }
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('取消收藏歌单成功');
    // 可以在这里刷新收藏列表
  } else {
    console.error('取消收藏歌单失败:', data.message);
  }
  return data;
}
```

### 获取收藏歌单内音乐

```javascript
async function getFavoritePlaylistMusic(playlistId) {
  const token = localStorage.getItem('userToken');
  
  const response = await fetch(`https://music.cnmsb.xin/api/user/favorite-playlists/${playlistId}`, {
    method: 'GET',
    headers: {
      'Authorization': token
    }
  });
  
  const data = await response.json();
  if (data.success) {
    console.log('歌单音乐列表:', data.music);
    // data.music 包含歌单内的所有音乐
    // 每首音乐包含：
    // - id, title, artist, album, duration, filename, position
  } else if (data.message === '用户未收藏该歌单') {
    alert('您未收藏该歌单，无法查看音乐');
  } else {
    console.error('获取歌单音乐失败:', data.message);
  }
  return data;
}
```

---

## 注意事项

1. **CORS:** 所有 API 都支持跨域请求
2. **Token 管理:** Token 有效期为 30 天，过期后需要重新登录
3. **错误处理:** 所有 API 都返回统一的 JSON 格式，包含 `success` 和 `message` 字段
4. **速率限制:** 建议客户端实现适当的请求速率限制，避免频繁请求
5. **头像上传:**
   - 支持的图片格式：jpg, jpeg, png, gif, webp, bmp
   - 最大文件大小：50MiB
   - 只允许图片类型文件上传，会严格验证 MIME 类型
   - 头像文件保存在 `avatars/` 目录下
6. **密码修改:**
   - 新密码长度不能少于 6 位
   - 新密码不能与原密码相同
   - 需要提供正确的原密码才能修改
   - 使用 Argon2 算法加密密码
7. **歌单管理:**
   - 每个歌单都有唯一的 ID 和创建者（user_id）
   - 任何用户（包括未登录用户）都可以搜索歌单、查看歌单详情和歌单内容
   - 任何登录用户可以查看自己创建的歌单列表（不会混合其他用户的歌单）
   - 只有歌单的创建者才能更新和删除歌单（通过 token 验证）
   - 删除歌单会级联删除歌单中的所有音乐关联
   - 歌单名称长度限制：255 个字符
   - 歌单描述长度限制：500 个字符
   - `musicCount` 字段会自动更新，无需手动维护
   - 歌单按创建时间倒序排列（最新的在前面）
   - 响应中包含 `userId` 字段，可以识别歌单的创建者
   - 歌单不包含封面字段，客户端应根据歌单中的音乐列表自动选择封面（如使用第一首音乐的封面）
   - 新增 API：`POST /api/playlist/{id}` - 获取歌单详情（无需登录）
   - 新增 API：`POST /api/playlists/search` - 搜索歌单（无需登录）
   - 新增 API：`POST /api/artists/search` - 搜索歌手（无需登录）
   - 获取歌单音乐列表 API 已移除登录要求，任何用户都可以访问
   - 获取歌单列表 API 只返回当前用户的歌单，不会混合其他用户的歌单
   - 搜索歌单 API 会返回歌单的第一首音乐封面 URL，方便客户端展示
   - 搜索歌单 API 使用 POST 方式，参数在请求体中传递
   - 搜索歌手 API 会返回匹配到的第一个歌手及其所有音乐列表
   - 搜索歌手 API 返回的音乐列表包含完整的音乐信息
   - 获取歌单列表 API 只返回当前用户的歌单，不会混合其他用户的歌单
   - 搜索歌单 API 会返回歌单的第一首音乐封面 URL，方便客户端展示
   - 搜索歌单 API 使用 POST 方式，参数在请求体中传递

8. **VIP 与会员:**
   - 用户登录响应 `data.user` 中含 `isVip`、`vipExpiresAt`（与歌单列表根级字段含义一致）。
   - `GET /api/user/playlists` 响应根级含 `isVip`、`vipExpiresAt`，便于未再次登录时刷新会员状态。
   - `GET /api/vip/pricing` 公开读取价目表（无需登录）。
   - `POST /api/vip/pay/create` 需要登录，用于根据 `pricingId` 发起 VIP 购买并返回收银台地址或二维码。
9. **忘记密码:**
   - `POST /api/user/send-reset-code` 向邮箱发送验证码。
   - `POST /api/user/reset-password` 验证码通过后重置密码。

10. **分享视频渲染:**
   - 创建任务与查询状态需要登录；下载成片 **无需登录**。
   - 非 VIP 必须 `watermarked: true`，否则 403；前端与后端均需校验。
   - 创建成功后后台异步渲染，完成后邮件通知（HTML）并附 `/api/video/render/{jobId}/download` 链接。
   - 建议前端：水印确认弹窗 → 提交后 toast 提示查收邮件，勿全屏阻塞轮询。

11. **收藏歌单:**
   - 用户可以收藏其他用户创建的歌单
   - 收藏歌单需要登录，通过 Authorization header 验证
   - 同一用户不能重复收藏同一个歌单
   - 只有收藏过该歌单的用户才能查看歌单内的音乐（权限控制）
   - 取消收藏歌单后，无法再查看该歌单内的音乐
   - 收藏歌单列表按收藏时间倒序排列（最新收藏的在前面）
   - 收藏歌单列表包含歌单的创建者信息
   - 新增 API：`GET /api/user/favorite-playlists` - 获取收藏歌单列表（需要登录）
   - 新增 API：`POST /api/user/favorite-playlists` - 收藏歌单（需要登录）
   - 新增 API：`DELETE /api/user/favorite-playlists/{id}` - 取消收藏歌单（需要登录）
   - 新增 API：`GET /api/user/favorite-playlists/{id}` - 获取收藏歌单内音乐（需要登录）

---

## 歌手相关 API

### 1. 搜索歌手

**端点:** `POST /api/artists/search`

**无需登录**

**请求头:**
```
Content-Type: application/json
```

**请求体:**
```json
{
  "query": "string"  // 搜索关键词
}
```

**响应示例（成功）:**
```json
{
  "success": true,
  "message": "搜索成功",
  "artist": {
    "name": "周杰伦",
    "musicCount": 50,
    "musicList": [
      {
        "id": 1,
        "title": "七里香",
        "artist": "周杰伦",
        "album": "七里香",
        "duration": 298,
        "fileFormat": "mp3",
        "language": "中文"
      },
      {
        "id": 2,
        "title": "晴天",
        "artist": "周杰伦",
        "album": "叶惠美",
        "duration": 269,
        "fileFormat": "mp3",
        "language": "中文"
      }
    ]
  }
}
```

**响应示例（未找到歌手）:**
```json
{
  "success": true,
  "message": "搜索成功",
  "artist": {
    "name": "",
    "musicCount": 0,
    "musicList": []
  }
}
```

**说明:**
- 此 API **无需登录**即可访问
- 搜索关键词会匹配歌手名称（模糊匹配）
- 返回匹配到的**第一个**歌手及其所有音乐
- 返回结果包含：
  - 歌手名称（name）
  - 该歌手的音乐数量（musicCount）
  - 该歌手的所有音乐列表（musicList），包含完整的音乐信息
- 如果没有找到匹配的歌手，返回空的歌手信息和空的音乐列表

**注意事项:**
- 搜索关键词不能为空
- 搜索是模糊匹配，使用 `LIKE %keyword%`
- 使用 POST 方式，参数在请求体中传递
- 只返回匹配到的第一个歌手（音乐数量最多的歌手）

**使用场景:**
- 用户搜索歌手以查看该歌手的所有音乐
- 展示歌手的音乐作品集
- 音乐分类浏览

---

## 联系方式

如有问题或建议，请联系：support@cnmsb.xin
