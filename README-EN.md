# Neko Music API Documentation

Chinese Documentation: [中文 API 文档](README.md)

### Using this API requires compliance with this project's LICENSE agreement. You must open source and retain the Neko Music attribution and source code link!

#### Last Updated(yyyy/mm/dd): 2026/5/24

## Overview

Neko Music provides a complete RESTful API supporting music search, playback, user authentication, favorites, landscape share-video rendering, and more. All APIs are based on HTTP/HTTPS protocol and use JSON format for data exchange.

**Base URL:** `https://music.cnmsb.xin`

## Table of Contents

- [Authentication](#authentication)
- [User APIs](#user-apis)
- [Slider captcha & registration email verification](API-captcha-slider-en.md)
- [Playlist APIs](#playlist-apis)
- [VIP & Pricing APIs](#vip--pricing-apis)
- [Artist APIs](#artist-apis)
- [Music APIs](#music-apis)
- [Share Video Render APIs](#share-video-render-apis)
- [Error Codes](#error-codes)

---

## Authentication

### User Authentication

All APIs that require user login must include the user Token in the request header:

```
Authorization: <token>
```

Token is generated and returned to the client when the user logs in.

**Token Validity:** 30 days

---

## User APIs

### 1. User Registration

**Endpoint:** `POST /api/user/register`

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "username": "string",      // Username (required)
  "password": "string",      // Password (required)
  "email": "string",         // Email (required)
  "verificationCode": "string"  // Email verification code (required)
}
```

**Response Example:**
```json
{
  "success": true,
  "message": "Registration successful",
  "data": {
    "user": {
      "id": 1,
      "username": "username",
      "email": "email@example.com",
      "createdAt": "2024-01-01T00:00:00"
    },
    "token": "64-character hexadecimal string"
  }
}
```

### 2. User Login

**Endpoint:** `POST /api/user/login`

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "username": "string",  // Username or email
  "password": "string"   // Password
}
```

**Response Example:**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "id": 1,
      "username": "username",
      "email": "email@example.com",
      "createdAt": "2024-01-01T00:00:00",
      "isVip": false,
      "vipExpiresAt": null
    },
    "token": "64-character hexadecimal string"
  }
}
```

### 3. Send Email Verification Code

Used before **sign-up** to send a numeric code to the mailbox. Requires a valid `captchaPassToken` from the slider verify step.

**Detailed doc:** [API-captcha-slider-en.md](API-captcha-slider-en.md) (`GET /api/captcha/slider`, `POST /api/captcha/slider/verify`, and this endpoint).

**Endpoint:** `POST /api/user/send-verification`

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "email": "string",
  "username": "string",
  "captchaPassToken": "string"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `email` | Yes | Mailbox that receives the code |
| `username` | No | Display name; server may default if omitted |
| `captchaPassToken` | Yes | One-time token from successful `POST /api/captcha/slider/verify` |

**Response Example:**
```json
{
  "success": true,
  "message": "Verification code sent to your email",
  "data": null
}
```

If captcha is missing or invalid, `success` is `false` with an explanatory `message`. Rate limiting may return HTTP `429` when enabled.

### 4. Send Reset Password Code

**Endpoint:** `POST /api/user/send-reset-code`

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "email": "string"
}
```

**Response Example (success):**
```json
{
  "success": true,
  "message": "Verification code sent to your email",
  "data": null
}
```

**Notes:** If the email is not registered, `success` is `false`, `message` indicates that (Chinese: 该邮箱未注册), and **no** verification email is sent.

### 5. Reset Password

**Endpoint:** `POST /api/user/reset-password`

**Request Body:**
```json
{
  "email": "string",
  "code": "string",
  "newPassword": "string"
}
```

- `newPassword`: 6–30 characters.

**Response Example (success):**
```json
{
  "success": true,
  "message": "Password reset successful, please log in with your new password",
  "data": null
}
```

### 6. Get User Avatar

**Endpoint:** `GET /api/user/avatar/{userId}`

**Path Parameters:**
- `userId`: User ID

**Response:** Image file (PNG/JPG)

### 7. Get Favorites List

**Endpoint:** `GET /api/user/favorites`

**Request Headers:**
```
Authorization: <token>
```

**Response Example:**
```json
{
  "success": true,
  "favorites": [
    {
      "id": 1,
      "title": "Song Title",
      "artist": "Artist",
      "album": "Album",
      "duration": 180,
      "filename": "song.mp3"
    }
  ]
}
```

### 8. Add to Favorites

**Endpoint:** `POST /api/user/favorites`

**Request Headers:**
```
Authorization: <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "musicId": 1  // Music ID
}
```

**Response Example:**
```json
{
  "success": true,
  "message": "Added to favorites successfully"
}
```

### 9. Remove from Favorites

**Endpoint:** `DELETE /api/user/favorites/{musicId}`

**Request Headers:**
```
Authorization: <token>
```

**Path Parameters:**
- `musicId`: Music ID

**Response Example:**
```json
{
  "success": true,
  "message": "Removed from favorites successfully"
}
```

### 10. Get Favorite Playlists List

**Endpoint:** `GET /api/user/favorite-playlists`

**Request Headers:**
```
Authorization: <token>
```

**Response Example:**
```json
{
  "success": true,
  "playlists": [
    {
      "id": 1,
      "name": "My Playlist",
      "description": "This is my playlist description",
      "musicCount": 5,
      "createdAt": 1706500800000,
      "updatedAt": 1706501100000,
      "favoriteTime": 1706501400000,
      "creator": {
        "id": 1,
        "username": "username"
      }
    }
  ]
}
```

**Notes:**
- `createdAt`, `updatedAt`, `favoriteTime` are Unix timestamps in milliseconds
- Only returns playlists favorited by the current user
- Sorted by favorite time in descending order

### 11. Favorite a Playlist

**Endpoint:** `POST /api/user/favorite-playlists`

**Request Headers:**
```
Authorization: <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "playlistId": 1  // Playlist ID
}
```

**Response Example:**
```json
{
  "success": true,
  "message": "Playlist favorited successfully"
}
```

**Response Example (Already Favorited):**
```json
{
  "success": false,
  "message": "Failed to favorite playlist or already favorited"
}
```

### 12. Unfavorite a Playlist

**Endpoint:** `DELETE /api/user/favorite-playlists/{playlistId}`

**Request Headers:**
```
Authorization: <token>
```

**Path Parameters:**
- `playlistId`: Playlist ID

**Response Example:**
```json
{
  "success": true,
  "message": "Playlist unfavorited successfully"
}
```

### 13. Get Music in Favorite Playlist

**Endpoint:** `GET /api/user/favorite-playlists/{playlistId}`

**Request Headers:**
```
Authorization: <token>
```

**Path Parameters:**
- `playlistId`: Playlist ID

**Response Example:**
```json
{
  "success": true,
  "music": [
    {
      "id": 1,
      "title": "Song Title",
      "artist": "Artist",
      "album": "Album",
      "duration": 180,
      "filename": "song.mp3",
      "position": 1
    }
  ]
}
```

**Response Example (Not Favorited):**
```json
{
  "success": false,
  "message": "User has not favorited this playlist"
}
```

**Notes:**
- Only users who have favorited the playlist can view the music inside
- Music is sorted by `position` field in ascending order

### 14. Upload User Avatar

**Endpoint:** `POST /api/user/avatar/upload`

**Request Headers:**
```
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**Request Parameters:**
- `avatar`: Image file (multipart/form-data)
  - Supported formats: jpg, jpeg, png, gif, webp, bmp
  - Maximum file size: 10MiB

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Avatar uploaded successfully",
  "avatarPath": "avatars/1_550e8400-e29b-41d4-a716-446655440000.jpg"
}
```

**Response Example (Failure):**
```json
{
  "error": "Unauthorized access"
}
```

Or

```json
{
  "error": "File size exceeds 10MiB limit"
}
```

### 15. User Upload Music

**Endpoint:** `POST /api/user/upload`

**Request Headers:**
```Mi B
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**Request Parameters:**
- `title`: Song title (required)
- `artist`: Artist name (required)
- `language`: Language (required)
  - Options: Chinese, Cantonese, Shanghainese, English, Japanese, Korean, French, German, Russian, Instrumental
- `album`: Album name (optional)
- `tags`: Tags (optional)
- `duration`: Music duration in seconds (required)
- `uploadUserId`: Uploader user ID (required)
- `musicFile`: Music file (required, multipart/form-data)
  - Supported formats: MP3, FLAC, WAV
- `coverFile`: Cover image file (optional, multipart/form-data)
  - Supported formats: jpg, jpeg, png, gif, webp, bmp
- `lyricsFile`: Lyrics file (optional, multipart/form-data)
  - Supported formats: lrc
  - If no lyrics file is provided, the system will automatically use a default empty lyrics file

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Upload successful, awaiting review",
  "data": {
    "id": 1,
    "status": "pending",
    "createdAt": "2026-02-12T16:30:00"
  }
}
```

**Response Example (Duplicate Music):**
```json
{
  "success": false,
  "message": "Duplicate music exists, please check and upload again"
}
```

**Response Example (Unauthorized):**
```json
{
  "success": false,
  "message": "Unauthorized access"
}
```

**Notes:**
- Uploaded music enters pending review status (`status: "pending"`)
- After admin approval, music will be officially added to the music library
- The system automatically checks for duplicate music (same title, artist, album)
- If no lyrics file is uploaded, the system automatically creates an empty lyrics file (no_lrc.lrc)
- Uploaded files are saved in the `user_upload/` directory with filename format `music_<timestamp>.<ext>`

**Important Notes:**
- Title, artist, and language are required fields
- Music file must be provided and in MP3, FLAC, or WAV format
- Music duration needs to be manually entered by the user or parsed and passed by the frontend
- Cover and lyrics files are optional
- After successful upload, users need to wait for admin review before the music appears in the music library

**Frontend Integration Example:**
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
    console.log('Music uploaded successfully, awaiting review');
    console.log('Upload record ID:', data.data.id);
    console.log('Status:', data.data.status);
  } else {
    console.error('Music upload failed:', data.message);
  }
  return data;
}

// Usage example
const formData = new FormData();
formData.append('title', 'Song Title');
formData.append('artist', 'Artist Name');
formData.append('language', 'Chinese');
formData.append('album', 'Album Name');
formData.append('tags', 'Pop,Chinese');
formData.append('duration', 235); // Music duration (seconds)
formData.append('uploadUserId', 0);
formData.append('musicFile', musicFileObject);
formData.append('coverFile', coverFileObject); // Optional
formData.append('lyricsFile', lyricsFileObject); // Optional

uploadMusic(formData);
```

### 16. Change User Password

**Endpoint:** `POST /api/user/password/change`

**Request Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "oldPassword": "string",  // Old password (required)
  "newPassword": "string"   // New password (required, must be at least 6 characters)
}
```

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Password changed successfully"
}
```

**Response Example (Failure):**
```json
{
  "error": "Old password is incorrect"
}
```

Or

```json
{
  "error": "New password must be at least 6 characters"
}
```

Or

```json
{
  "error": "New password cannot be the same as old password"
}
```

### 15. Get User Uploaded Approved Music

**Endpoint:** `GET /api/user/uploaded-music`

**Request Headers:**
```
Authorization: <token>
```

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Successfully retrieved list of user uploaded approved music",
  "userId": 123,
  "musicList": [
    {
      "id": 1,
      "title": "Song Title",
      "artist": "Artist",
      "album": "Album",
      "duration": 180,
      "language": "Chinese",
      "tags": "Pop,Chinese",
      "fileFormat": "mp3",
      "createdAt": "2026-02-16T10:00:00"
    }
  ],
  "total": 1
}
```

**Response Example (Not Logged In):**
```json
{
  "success": false,
  "message": "User not logged in or token invalid"
}
```

**Notes:**
- This API requires login to access
- Only returns music uploaded by the current user that has been approved
- Music is sorted by creation time in descending order (newest first)
- Only includes music with `approved` status
- Each music entry includes:
  - id, title, artist, album, duration
  - language, tags, fileFormat
  - createdAt: Creation time

**Use Cases:**
- Display user uploaded works in personal center
- User views their own music library
- Statistics on number of music uploaded by user

**Frontend Integration Example:**
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
    console.log(`Retrieved ${data.total} approved music:`, data.musicList);
    // data.musicList is an array containing user uploaded approved music
    // Each music entry includes complete song information
  } else {
    console.error('Failed to get user uploaded music:', data.message);
  }
  return data;
}
```

---

## Playlist APIs

- [Search Playlists](#1-search-playlists)
- [Get Playlist Details](#2-get-playlist-details)
- [Get Playlist Music List](#3-get-playlist-music-list)
- [Create Playlist](#4-create-playlist)
- [Get Playlists List](#5-get-playlists-list)
- [Update Playlist](#6-update-playlist)
- [Delete Playlist](#7-delete-playlist)
- [Add Music to Playlist](#8-add-music-to-playlist)
- [Remove Music from Playlist](#9-remove-music-from-playlist)
- [Playlist Permissions](#playlist-permissions)

### 1. Search Playlists

**Endpoint:** `POST /api/playlists/search`

**No login required**

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "query": "string"  // Search keyword
}
```

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Search successful",
  "total": 2,
  "results": [
    {
      "id": 1,
      "userId": 1,
      "name": "My Playlist",
      "description": "This is my playlist description",
      "musicCount": 5,
      "createdAt": "2026-01-29 12:00:00",
      "updatedAt": "2026-01-29 12:05:00",
      "firstMusicId": 1,
      "firstMusicCover": "/path/to/cover.jpg"
    },
    {
      "id": 2,
      "userId": 2,
      "name": "Pop Music",
      "description": "Collection of pop songs",
      "musicCount": 10,
      "createdAt": "2026-01-28 10:00:00",
      "updatedAt": "2026-01-28 10:00:00",
      "firstMusicCover": "/api/user/avatar/default"
    }
  ]
}
```

**Response Example (Failure):**
```json
{
  "success": false,
  "message": "Missing search keyword"
}
```

**Notes:**
- This API **does not require login** to access
- Search keyword matches playlist name and description
- Results are sorted by creation time in descending order
- Each result includes:
  - Basic playlist information (id, name, description, musicCount, createdAt, updatedAt)
  - First music ID (firstMusicId)
  - First music cover URL (firstMusicCover)
  - If playlist has no music, firstMusicCover is the default avatar `/api/user/avatar/default`

**Important Notes:**
- Search keyword cannot be empty
- Search uses fuzzy matching with `LIKE %keyword%`
- Uses POST method with parameters in request body

### 2. Get Playlist Details

**Endpoint:** `GET /api/playlist/{id}`

**Path Parameters:**
- `id`: Playlist ID

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Successfully retrieved playlist details",
  "playlist": {
    "id": 1,
    "userId": 1,
    "name": "My Playlist",
    "description": "This is my playlist description",
    "musicCount": 5,
    "createdAt": "2026-01-29 12:00:00",
    "updatedAt": "2026-01-29 12:05:00"
  }
}
```

