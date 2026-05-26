---
layout: default
title: Face API Docs
---

# 🔐 顔認証 API
### 複数システム認可プラットフォーム
##### Face Authentication API — Multi-System Authorization Platform

![Version](https://img.shields.io/badge/バージョン-1.0.0-6366f1?style=flat-square)
![Auth](https://img.shields.io/badge/認証-顔認証-10b981?style=flat-square)
![JWT](https://img.shields.io/badge/トークン-JWT-f59e0b?style=flat-square)
![Liveness](https://img.shields.io/badge/ライブネス-まばたき＋顔向き-ef4444?style=flat-square)

---

# 1. システム概要 / System Overview

このドキュメントは、顔認証ログインおよび複数システム認可の全体仕様について説明します。

This document describes the overall specification of the Face Authentication Login and Multi-System Authorization platform.

## 対象 / Scope

- 顔認証ログイン / Face authentication login
- ライブネス検知（まばたき・顔向き確認）/ Liveness detection (blink + head turn verification)
- Blob 動画アップロード認証 / Blob video-based authentication
- システム別アクセス権限制御 / System-based authorization
- JWT トークン発行 / JWT generation
- アクセスログ記録 / Access logging

---

# 2. 全体フロー / Overall Flow

## システム全体フロー図 / Overall System Flow Diagram

[![Face Authentication Flow](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/Flow.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/Flow.png)

---

## 処理概要 / Flow Summary

### 1. ログイン画面表示 / Open Login Screen
ユーザーが WMS/TMS ログイン画面を開きます。User opens WMS/TMS login screen.

### 2. カメラ起動 / Camera Starts
ブラウザでカメラが起動します。Frontend starts camera.

### 3. ライブネス確認 / Liveness Verification
ユーザー本人確認のため以下を実施します。Liveness verification is performed.
- Blink Detection
- Head Turn Detection

### 4. 5秒動画録画 / Record 5-second Video
5秒間の動画を録画します。Record 5-second face video.

### 5. Blob 生成 / Convert to Blob
録画した動画を Blob に変換します。Convert recorded video into Blob object.

### 6. API 送信 / Send API Request
Blob 動画をバックエンドへ送信します。Blob video is sent to backend API.

### 7. 動画解析 / Video Processing
サーバー側で動画フレーム抽出を行います。Backend extracts frames from uploaded video.

### 8. 顔認証 / Face Matching
抽出フレームから特徴量ベクトルを生成し、登録済みデータと照合します。Generate face embeddings from frames and compare against stored vectors.

### 9. 権限確認 / Authorization Check
対象システムへのアクセス権限を確認します。Check whether user can access requested system.

### 10. JWT 生成 / Generate JWT
認証成功後 JWT を発行します。JWT is generated after successful authentication.

### 11. ログ保存 / Save Access Log
ログイン結果を access_logs に保存します。Authentication result is saved in access_logs.

### 12. ログイン完了 / Login Success
ダッシュボードへ遷移します。User is redirected to dashboard.

---

# 3. データベース構成 / Database Structure

## ER 図 / ER Diagram

[![Database ER Diagram](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/ER.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/ER.png)

---

# 4. テーブル一覧 / Table Definitions

## users

[![users](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/User.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/User.png)

ユーザー基本情報を保存します。Stores user information.

---

## face_embeddings

顔特徴量ベクトルを保存します。Stores facial embedding vectors.

---

## systems

[![systems](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/System.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/System.png)

WMS / TMS システム情報。Stores system information.

---

## roles

[![roles](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/Roles.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/Roles.png)

ロール定義。Stores role definitions.

---

## user_roles

ユーザーとロールの紐付け。Maps users to roles.

---

## role_permissions

ロール別アクセス権限。Role-based permissions.

---

## user_system_permissions

ユーザー個別権限 override。Per-user permission override.

---

## access_logs

ログイン履歴保存。Stores login history.

---

# 5. エラーコード一覧 / Error Code Reference

All endpoints return errors in this format:

```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human readable description"
}
```

| HTTP | Error Code | Description | Endpoint |
|------|------------|-------------|----------|
| 400 | `MISSING_FIELDS` | Required fields are missing | All |
| 400 | `INVALID_SYSTEM_CODE` | systemCode does not exist in systems table | `/face-login` |
| 401 | `FACE_NOT_RECOGNIZED` | No face embedding matched above threshold | `/face-login` |
| 401 | `LIVENESS_FAILED` | Blink or head turn not detected in video | `/face-login` |
| 403 | `ACCESS_DENIED` | User authenticated but not authorized for system | `/face-login` |
| 403 | `USER_DISABLED` | User account is deactivated | `/face-login` |
| 409 | `USER_ALREADY_EXISTS` | Email already registered | `/register` |
| 422 | `INVALID_VIDEO` | Video too short or not a valid Blob | `/face-login`, `/register` |
| 422 | `NO_FACE_DETECTED` | No face found in extracted frames | `/face-login`, `/register` |
| 422 | `EMBEDDING_FAILED` | Could not generate embedding vectors | `/register` |
| 429 | `TOO_MANY_ATTEMPTS` | Too many failed logins, rate limited | `/face-login` |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error | All |

---

# 6. 詳細クエリフロー / Detailed Query Flow

## 詳細フロー図 / Detailed Flow Diagram

[![Detailed Query Flow](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/DetailedFlow.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/DetailedFlow.png)

---

## 🔵 POST /api/register — ユーザー登録 / Register User

Register a new user and store their face embeddings.

```
Client                          Backend                         Database
  │                                │                                │
  │── POST /api/register ─────────>│                                │
  │   { name, email, dept, video } │                                │
  │                                │── Validate fields ────────────>│
  │                                │── Check email exists ─────────>│ users
  │                                │<── email not found ────────────│
  │                                │── INSERT user ────────────────>│ users
  │                                │── Extract frames from video    │
  │                                │── Generate embeddings          │
  │                                │── INSERT embeddings ──────────>│ face_embeddings
  │                                │── SELECT role ────────────────>│ roles
  │                                │── INSERT user_role ───────────>│ user_roles
  │<── 201 { success, userId } ────│                                │
```

**Request**
```json
{
  "name": "Rem",
  "email": "rem@example.com",
  "department": "Engineering",
  "video": "Blob (5 seconds)"
}
```

---

### Step 1 — Validate Request Fields

Check that all required fields are present and video is a valid Blob.

```
IF name IS NULL OR email IS NULL OR department IS NULL OR video IS NULL
  → 400 MISSING_FIELDS

IF video duration < 5 seconds OR video is not valid Blob format
  → 422 INVALID_VIDEO
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "MISSING_FIELDS",
  "message": "name, email, department, and video are required"
}
```

**Error Response (422)**
```json
{
  "success": false,
  "error": "INVALID_VIDEO",
  "message": "Video must be at least 5 seconds and a valid Blob format"
}
```

---

### Step 2 — Check Email Uniqueness | Table: `users`

```sql
SELECT id FROM users WHERE email = 'rem@example.com';
```

| id    | name | email           | department  | is_active | created_at          |
|-------|------|-----------------|-------------|-----------|---------------------|
| u_001 | Rem  | rem@example.com | Engineering | true      | 2025-01-15 09:00:00 |

```
IF row found → 409 USER_ALREADY_EXISTS
IF no row   → continue
```

**Error Response (409)**
```json
{
  "success": false,
  "error": "USER_ALREADY_EXISTS",
  "message": "A user with email 'rem@example.com' already exists"
}
```

---

### Step 3 — Insert User | Table: `users`

```sql
INSERT INTO users (id, name, email, department, is_active, created_at)
VALUES ('u_001', 'Rem', 'rem@example.com', 'Engineering', true, NOW());
```

| id    | name | email           | department  | is_active | created_at          |
|-------|------|-----------------|-------------|-----------|---------------------|
| u_001 | Rem  | rem@example.com | Engineering | true      | 2025-01-15 09:00:00 |

---

### Step 4 — Extract Frames from Video

Backend processes the uploaded Blob video.

```
video Blob
  → extract frames at 1fps
  → filter frames with detectable face region
  → select best N frames

IF no face detected in any frame → 422 NO_FACE_DETECTED
```

**Error Response (422)**
```json
{
  "success": false,
  "error": "NO_FACE_DETECTED",
  "message": "No face could be extracted from the uploaded video"
}
```

---

### Step 5 — Generate & Store Face Embeddings | Table: `face_embeddings`

```sql
INSERT INTO face_embeddings (id, user_id, embedding_vector, is_active, created_at)
VALUES (
  'fe_001',
  'u_001',
  '[0.123, 0.456, 0.789, ...]',
  true,
  NOW()
);
```

| id     | user_id | embedding_vector        | is_active | created_at          |
|--------|---------|-------------------------|-----------|---------------------|
| fe_001 | u_001   | [0.123, 0.456, 0.789…]  | true      | 2025-01-15 09:00:05 |

```
IF embedding generation fails → 422 EMBEDDING_FAILED
```

**Error Response (422)**
```json
{
  "success": false,
  "error": "EMBEDDING_FAILED",
  "message": "Could not generate face embeddings from the provided video"
}
```

---

### Step 6 — Assign Default Role | Tables: `roles`, `user_roles`

```sql
-- Find default role for the system
SELECT id FROM roles WHERE name = 'operator' AND system_id = 'sys_001';

-- Assign role to user
INSERT INTO user_roles (user_id, role_id)
VALUES ('u_001', 'role_001');
```

**`roles`**

| id       | name     | system_id | description       |
|----------|----------|-----------|-------------------|
| role_001 | operator | sys_001   | WMS operator role |
| role_002 | admin    | sys_001   | WMS admin role    |

**`user_roles`**

| user_id | role_id  |
|---------|----------|
| u_001   | role_001 |

---

### Step 7 — Return Success Response

**Response (201)**
```json
{
  "success": true,
  "userId": "u_001",
  "message": "User registered successfully"
}
```

---

## 🟢 POST /api/face-login — 顔認証ログイン / Face Login

Authenticate a user via face video and return a JWT.

```
Client                          Backend                         Database
  │                                │                                │
  │── POST /api/face-login ───────>│                                │
  │   { video, systemCode }        │                                │
  │                                │── Validate fields ─────────────│
  │                                │── Check rate limit ────────────│
  │                                │── Validate video ──────────────│
  │                                │── Run liveness checks ─────────│
  │                                │── Extract frames from video    │
  │                                │── Generate embeddings          │
  │                                │── SELECT all embeddings ──────>│ face_embeddings
  │                                │── Compare cosine similarity    │
  │                                │── SELECT system ──────────────>│ systems
  │                                │── SELECT user by id ──────────>│ users
  │                                │── Check user is_active ────────│
  │                                │── SELECT user override ───────>│ user_system_permissions
  │                                │   IF no override:              │
  │                                │── SELECT user roles ──────────>│ user_roles
  │                                │── SELECT role permissions ────>│ role_permissions
  │                                │── Generate JWT token ──────────│
  │                                │── INSERT access_log ──────────>│ access_logs
  │<── 200 { token, user, ... } ───│                                │
```

**Request**
```json
{
  "video": "Blob (5 seconds)",
  "systemCode": "WMS"
}
```

---

### Step 1 — Validate Request Fields

```
IF video IS NULL OR systemCode IS NULL
  → 400 MISSING_FIELDS
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "MISSING_FIELDS",
  "message": "video and systemCode are required"
}
```

---

### Step 2 — Check Rate Limit

```
Count failed login attempts for this IP in last 5 minutes.

IF failed attempts >= 5
  → 429 TOO_MANY_ATTEMPTS
```

**Error Response (429)**
```json
{
  "success": false,
  "error": "TOO_MANY_ATTEMPTS",
  "message": "Too many failed login attempts. Try again in 5 minutes",
  "retryAfter": 300
}
```

---

### Step 3 — Validate Video Blob

```
IF video duration < 5 seconds OR video is not valid Blob format
  → 422 INVALID_VIDEO
```

**Error Response (422)**
```json
{
  "success": false,
  "error": "INVALID_VIDEO",
  "message": "Video must be at least 5 seconds and a valid Blob format"
}
```

---

### Step 4 — Liveness Check

```
Analyze video for:
  - Blink Detection  → at least 1 blink detected
  - Head Turn Detection → left or right head turn detected

IF blink not detected OR head turn not detected
  → 401 LIVENESS_FAILED
```

**Error Response (401)**
```json
{
  "success": false,
  "error": "LIVENESS_FAILED",
  "message": "Liveness check failed — blink or head turn not detected"
}
```

---

### Step 5 — Extract Frames from Video

```
video Blob
  → extract frames at 1fps
  → filter frames with detectable face region
  → select best N frames

IF no face detected in any frame → 422 NO_FACE_DETECTED
```

**Error Response (422)**
```json
{
  "success": false,
  "error": "NO_FACE_DETECTED",
  "message": "No face could be extracted from the uploaded video"
}
```

---

### Step 6 — Generate Embedding from Extracted Frames

```
Run face embedding model on selected frames
→ produce embedding vector for this login attempt

IF embedding generation fails → 500 INTERNAL_SERVER_ERROR
```

---

### Step 7 — Face Matching | Table: `face_embeddings`

Retrieve all active embeddings and compare using cosine similarity.

```sql
SELECT
  user_id,
  embedding_vector
FROM face_embeddings
WHERE is_active = true;
```

| id     | user_id | embedding_vector        | is_active |
|--------|---------|-------------------------|-----------|
| fe_001 | u_001   | [0.123, 0.456, 0.789…]  | true      |
| fe_002 | u_002   | [0.321, 0.654, 0.987…]  | true      |
| fe_003 | u_003   | [0.741, 0.852, 0.963…]  | true      |

```
Compute cosine similarity between login embedding and each stored vector.
Best match → u_001 (score: 0.98)

IF best match score < threshold (e.g. 0.80)
  → 401 FACE_NOT_RECOGNIZED
```

**Error Response (401)**
```json
{
  "success": false,
  "error": "FACE_NOT_RECOGNIZED",
  "message": "No matching face found",
  "confidenceScore": 0.31
}
```

---

### Step 8 — Find System | Table: `systems`

Resolve the `systemCode` to a `system_id`.

```sql
SELECT id, code, name
FROM systems
WHERE code = 'WMS';
```

| id      | code | name                        | description           |
|---------|------|-----------------------------|-----------------------|
| sys_001 | WMS  | Warehouse Management System | Handles warehouse ops |
| sys_002 | TMS  | Transport Management System | Handles transport ops |

```
Result → sys_001

IF no system found with this code
  → 400 INVALID_SYSTEM_CODE
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "INVALID_SYSTEM_CODE",
  "message": "System 'XYZ' does not exist"
}
```

---

### Step 9 — Fetch Matched User & Check Active Status | Table: `users`

```sql
SELECT id, name, email, department, is_active
FROM users
WHERE id = 'u_001';
```

| id    | name | email           | department  | is_active |
|-------|------|-----------------|-------------|-----------|
| u_001 | Rem  | rem@example.com | Engineering | true      |

```
IF is_active = false
  → 403 USER_DISABLED
```

**Error Response (403)**
```json
{
  "success": false,
  "error": "USER_DISABLED",
  "message": "This user account has been deactivated"
}
```

---

### Step 10 — Check User Override Permission | Table: `user_system_permissions`

Check if an explicit per-user override exists for this system.

```sql
SELECT can_access
FROM user_system_permissions
WHERE user_id = 'u_001'
AND system_id = 'sys_001';
```

| id     | user_id | system_id | can_access |
|--------|---------|-----------|------------|
| usp_01 | u_001   | sys_001   | true       |
| usp_02 | u_002   | sys_001   | false      |

```
IF row found AND can_access = true  → authorized ✅ skip to Step 13
IF row found AND can_access = false → 403 ACCESS_DENIED
IF no row found                     → continue to Step 11 (role check)
```

**Error Response (403)**
```json
{
  "success": false,
  "error": "ACCESS_DENIED",
  "message": "User 'u_001' does not have access to system 'WMS'",
  "userId": "u_001",
  "systemCode": "WMS"
}
```

---

### Step 11 — Get User Roles | Table: `user_roles`

(Only reached if no user override exists)

```sql
SELECT role_id
FROM user_roles
WHERE user_id = 'u_001';
```

| user_id | role_id  |
|---------|----------|
| u_001   | role_001 |

```
IF no roles found → 403 ACCESS_DENIED
```

---

### Step 12 — Check Role Permissions | Table: `role_permissions`

(Only reached if no user override exists)

```sql
SELECT can_access
FROM role_permissions
WHERE role_id IN ('role_001')
AND system_id = 'sys_001';
```

| id    | role_id  | system_id | can_access |
|-------|----------|-----------|------------|
| rp_01 | role_001 | sys_001   | true       |
| rp_02 | role_001 | sys_002   | false      |

```
IF any row has can_access = true  → authorized ✅ continue to Step 13
IF all rows have can_access = false OR no rows → 403 ACCESS_DENIED
```

**Error Response (403)**
```json
{
  "success": false,
  "error": "ACCESS_DENIED",
  "message": "User 'u_001' does not have access to system 'WMS'",
  "userId": "u_001",
  "systemCode": "WMS"
}
```

---

### Step 13 — Generate JWT Token

Build and sign the JWT with user info, roles, and system.

```
JWT Payload:
{
  "userId":     matched user id
  "name":       matched user name
  "roles":      list of role names from user_roles
  "systemCode": requested systemCode (for downstream authorization)
  "systemId":   resolved system id
  "loginAt":    current UTC timestamp
  "exp":        loginAt + 8 hours
}
```

```json
{
  "userId": "u_001",
  "name": "Rem",
  "roles": ["operator"],
  "systemCode": "WMS",
  "systemId": "sys_001",
  "loginAt": "2025-01-15T09:05:00Z",
  "exp": 1736934300
}
```

---

### Step 14 — Save Access Log | Table: `access_logs`

Always insert a log record regardless of success or failure.

```sql
INSERT INTO access_logs (
  user_id,
  system_id,
  authenticated,
  authorized,
  confidence_score,
  created_at
)
VALUES ('u_001', 'sys_001', true, true, 0.98, NOW());
```

| id    | user_id | system_id | authenticated | authorized | confidence_score | created_at          |
|-------|---------|-----------|---------------|------------|------------------|---------------------|
| al_01 | u_001   | sys_001   | true          | true       | 0.98             | 2025-01-15 09:05:01 |
| al_02 | u_002   | sys_001   | false         | false      | 0.41             | 2025-01-14 08:30:10 |
| al_03 | u_001   | sys_002   | true          | false      | 0.95             | 2025-01-13 11:00:00 |

```
authenticated = true  → face matched above threshold
authorized    = true  → user has permission to access system
confidence_score      → cosine similarity score from face matching
```

---

### Step 15 — Return Success Response

**Response (200)**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "u_001",
    "name": "Rem"
  },
  "systemCode": "WMS",
  "redirectTo": "/dashboard"
}
```

---

## 🔴 POST /api/register/system — システム登録 / Register System

Register a new system (WMS, TMS, etc.).

```
Client                          Backend                         Database
  │                                │                                │
  │── POST /api/register/system ──>│                                │
  │   { code, name, description }  │                                │
  │                                │── Validate fields ─────────────│
  │                                │── Check code uniqueness ──────>│ systems
  │                                │── INSERT system ──────────────>│ systems
  │<── 201 { success, systemId } ──│                                │
```

**Request**
```json
{
  "code": "WMS",
  "name": "Warehouse Management System",
  "description": "Handles warehouse operations"
}
```

---

### Step 1 — Validate Request Fields

```
IF code IS NULL OR name IS NULL
  → 400 MISSING_FIELDS
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "MISSING_FIELDS",
  "message": "code and name are required"
}
```

---

### Step 2 — Check System Code Uniqueness | Table: `systems`

```sql
SELECT id FROM systems WHERE code = 'WMS';
```

| id      | code | name                        |
|---------|------|-----------------------------|
| sys_001 | WMS  | Warehouse Management System |

```
IF row found → 409 SYSTEM_ALREADY_EXISTS
IF no row   → continue
```

**Error Response (409)**
```json
{
  "success": false,
  "error": "SYSTEM_ALREADY_EXISTS",
  "message": "System with code 'WMS' already exists"
}
```

---

### Step 3 — Insert System | Table: `systems`

```sql
INSERT INTO systems (id, code, name, description, created_at)
VALUES ('sys_001', 'WMS', 'Warehouse Management System', 'Handles warehouse operations', NOW());
```

| id      | code | name                        | description           | created_at          |
|---------|------|-----------------------------|-----------------------|---------------------|
| sys_001 | WMS  | Warehouse Management System | Handles warehouse ops | 2025-01-01 00:00:00 |
| sys_002 | TMS  | Transport Management System | Handles transport ops | 2025-01-01 00:00:00 |

---

### Step 4 — Return Success Response

**Response (201)**
```json
{
  "success": true,
  "systemId": "sys_001",
  "message": "System registered successfully"
}
```

---

## 🟡 POST /api/roles — ロール作成 / Create Role

Create a role scoped to a specific system.

```
Client                          Backend                         Database
  │                                │                                │
  │── POST /api/roles ────────────>│                                │
  │   { name, systemId }           │                                │
  │                                │── Validate fields ─────────────│
  │                                │── Check system exists ────────>│ systems
  │                                │── Check role name unique ─────>│ roles
  │                                │── INSERT role ────────────────>│ roles
  │<── 201 { success, roleId } ────│                                │
```

**Request**
```json
{
  "name": "operator",
  "systemId": "sys_001"
}
```

---

### Step 1 — Validate Request Fields

```
IF name IS NULL OR systemId IS NULL
  → 400 MISSING_FIELDS
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "MISSING_FIELDS",
  "message": "name and systemId are required"
}
```

---

### Step 2 — Check System Exists | Table: `systems`

```sql
SELECT id FROM systems WHERE id = 'sys_001';
```

| id      | code | name                        |
|---------|------|-----------------------------|
| sys_001 | WMS  | Warehouse Management System |

```
IF no row found → 400 INVALID_SYSTEM_CODE
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "INVALID_SYSTEM_CODE",
  "message": "System 'sys_001' does not exist"
}
```

---

### Step 3 — Check Role Name Uniqueness | Table: `roles`

```sql
SELECT id FROM roles WHERE name = 'operator' AND system_id = 'sys_001';
```

| id       | name     | system_id |
|----------|----------|-----------|
| role_001 | operator | sys_001   |

```
IF row found → 409 ROLE_ALREADY_EXISTS
IF no row   → continue
```

**Error Response (409)**
```json
{
  "success": false,
  "error": "ROLE_ALREADY_EXISTS",
  "message": "Role 'operator' already exists for this system"
}
```

---

### Step 4 — Insert Role | Table: `roles`

```sql
INSERT INTO roles (id, name, system_id, created_at)
VALUES ('role_001', 'operator', 'sys_001', NOW());
```

| id       | name     | system_id | created_at          |
|----------|----------|-----------|---------------------|
| role_001 | operator | sys_001   | 2025-01-01 00:00:00 |
| role_002 | admin    | sys_001   | 2025-01-01 00:00:00 |

---

### Step 5 — Return Success Response

**Response (201)**
```json
{
  "success": true,
  "roleId": "role_001",
  "message": "Role created successfully"
}
```

---

## 🟡 POST /api/roles/permissions — 権限割り当て / Assign Role Permission

Assign a role permission to access a specific system.

```
Client                          Backend                         Database
  │                                │                                │
  │── POST /api/roles/permissions >│                                │
  │   { roleId, systemId,          │                                │
  │     canAccess }                │                                │
  │                                │── Validate fields ─────────────│
  │                                │── Check role exists ──────────>│ roles
  │                                │── Check system exists ────────>│ systems
  │                                │── UPSERT role_permission ─────>│ role_permissions
  │<── 200 { success } ────────────│                                │
```

**Request**
```json
{
  "roleId": "role_001",
  "systemId": "sys_001",
  "canAccess": true
}
```

---

### Step 1 — Validate Request Fields

```
IF roleId IS NULL OR systemId IS NULL OR canAccess IS NULL
  → 400 MISSING_FIELDS
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "MISSING_FIELDS",
  "message": "roleId, systemId, and canAccess are required"
}
```

---

### Step 2 — Check Role Exists | Table: `roles`

```sql
SELECT id FROM roles WHERE id = 'role_001';
```

| id       | name     | system_id |
|----------|----------|-----------|
| role_001 | operator | sys_001   |

```
IF no row found → 404 ROLE_NOT_FOUND
```

**Error Response (404)**
```json
{
  "success": false,
  "error": "ROLE_NOT_FOUND",
  "message": "Role 'role_001' does not exist"
}
```

---

### Step 3 — Check System Exists | Table: `systems`

```sql
SELECT id FROM systems WHERE id = 'sys_001';
```

| id      | code | name                        |
|---------|------|-----------------------------|
| sys_001 | WMS  | Warehouse Management System |

```
IF no row found → 400 INVALID_SYSTEM_CODE
```

---

### Step 4 — Upsert Role Permission | Table: `role_permissions`

```sql
INSERT INTO role_permissions (id, role_id, system_id, can_access)
VALUES ('rp_01', 'role_001', 'sys_001', true)
ON CONFLICT (role_id, system_id)
DO UPDATE SET can_access = EXCLUDED.can_access;
```

| id    | role_id  | system_id | can_access |
|-------|----------|-----------|------------|
| rp_01 | role_001 | sys_001   | true       |
| rp_02 | role_001 | sys_002   | false      |
| rp_03 | role_002 | sys_001   | true       |

---

### Step 5 — Return Success Response

**Response (200)**
```json
{
  "success": true,
  "message": "Role permission updated successfully"
}
```

---

## 🟣 POST /api/users/permissions — ユーザー個別権限 / Set User Permission Override

Set or update a per-user permission override for a specific system. This takes priority over role permissions.

```
Client                          Backend                         Database
  │                                │                                │
  │── POST /api/users/permissions >│                                │
  │   { userId, systemId,          │                                │
  │     canAccess }                │                                │
  │                                │── Validate fields ─────────────│
  │                                │── Check user exists ──────────>│ users
  │                                │── Check system exists ────────>│ systems
  │                                │── UPSERT user permission ─────>│ user_system_permissions
  │<── 200 { success } ────────────│                                │
```

**Request**
```json
{
  "userId": "u_001",
  "systemId": "sys_001",
  "canAccess": true
}
```

---

### Step 1 — Validate Request Fields

```
IF userId IS NULL OR systemId IS NULL OR canAccess IS NULL
  → 400 MISSING_FIELDS
```

**Error Response (400)**
```json
{
  "success": false,
  "error": "MISSING_FIELDS",
  "message": "userId, systemId, and canAccess are required"
}
```

---

### Step 2 — Check User Exists | Table: `users`

```sql
SELECT id, is_active FROM users WHERE id = 'u_001';
```

| id    | name | email           | is_active |
|-------|------|-----------------|-----------|
| u_001 | Rem  | rem@example.com | true      |

```
IF no row found → 404 USER_NOT_FOUND
```

**Error Response (404)**
```json
{
  "success": false,
  "error": "USER_NOT_FOUND",
  "message": "User 'u_001' does not exist"
}
```

---

### Step 3 — Check System Exists | Table: `systems`

```sql
SELECT id FROM systems WHERE id = 'sys_001';
```

| id      | code | name                        |
|---------|------|-----------------------------|
| sys_001 | WMS  | Warehouse Management System |

```
IF no row found → 400 INVALID_SYSTEM_CODE
```

---

### Step 4 — Upsert User Permission Override | Table: `user_system_permissions`

```sql
INSERT INTO user_system_permissions (id, user_id, system_id, can_access)
VALUES ('usp_01', 'u_001', 'sys_001', true)
ON CONFLICT (user_id, system_id)
DO UPDATE SET can_access = EXCLUDED.can_access;
```

| id     | user_id | system_id | can_access |
|--------|---------|-----------|------------|
| usp_01 | u_001   | sys_001   | true       |
| usp_02 | u_002   | sys_001   | false      |
| usp_03 | u_001   | sys_002   | false      |

---

### Step 5 — Return Success Response

**Response (200)**
```json
{
  "success": true,
  "message": "User permission override updated successfully"
}
```

---

# 7. 重要ポイント / Important Notes

- 顔データは静止画ではなく **5秒動画 Blob** で送信されます / Face data is uploaded as a 5-second Blob video
- Blink Detection を実装しています / Blink detection is implemented
- Head Turn Detection を実装しています / Head turn detection is implemented
- サーバー側で動画からフレーム抽出後に顔照合を行います / Backend extracts frames before face matching
- `user_system_permissions` は `role_permissions` より優先されます / user permissions override role permissions
- JWT に `systemCode` を含みます / JWT includes systemCode for downstream authorization
- JWT に roles / systems を含みます / JWT includes roles and accessible systems
- `access_logs` にすべてのログイン履歴を保存します / All login attempts are saved to access_logs — success and failure both

---

# 8. 今後の改善案 / Future Improvements

- Face recognition confidence threshold tuning
- Multi-factor fallback authentication
- Token refresh endpoint `POST /api/auth/refresh`
- Admin dashboard for access log review
- Bulk user registration endpoint
- Role hierarchy support
