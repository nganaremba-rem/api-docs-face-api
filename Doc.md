
<div align="center">

<br/>

# 🔐 Face Authentication API
### Multi-System Authorization Platform

<br/>

![Version](https://img.shields.io/badge/version-1.0.0-6366f1?style=flat-square)
![Auth](https://img.shields.io/badge/auth-face--recognition-10b981?style=flat-square)
![JWT](https://img.shields.io/badge/token-JWT-f59e0b?style=flat-square)
![Liveness](https://img.shields.io/badge/liveness-blink%20%2B%20head--turn-ef4444?style=flat-square)

<br/>

</div>

---

## 📋 Table of Contents

- [System Overview](#-system-overview)
- [Overall Flow](#-overall-flow)
- [Database Structure](#-database-structure)
- [Table Definitions](#-table-definitions)
- [API Reference](#-api-reference)
- [Processing Logic](#-processing-logic)
- [Detailed Query Flow](#-detailed-query-flow)
- [Important Notes](#-important-notes)

---

## 🧩 System Overview

This document describes the full specification of the **Face Authentication Login** and **Multi-System Authorization** platform.

| Feature | Description |
|---|---|
| 🎥 Face Login | Authenticate via 5-second Blob video |
| 👁️ Liveness Detection | Blink + head turn verification |
| 🏢 Multi-System Auth | WMS / TMS access control |
| 🔑 JWT Issuance | Token includes roles and accessible systems |
| 📋 Access Logging | All attempts saved to `access_logs` |

---

## 🔄 Overall Flow

<div align="center">

<!-- Adjust width here → FLOW_IMG_WIDTH = 520px -->
<img src="./Flow.png" width="520px" alt="Overall System Flow Diagram"/>

*Figure 1 — Overall system flow*

</div>

### Flow Summary

```
① Open Login Screen     →  User opens WMS/TMS login page
② Camera Starts         →  Frontend initializes camera
③ Liveness Check        →  Blink detection + head turn detection
④ Record Video          →  5-second face video captured
⑤ Convert to Blob       →  Video converted to Blob object
⑥ Send to API           →  Blob uploaded to backend
⑦ Frame Extraction      →  Backend extracts frames from video
⑧ Face Matching         →  Embeddings generated and compared
⑨ Authorization Check   →  Verify access to requested system
⑩ Generate JWT          →  Token issued on success
⑪ Save Log              →  Result saved to access_logs
⑫ Redirect              →  User redirected to dashboard
```

---

## 🗄️ Database Structure

<div align="center">

<!-- Adjust width here → ER_IMG_WIDTH = 700px -->
<img src="./ER.png" width="700px" alt="Database ER Diagram"/>

*Figure 2 — Entity Relationship Diagram*

</div>

---

## 📦 Table Definitions

<details>
<summary><strong>users</strong> — User base information</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./User.png" width="360px" alt="users table"/>

</details>

<details>
<summary><strong>face_embeddings</strong> — Facial feature vectors</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./Face embeddings.png" width="360px" alt="face_embeddings table"/>

</details>

<details>
<summary><strong>systems</strong> — WMS / TMS system info</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./System.png" width="360px" alt="systems table"/>

</details>

<details>
<summary><strong>roles</strong> — Role definitions</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./Roles.png" width="360px" alt="roles table"/>

</details>

<details>
<summary><strong>user_roles</strong> — User ↔ role mapping</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./user roles.png" width="360px" alt="user_roles table"/>

</details>

<details>
<summary><strong>role_permissions</strong> — Role-based access control</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./role permissions.png" width="360px" alt="role_permissions table"/>

</details>

<details>
<summary><strong>user_system_permissions</strong> — Per-user override permissions</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./user system permissions.png" width="360px" alt="user_system_permissions table"/>

</details>

<details>
<summary><strong>access_logs</strong> — Login history</summary>
<br/>

<!-- Adjust width here → TABLE_IMG_WIDTH = 360px -->
<img src="./access logs.png" width="360px" alt="access_logs table"/>

</details>

---

## 🚀 API Reference

### Endpoint

```http
POST /face-login
```

> **Content-Type:** `multipart/form-data`

---

### Request & Response

<table>
<tr>
<th width="50%">📤 Request</th>
<th width="50%">📥 Response</th>
</tr>
<tr>
<td>

```json
{
  "video": "Blob (5 seconds)",
  "systemCode": "WMS"
}
```

</td>
<td>

```json
{
  "success": true,
  "token": "jwt_token_here",
  "user": {
    "id": "u_001",
    "name": "Rem"
  },
  "redirectTo": "/dashboard"
}
```

</td>
</tr>
<tr>
<td>

| Field | Type | Description |
|---|---|---|
| `video` | `Blob` | 5-second face recording |
| `systemCode` | `string` | Target system (`WMS` / `TMS`) |

</td>
<td>

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Auth result |
| `token` | `string` | JWT access token |
| `user.id` | `string` | Matched user ID |
| `redirectTo` | `string` | Post-login redirect path |

</td>
</tr>
</table>

---

### Error Responses

| Status | Code | Meaning |
|---|---|---|
| `401` | `FACE_NOT_MATCHED` | No matching face found |
| `403` | `ACCESS_DENIED` | User lacks system permission |
| `422` | `LIVENESS_FAILED` | Blink / head-turn not detected |
| `500` | `PROCESSING_ERROR` | Frame extraction failed |

---

## ⚙️ Processing Logic

```
 1.  User opens login screen
 2.  Frontend starts camera
 3.  Blink detection runs
 4.  Head turn detection runs
 5.  Frontend records 5-second face video
 6.  Video converted into Blob
 7.  Blob uploaded to backend
 8.  Backend extracts frames from video
 9.  Face embeddings generated from frames
10.  Compare with stored face_embeddings
11.  Return best matched user
12.  Read requested systemCode
13.  Check user_system_permissions  ← overrides role permissions
14.  Check role_permissions
15.  Generate JWT token
16.  Insert into access_logs
17.  Return login response
18.  Redirect to dashboard
```

---

## 🔍 Detailed Query Flow

<div align="center">

<!-- Adjust width here → DETAIL_IMG_WIDTH = 700px -->
<img src="./DetailedFlow.png" width="700px" alt="Detailed Query Flow Diagram"/>

*Figure 3 — Detailed query flow*

</div>

### Step 1 — Face Matching

```sql
SELECT
  user_id,
  embedding_vector
FROM face_embeddings
WHERE is_active = true;
```

> Compare embeddings extracted from the uploaded video against all active stored vectors.

---

### Step 2 — Find System

```sql
SELECT id
FROM systems
WHERE code = 'WMS';
```

> Resolve `systemCode` from the request to an internal system ID.

---

### Step 3 — Check User Override Permission

```sql
SELECT can_access
FROM user_system_permissions
WHERE user_id = 'u_001'
  AND system_id = 'sys_001';
```

> User-level permission takes **priority** over role-based permission.

---

### Step 4 — Get User Roles

```sql
SELECT role_id
FROM user_roles
WHERE user_id = 'u_001';
```

---

### Step 5 — Check Role Permissions

```sql
SELECT can_access
FROM role_permissions
WHERE role_id IN (...)
  AND system_id = 'sys_001';
```

---

### Step 6 — Generate JWT

JWT payload includes:

```json
{
  "userId": "u_001",
  "roles": ["admin", "operator"],
  "accessibleSystems": ["WMS", "TMS"],
  "loginTimestamp": "2026-05-26T10:00:00Z"
}
```

---

### Step 7 — Save Access Log

```sql
INSERT INTO access_logs (
  user_id,
  system_id,
  authenticated,
  authorized,
  confidence_score,
  created_at
)
VALUES (...);
```

---

### Step 8 — Return Response

Login success response returned. User redirected to `/dashboard`.

---

## ⚠️ Important Notes

> [!IMPORTANT]
> - Face data is uploaded as a **5-second Blob video**, not a static image
> - **Blink detection** is required to pass liveness check
> - **Head turn detection** is required to pass liveness check
> - Frame extraction happens server-side before any face matching
> - `user_system_permissions` **overrides** `role_permissions`
> - JWT contains `roles` and `accessibleSystems`
> - **All login attempts** (success and failure) are saved to `access_logs`

---

## 🔮 Future Improvements

- [ ] Face recognition confidence threshold tuning
- [ ] Multi-factor fallback (PIN / OTP)
- [ ] Real-time liveness score feedback to frontend
- [ ] Admin dashboard for access log review

---

<div align="center">

*Face Authentication API — Internal Documentation*

</div>