**Response Example (Playlist Not Found):**
```json
{
  "success": false,
  "message": "Playlist not found"
}
```

**Notes:**
- This API **does not require login** to access
- Any user (including non-logged-in users) can view basic playlist information
- Returned playlist information includes `userId` field to identify the playlist creator

### 3. Create Playlist

**Endpoint:** `POST /api/user/playlist/create`

**Request Headers:**
```
Authorization: <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "string",          // Playlist name (required)
  "description": "string"   // Playlist description (optional)
}
```

**Parameter Limits:**
- `name`: Maximum 255 characters
- `description`: Maximum 500 characters

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Playlist created successfully",
  "playlist": {
    "id": 1,
    "name": "My Playlist",
    "description": "This is my playlist description",
    "musicCount": 0,
    "createdAt": "2026-01-29 12:00:00",
    "updatedAt": "2026-01-29 12:00:00"
  }
}
```

**Response Example (Failure):**
```json
{
  "success": false,
  "message": "Playlist name cannot be empty"
}
```

### 4. Get Playlists List

**Endpoint:** `GET /api/user/playlists`

**Request Headers:**
```
Authorization: <token>
```

**Response Example:**
```json
{
  "success": true,
  "message": "Successfully retrieved playlists list",
  "isVip": false,
  "vipExpiresAt": null,
  "playlists": [
    {
      "id": 1,
      "userId": 1,
      "name": "My Playlist",
      "description": "This is my playlist description",
      "musicCount": 5,
      "createdAt": "2026-01-29 12:00:00",
      "updatedAt": "2026-01-29 12:05:00"
    }
  ]
}
```

**Notes:** This API only returns playlists created by the current logged-in user. Playlists are sorted by creation time in descending order. Each user can only see their own playlists and will not see other users' playlists. The JSON **root** also includes membership fields `isVip` (boolean) and `vipExpiresAt` (VIP expiry as ISO-8601, or `null` if not VIP / no expiry), aligned with the login response user object.

**Important Notes:**
- This API requires login to access
- Only returns the current user's playlists, does not mix with other users' playlists
- To view other users' playlists, use `GET /api/playlist/{id}` API
- Playlists are sorted by creation time in descending order (newest first)

### 5. Update Playlist

**Endpoint:** `POST /api/user/playlist/update`

**Request Headers:**
```
Authorization: <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "id": 1,                   // Playlist ID (required)
  "name": "string",          // Playlist name (required)
  "description": "string"   // Playlist description (optional, omit or pass null to not modify)
}
```

**Parameter Description:**
- `id`: Playlist ID (required), must be a playlist created by the current user
- `name`: Playlist name (required), maximum 255 characters
- `description`: Playlist description (optional), maximum 500 characters
  - If you only want to modify the description, you still need to pass the `name` field (use current name)
  - If you don't want to modify the description, you can omit this field or pass `null`

**Use Cases:**
1. **Modify both name and description**: Pass all fields
2. **Modify only description**: Pass `id`, `name` (current value) and new `description`
3. **Modify only name**: Pass `id` and new `name`, omit `description`

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Playlist updated successfully",
  "playlist": {
    "id": 1,
    "name": "Updated Playlist Name",
    "description": "Updated description",
    "musicCount": 5,
    "createdAt": "2026-01-29 12:00:00",
    "updatedAt": "2026-01-29 12:10:00"
  }
}
```

**Response Example (Permission Error):**
```json
{
  "success": false,
  "message": "No permission to modify this playlist"
}
```

**Notes:** Only the playlist creator (matching user_id) can update playlist information.

### 6. Delete Playlist

**Endpoint:** `POST /api/user/playlist/delete`

**Request Headers:**
```
Authorization: <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "id": 1  // Playlist ID (required)
}
```

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Playlist deleted successfully"
}
```

**Response Example (Permission Error):**
```json
{
  "success": false,
  "message": "No permission to delete this playlist"
}
```

**Notes:**
- Only the playlist creator (matching user_id) can delete the playlist
- Deleting a playlist will cascade delete all associated records in the `playlist_music` table
- This operation cannot be undone

### 7. Get Playlist Music List

**Endpoint:** `GET /api/user/playlist/music/{playlistId}`

**Request Headers:**
```
Authorization: <token>
```

**Path Parameters:**
- `playlistId`: Playlist ID

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Successfully retrieved playlist music list",
  "playlistId": 1,
  "total": 5,
  "musicList": [
    {
      "id": 1,
      "title": "Song Title",
      "artist": "Artist",
      "album": "Album",
      "duration": 180,
      "coverPath": "/path/to/cover.jpg",
      "filePath": "/path/to/music.mp3",
      "fileFormat": "mp3",
      "language": "Chinese",
      "position": 1,
      "addedAt": "2026-01-29 12:00:00"
    }
  ]
}
```

**Response Example (Playlist Not Found):**
```json
{
  "success": false,
  "message": "Playlist not found"
}
```

**Notes:**
- Music list is sorted by `position` field in ascending order
- Returned music information includes complete song details and addition time
- This API **does not require login** to access (token verification has been removed from backend)
- Any user (including non-logged-in users) can view playlist content

### 8. Add Music to Playlist

**Endpoint:** `POST /api/user/playlist/music/add`

**Request Headers:**
```
Authorization: <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "playlistId": 1,  // Playlist ID (required)
  "musicId": 1      // Music ID (required)
}
```

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Music added to playlist successfully"
}
```

**Response Example (Failure):**
```json
{
  "success": false,
  "message": "Failed to add music to playlist or music already exists in playlist"
}
```

**Response Example (Permission Error):**
```json
{
  "success": false,
  "message": "No permission to modify this playlist"
}
```

**Notes:**
- Only the playlist creator can add music to the playlist
- If music already exists in the playlist, it will return failure
- Music is automatically added to the **top** of the playlist (position = 1, other music positions + 1)
- After successful addition, the playlist's `musicCount` field is automatically updated

### 9. Remove Music from Playlist

**Endpoint:** `POST /api/user/playlist/music/remove`

**Request Headers:**
```
Authorization: <token>
Content-Type: application/json
```

**Request Body:**

`playlistId` is required. Music IDs can be supplied in either or both of the following ways (values are **merged**, then **deduplicated** while preserving first-seen order, then removed one by one):

- `musicId`: a single music ID (backward compatible with older clients)
- `musicIds`: an array of integers to remove multiple tracks in one request

After merge and deduplication there must be **at least one** music ID, otherwise the API returns `400`.

```json
{
  "playlistId": 1,
  "musicId": 1
}
```

Batch example:

```json
{
  "playlistId": 1,
  "musicIds": [1, 2, 3]
}
```

You may send both `musicId` and `musicIds`; duplicate IDs are only removed once.

**Response Example (Success, single track):**
```json
{
  "success": true,
  "removedCount": 1,
  "message": "音乐从歌单中移除成功"
}
```

**Response Example (Success, multiple tracks):**
```json
{
  "success": true,
  "removedCount": 3,
  "message": "已从歌单中移除 3 首音乐"
}
```

*(The `message` field is returned in Chinese by the server.)*

**Response Example (Partial failure, HTTP 400):**  
If some IDs are not in the playlist or cannot be removed, tracks already removed in the same request stay removed. The response lists IDs that failed.

```json
{
  "success": false,
  "removedCount": 1,
  "failedMusicIds": [99, 100],
  "message": "部分音乐未能从歌单中移除（不在歌单中或移除失败），失败数量: 2"
}
```

**Response Example (Permission Error):**
```json
{
  "success": false,
  "message": "无权限修改此歌单"
}
```

**Notes:**
- Only the playlist creator can remove music from the playlist
- If any requested ID is not in the playlist or removal fails, the API returns `400` with `failedMusicIds`; removals that already succeeded in the same request are **not** rolled back
- `musicIds` must be a JSON array; otherwise `400` is returned with an error message
- After removal, remaining tracks are reordered (`position` of items after the deleted slot is decremented). For multiple IDs, removal runs in deduplicated order, equivalent to calling single-track removal repeatedly
- After successful removal(s), the playlist's `musicCount` field is updated automatically

---

## Playlist Permissions

### Permission Rules

| Operation | Permission Requirement |
|-----------|----------------------|
| Search Playlists | No login required (any user can search) |
| Get Playlist Details | No login required (any user can view) |
| View Playlists List | Any logged-in user (only returns their own playlists) |
| View Playlist Content | No login required (any user can view music in playlist) |
| Create Playlist | Any logged-in user |
| Update Playlist | Only playlist creator |
| Delete Playlist | Only playlist creator |
| Add Music to Playlist | Only playlist creator |
| Remove Music from Playlist | Only playlist creator |

### Permission Verification

- **View Permissions**: All users (including non-logged-in users) can search playlists, view playlist details and playlist content without additional permission verification
- **Playlist List**: Logged-in users can only see their own playlists and will not see other users' playlists
- **Modify Permissions**: All modification and deletion operations verify if the user is the playlist creator (identify user identity through token)
- If a user attempts to modify or delete a playlist that doesn't belong to them, the server will return `403 Forbidden` status code and corresponding error message

---

## VIP & Pricing APIs

Public endpoints are for **displaying** current membership packages and prices only. Purchase and payment are done on the site’s VIP page; payment gateways, callbacks, and admin maintenance APIs are not documented here.

### 1. Get VIP pricing (no login)

**Endpoint:** `GET /api/vip/pricing`

**Response Example:**
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

**Fields:**

| Field | Description |
|-------|-------------|
| `months` / `days` | Package duration (months + days); at least one must be &gt; 0 |
| `priceYuan` | Price in CNY (yuan) |
| `sortOrder` | Display order (lower values first; server-defined) |
| `updatedAt` | Last update time (Asia/Shanghai, ISO-8601 with offset) |

**Membership fields (client):** Login `data.user` and `GET /api/user/playlists` root both include `isVip` and `vipExpiresAt` with the same meaning.

---

## Music APIs

### 1. Search Music

**Endpoint:** `POST /api/music/search`

**No login required**

**Request Headers:**
```
Content-Type: application/json
```

Two modes on the same endpoint. Use **`query` OR `items`**, not both.

#### Mode A: Fuzzy search (single keyword)

**Request Body:**
```json
{
  "query": "晴天"
}
```

**Notes:**
- Fuzzy match on title, artist, album, and pinyin columns; up to ~50 results by relevance
- If local DB has no match and `netease_search_fill` is enabled, may ingest **one** track from NetEase

**Response Example (found):**
```json
{
  "success": true,
  "message": "Search successful",
  "results": [
    {
      "id": 1,
      "title": "晴天",
      "artist": "周杰伦",
      "album": "叶惠美",
      "duration": 269,
      "uploadUserId": 0,
      "createdAt": "2024-01-01 12:00:00.0"
    }
  ]
}
```

**Response Example (not found):**
```json
{
  "success": false,
  "message": "No matching music found",
  "results": null
}
```

#### Mode B: Batch exact search (title + artist)

**Request Body:**
```json
{
  "items": [
    { "title": "晴天", "artist": "周杰伦" },
    { "title": "STAY WIT ME", "artist": "TRYBEL BAND" },
    { "title": "Title only" }
  ]
}
```

| Field | Description |
|-------|-------------|
| `items` | Required; length unlimited; `results[i]` maps to `items[i]` |
| `items[].title` | Required, non-empty after trim |
| `items[].artist` | Optional; exact title+artist match when present |

**Match rules:**
- With `artist`: exact title and artist (simplified Chinese normalized); multiple hits → highest `id`
- Without `artist`: exact title only if **unique** artist for that title; else `null` for that slot
- Per-item NetEase fill when local miss and fill is enabled

**Response Example (partial hits):**
```json
{
  "success": true,
  "message": "Search successful (2/3 found, 1 ingested from NetEase)",
  "results": [
    { "id": 1, "title": "晴天", "artist": "周杰伦", "album": "叶惠美", "duration": 269, "uploadUserId": 0, "createdAt": "2024-01-01 12:00:00.0" },
    null,
    { "id": 42, "title": "STAY WIT ME", "artist": "TRYBEL BAND", "album": "Unknown", "duration": 200, "uploadUserId": 0, "createdAt": "2026-05-24 10:00:00.0" }
  ]
}
```

#### Music object fields (both modes)

| Field | Type | Description |
|-------|------|-------------|
| `id` | number | Music ID |
| `title` | string | Title |
| `artist` | string | Artist |
| `album` | string | Album |
| `duration` | number | Duration in seconds |
| `uploadUserId` | number | Uploader user ID, 0 if none |
| `createdAt` | string | Created timestamp |

**Media URLs (client builds from `id`; no path fields in response):**
- Audio: `GET /api/music/file/{id}`
- Cover: `GET /api/music/cover/{id}`

**Bad request (400):**
```json
{
  "error": "Request format error: query and items cannot be provided together"
}
```

### 2. Get Music Information

**Endpoint:** `GET /api/music/info/{id}`

**Path Parameters:**
- `id`: Music ID

**Response Example:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "Song Title",
    "artist": "Artist",
    "album": "Album",
    "duration": 180,
    "coverUrl": "/api/music/cover/1",
    "fileUrl": "/api/music/file/1",
    "lyrics": "Lyrics content"
  }
}
```

### 3. Get Music File

**Endpoint:** `GET /api/music/file/{id}`

**Path Parameters:**
- `id`: Music ID

**Response:** Audio file (MP3)

### 4. Get Music Cover

**Endpoint:** `GET /api/music/cover/{id}`

**Path Parameters:**
- `id`: Music ID

**Response:** Image file (PNG/JPG)

### 5. Get Lyrics

**Endpoint:** `GET /api/music/lyrics/{id}`

**Path Parameters:**
- `id`: Music ID

**Response Example:**
```json
{
  "success": true,
  "message": "Successfully retrieved lyrics",
  "data": "Lyrics content"
}
```

**Notes:**
- This API **does not require login** to access

### 6. Get Play Count Ranking

**Endpoint:** `GET /api/music/ranking`

**No login required**

**Query Parameters:**
- `limit`: Number of results (optional, default is 200, maximum is 200)

**Response Example:**
```json
{
  "success": true,
  "message": "Successfully retrieved play count ranking",
  "data": [
    {
      "id": 1,
      "title": "Song Title",
      "artist": "Artist",
      "album": "Album",
      "duration": 180,
      "language": "中文",
      "tags": "Pop",
      "playCount": 100
    }
  ]
}
```

**Notes:**
- This API **does not require login** to access
- Returns music list sorted by play count from high to low
- Only returns music with play count greater than 0
- Returns top 200 by default, maximum supports 200 results
- Each music entry includes:
  - id, title, artist, album, duration
  - coverPath: Cover file path
  - coverUrl: Cover access URL (default icon if cover doesn't exist)
  - language: Language
  - tags: Tags
  - playCount: Play count

**Use Cases:**
- Display popular music on homepage
- Music ranking page
- Recommend popular music to users

**Frontend Integration Example:**
```javascript
async function getMusicRanking(limit = 200) {
  const response = await fetch(`https://music.cnmsb.xin/api/music/ranking?limit=${limit}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log(`Retrieved ${data.data.length} popular music:`, data.data);
    // data.data is an array containing music sorted by play count
  } else {
    console.error('Failed to get ranking:', data.message);
  }
  return data;
}
```

### 7. Get Latest Uploaded Music

**Endpoint:** `GET /api/music/latest`

**No login required**

**Query Parameters:**
- `limit`: Number of results (optional, default is 300, maximum is 500)

**Response Example:**
```json
{
  "success": true,
  "message": "Successfully retrieved latest music",
  "data": [
    {
      "id": 1,
      "title": "Song Title",
      "artist": "Artist",
      "album": "Album",
      "duration": 180,
      "language": "中文",
      "tags": "Pop",
      "fileFormat": "mp3",
      "createdAt": 1704067200000
    }
  ]
}
```

**Notes:**
- This API **does not require login** to access
- Returns music list sorted by upload time from new to old
- Returns latest 300 music by default, maximum supports 500 results
- Each music entry includes:
  - id, title, artist, album, duration
  - coverPath: Cover file path
  - coverUrl: Cover access URL (default icon if cover doesn't exist)
  - language: Language
  - tags: Tags
  - fileFormat: Audio file format (mp3/flac/wav)
  - createdAt: Creation time (Unix timestamp, milliseconds)

**Use Cases:**
- Display latest released music on homepage
- New music page
- Recommend latest uploaded music to users

**Frontend Integration Example:**
```javascript
async function getLatestMusic(limit = 300) {
  const response = await fetch(`https://music.cnmsb.xin/api/music/latest?limit=${limit}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log(`Retrieved ${data.data.length} latest music:`, data.data);
    // data.data is an array containing music sorted by upload time
  } else {
    console.error('Failed to get latest music:', data.message);
  }
  return data;
}
```

---

## Share Video Render APIs

Renders a track into a **1920×1080 landscape MP4** (cover art, waveform, title/artist, optional platform watermark). Jobs run **asynchronously**: the create endpoint returns a `jobId` immediately; the server renders in the background. When finished, an **HTML email** is sent to the user’s registered address with a download link.

### Quotas & rules

| User | Clip length | Watermark | Daily limit |
|------|-------------|-----------|-------------|
| Non-VIP | Up to 15s from `startSec` | **Required** | 10 / calendar day (UTC+8) |
| VIP | From `startSec` to end of track | Optional (default off) | Unlimited |

- Non-VIP requests with `watermarked: false` → **403** “Non-members must enable watermark”.
- Daily free quota exceeded → **429**.
- Render queue full → **503**; feature disabled → **503** “Video rendering is not enabled”.

### 1. Create render job

**Endpoint:** `POST /api/video/render/create`

**Login required**

**Headers:**
```
Content-Type: application/json
Authorization: <token>
```

**Body:**
```json
{
  "musicId": 1,
  "startSec": 0,
  "watermarked": true
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| musicId | number | Yes | Music ID |
| startSec | number | No | Trim start (seconds), default 0 |
| watermarked | boolean | No | Platform watermark; VIP default `false`, non-VIP must be `true` |

- Watermark appearance is fixed by the platform; **cannot** upload or pick a custom image via this API.

**Success:** HTTP **202 Accepted**
```json
{
  "success": true,
  "message": "Job created; you will receive an email with the download link when finished",
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

- `remainingToday` is returned for non-VIP only.
- Clients should not block the UI polling; users can download from the email link.

### 2. Query job status

**Endpoint:** `GET /api/video/render/{jobId}`

**Login required** (own jobs only)

**Headers:**
```
Authorization: <token>
```

**In progress:**
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

**`status`:** `pending` | `processing` | `done` | `failed`

**Completed:**
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

**Failed:**
```json
{
  "success": true,
  "data": {
    "jobId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "failed",
    "musicId": 1,
    "durationSec": 15,
    "watermarked": true,
    "error": "Short error summary"
  }
}
```

### 3. Download MP4

**Endpoint:** `GET /api/video/render/{jobId}/download`

**No login required**

- Job must be `done`.
- Success: `video/mp4` stream, `Content-Disposition: attachment`.
- Not ready → **409**; missing job/file → **404**.
- Email link example:  
  `https://music.cnmsb.xin/api/video/render/{jobId}/download`  
  Open directly in a browser or downloader — **no** Authorization header or URL token.

### 4. Email notification

On success, an HTML email is sent (subject: `NekoMusic - 分享视频已生成`) with song title, artist, duration, and the download URL above. Watermarked clips are noted in the email.

If you do not receive the email, check spam; once the job is `done`, you can also use the `downloadUrl` from the status API or the download URL described above.

### Frontend example

```javascript
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

async function downloadVideoClip(jobId, filename = 'clip.mp4') {
  const res = await fetch(`https://music.cnmsb.xin/api/video/render/${jobId}/download`);
  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.message || `Download failed (${res.status})`);
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

## Error Codes

| Status Code | Description |
|-------------|-------------|
| 200 | Request successful |
| 202 | Accepted (async job queued) |
| 400 | Request parameter error |
| 401 | Unauthorized, login required or Token invalid |
| 403 | Forbidden, insufficient permissions |
| 404 | Resource not found |
| 409 | Conflict (e.g. video not finished yet) |
| 429 | Too many requests or daily quota exceeded |
| 503 | Service unavailable (feature off or render queue full) |
| 500 | Internal server error |

### Error Response Format

```json
{
  "success": false,
  "message": "Error description"
}
```

---

## Frontend Integration Examples

### User Login

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

### Search Music (fuzzy)

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
  // Cover/audio: `/api/music/cover/${id}`, `/api/music/file/${id}`
  return data;
}
```

### Batch exact search

```javascript
async function searchMusicBatch(items) {
  const response = await fetch('https://music.cnmsb.xin/api/music/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ items })
  });

  const data = await response.json();
  // data.results.length === items.length; null slot = not found
  return data;
}
```

### Get Favorites List

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

### Upload User Avatar

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
    console.log('Avatar uploaded successfully:', data.avatarPath);
  } else {
    console.error('Avatar upload failed:', data.error);
  }
  return data;
}
```

### Search Playlists

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
    console.log(`Found ${data.total} playlists:`, data.results);
    // data.results is an array containing matching playlists
    // Each playlist includes:
    // - id, name, description, musicCount, createdAt, updatedAt
    // - firstMusicId: First music ID
    // - firstMusicCover: First music cover URL
  } else {
    console.error('Search playlists failed:', data.message);
  }
  return data;
}
```

### Search Artists

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
    console.log(`Artist: ${artist.name}`);
    console.log(`Music count: ${artist.musicCount}`);
    console.log(`Music list:`, artist.musicList);
    // artist.musicList is an array containing all music by this artist
    // Each music: id, title, artist, album, duration, fileFormat, language
    // Cover/audio: /api/music/cover/{id}, /api/music/file/{id}
  } else {
    console.error('Search artist failed:', data.message);
  }
  return data;
}
```

### Get Playlist Details

```javascript
async function getPlaylistDetail(playlistId) {
  // No login required
  const response = await fetch(`https://music.cnmsb.xin/api/playlist/${playlistId}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log('Playlist details:', data.playlist);
    // data.playlist contains basic playlist information
    // - id: Playlist ID
    // - userId: Creator ID
    // - name: Playlist name
    // - description: Playlist description
    // - musicCount: Number of music
    // - createdAt: Creation time
    // - updatedAt: Update time
  } else {
    console.error('Failed to get playlist details:', data.message);
  }
  return data;
}
```

### Create Playlist

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
    console.log('Playlist created successfully:', data.playlist);
  } else {
    console.error('Playlist creation failed:', data.message);
  }
  return data;
}
```

### Get Playlists List

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
    console.log('Retrieved my playlists list:', data.playlists);
    // data.playlists only contains playlists created by the current user
    // Will not mix with other users' playlists
  } else {
    console.error('Failed to get playlists list:', data.message);
  }
  return data;
}
```

### Update Playlist

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
    console.log('Playlist updated successfully:', data.playlist);
  } else if (data.message === 'No permission to modify this playlist') {
    alert('You do not have permission to modify this playlist');
  } else {
    console.error('Playlist update failed:', data.message);
  }
  return data;
}
```

### Delete Playlist

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
    console.log('Playlist deleted successfully');
    // You can refresh the playlist list here
  } else if (data.message === 'No permission to delete this playlist') {
    alert('You do not have permission to delete this playlist');
  } else {
    console.error('Playlist deletion failed:', data.message);
  }
  return data;
}
```

### Get Playlist Music List

```javascript
async function getPlaylistMusic(playlistId) {
  // No login required
  const response = await fetch(`https://music.cnmsb.xin/api/user/playlist/music/${playlistId}`, {
    method: 'GET'
  });

  const data = await response.json();
  if (data.success) {
    console.log(`Retrieved ${data.total} music:`, data.musicList);
    // data.musicList is an array containing all music in the playlist
  } else {
    console.error('Failed to get playlist music list:', data.message);
  }
  return data;
}
```

### Add Music to Playlist

```javascript
async function addMusicToPlaylist(playlistId, musicId) {
  const token = localStorage.getItem('userToken');

  const response = await fetch('https://music.cnmsb.xin/api/user/playlist/music/add', {
    method: 'POST',
    headers: {
      'Authorization': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      playlistId: playlistId,
      musicId: musicId
    })
  });

  const data = await response.json();
  if (data.success) {
    console.log('Music added to playlist successfully');
    // You can refresh the playlist content here
  } else if (data.message === 'No permission to modify this playlist') {
    alert('You do not have permission to modify this playlist');
  } else if (data.message.includes('music already exists in playlist')) {
    alert('Music already exists in playlist');
  } else {
    console.error('Failed to add music to playlist:', data.message);
  }
  return data;
}
```

### Remove Music from Playlist

Use a `musicIds` array for batch removal (you can still send a single `musicId` for backward compatibility). The example below normalizes one or many IDs into an array.

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
    console.log(data.message || 'Removed from playlist', data.removedCount != null ? `removedCount=${data.removedCount}` : '');
    // You can refresh the playlist content here
  } else if (data.message === '无权限修改此歌单') {
    alert('You do not have permission to modify this playlist');
  } else if (Array.isArray(data.failedMusicIds) && data.failedMusicIds.length) {
    console.error('Some tracks failed to remove', data.failedMusicIds, 'removedCount=', data.removedCount);
    alert(data.message || 'Partial removal failed');
  } else {
    console.error('Failed to remove music from playlist:', data.message);
  }
  return data;
}
```

### Change User Password

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
    alert('Password changed successfully!');
  } else {
    alert('Password change failed: ' + data.error);
  }
  return data;
}
```

### Get Favorite Playlists List

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
    console.log('Favorite playlists list:', data.playlists);
    // data.playlists contains all playlists favorited by the user
    // Each playlist includes:
    // - id, name, description, musicCount
    // - createdAt, updatedAt, favoriteTime (Unix timestamp)
    // - creator: Creator information {id, username}
  } else {
    console.error('Failed to get favorite playlists list');
  }
  return data;
}
```

### Favorite a Playlist

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
    console.log('Playlist favorited successfully');
    // You can refresh the favorites list here
  } else if (data.message.includes('already favorited')) {
    alert('You have already favorited this playlist');
  } else {
    console.error('Failed to favorite playlist:', data.message);
  }
  return data;
}
```

### Unfavorite a Playlist

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
    console.log('Playlist unfavorited successfully');
    // You can refresh the favorites list here
  } else {
    console.error('Failed to unfavorite playlist:', data.message);
  }
  return data;
}
```

### Get Music in Favorite Playlist

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
    console.log('Playlist music list:', data.music);
    // data.music contains all music in the playlist
    // Each music includes:
    // - id, title, artist, album, duration, filename, position
  } else if (data.message === 'User has not favorited this playlist') {
    alert('You have not favorited this playlist and cannot view the music');
  } else {
    console.error('Failed to get playlist music:', data.message);
  }
  return data;
}
```

---

## Important Notes

1. **CORS:** All APIs support cross-origin requests
2. **Token Management:** Token validity is 30 days, after which re-login is required
3. **Error Handling:** All APIs return a unified JSON format containing `success` and `message` fields
4. **Rate Limiting:** It is recommended that clients implement appropriate request rate limiting to avoid frequent requests
5. **Avatar Upload:**
   - Supported image formats: jpg, jpeg, png, gif, webp, bmp
   - Maximum file size: 50MiB
   - Only image type files are allowed for upload, MIME type is strictly verified
   - Avatar files are saved in the `avatars/` directory
6. **Password Change:**
   - New password must be at least 6 characters
   - New password cannot be the same as the old password
   - Correct old password must be provided to change
   - Passwords are encrypted using Argon2 algorithm
7. **Playlist Management:**
   - Each playlist has a unique ID and creator (user_id)
   - Any user (including non-logged-in users) can search playlists, view playlist details and playlist content
   - Any logged-in user can view their own created playlists list (will not mix with other users' playlists)
   - Only the playlist creator can update and delete the playlist (verified through token)
   - Deleting a playlist will cascade delete all music associations in the playlist
   - Playlist name length limit: 255 characters
   - Playlist description length limit: 500 characters
   - `musicCount` field is automatically updated, no manual maintenance needed
   - Playlists are sorted by creation time in descending order (newest first)
   - Response includes `userId` field to identify the playlist creator
   - Playlists do not include a cover field, clients should automatically select a cover based on the music list in the playlist (e.g., use the cover of the first music)
   - New API: `GET /api/playlist/{id}` - Get playlist details (no login required)
   - New API: `POST /api/playlists/search` - Search playlists (no login required)
   - New API: `POST /api/artists/search` - Search artists (no login required)
   - Get playlist music list API has removed login requirement, any user can access it
   - Get playlists list API only returns current user's playlists, will not mix with other users' playlists
   - Search playlists API returns the first music cover URL in the playlist for easy client display
   - Search playlists API uses POST method with parameters in request body
   - Search artists API returns the first matched artist and their complete music list
   - Search artists API returned music list includes complete music information
   - Get playlists list API only returns current user's playlists, will not mix with other users' playlists
   - Search playlists API returns the first music cover URL in the playlist for easy client display
   - Search playlists API uses POST method with parameters in request body

8. **VIP & membership:**
   - Login response `data.user` includes `isVip` and `vipExpiresAt` (same meaning as playlist list root fields).
   - `GET /api/user/playlists` root includes `isVip` and `vipExpiresAt` for refreshing membership without logging in again.
   - `GET /api/vip/pricing` is public (no login); payment and admin APIs are not in this public doc.
9. **Forgot password:**
   - `POST /api/user/send-reset-code`, `POST /api/user/reset-password`.

10. **Share video rendering:**
   - Create and status APIs require login; download **does not**.
   - Non-VIP must send `watermarked: true` (403 otherwise); validate on client and server.
   - After submit, rendering runs in the background; HTML email includes `/api/video/render/{jobId}/download`.
   - UI: watermark confirm dialog → toast to check email; avoid full-screen polling.

11. **Favorite Playlists:**
   - Users can favorite playlists created by other users
   - Favoriting a playlist requires login, verified through Authorization header
   - The same user cannot favorite the same playlist multiple times
   - Only users who have favorited the playlist can view the music inside (access control)
   - After unfavoriting a playlist, you can no longer view the music inside that playlist
   - Favorite playlists list is sorted by favorite time in descending order (newest favorited first)
   - Favorite playlists list includes the playlist creator information
   - New API: `GET /api/user/favorite-playlists` - Get favorite playlists list (login required)
   - New API: `POST /api/user/favorite-playlists` - Favorite a playlist (login required)
   - New API: `DELETE /api/user/favorite-playlists/{id}` - Unfavorite a playlist (login required)
   - New API: `GET /api/user/favorite-playlists/{id}` - Get music in favorite playlist (login required)

---

## Artist APIs

### 1. Search Artists

**Endpoint:** `POST /api/artists/search`

**No login required**

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "query": "string"  // Search keyword
}
```

**Response Example (Success):**
```json
{
  "success": true,
  "message": "Search successful",
  "artist": {
    "name": "Jay Chou",
    "musicCount": 50,
    "musicList": [
      {
        "id": 1,
        "title": "Qi Li Xiang",
        "artist": "Jay Chou",
        "album": "Qi Li Xiang",
        "duration": 298,
        "fileFormat": "mp3",
        "language": "Chinese"
      },
      {
        "id": 2,
        "title": "Qing Tian",
        "artist": "Jay Chou",
        "album": "Ye Hui Mei",
        "duration": 269,
        "fileFormat": "mp3",
        "language": "Chinese"
      }
    ]
  }
}
```

**Response Example (Artist Not Found):**
```json
{
  "success": true,
  "message": "Search successful",
  "artist": {
    "name": "",
    "musicCount": 0,
    "musicList": []
  }
}
```

**Notes:**
- This API **does not require login** to access
- Search keyword matches artist name (fuzzy matching)
- Returns the **first** matched artist and all their music
- Returned result includes:
  - Artist name (name)
  - Number of music by this artist (musicCount)
  - All music list by this artist (musicList), including complete music information
- If no matching artist is found, returns empty artist information and empty music list

**Important Notes:**
- Search keyword cannot be empty
- Search uses fuzzy matching with `LIKE %keyword%`
- Uses POST method with parameters in request body
- Only returns the first matched artist (artist with the most music)
- Music list does not include `filePath` or `coverPath`; use `id` with `/api/music/cover/{id}` and `/api/music/file/{id}`

**Use Cases:**
- Users search for artists to view all their music
- Display artist's music collection
- Browse music by category

---

## Contact

For questions or suggestions, please contact: support@cnmsb.xin