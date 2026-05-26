---
layout: default
title: Face API Docs
---

# 🔐 顔認証 API
### 複数システム認可プラットフォーム
### Face Authentication API — Multi-System Authorization Platform

![Version](https://img.shields.io/badge/Version-1.0.0-6366f1?style=flat-square)
![Auth](https://img.shields.io/badge/Auth-Face%20Recognition-10b981?style=flat-square)
![JWT](https://img.shields.io/badge/Token-JWT-f59e0b?style=flat-square)
![Liveness](https://img.shields.io/badge/Liveness-Blink%20%2B%20Head%20Turn-ef4444?style=flat-square)

---

# 1. システム概要

このドキュメントでは、顔認証ログインおよび複数システム認可プラットフォームの全体仕様について説明します。

This document describes the overall specification of the Face Authentication Login and Multi-System Authorization Platform.

---

## 対象範囲

- 顔認証ログイン
- ライブネス検知（まばたき・顔向き確認）
- 5秒 Blob 動画認証
- システム別アクセス権限制御
- JWT トークン発行
- アクセスログ保存

### English Translation

- Face authentication login
- Liveness detection (blink + head turn verification)
- 5-second Blob video authentication
- System-based authorization control
- JWT token generation
- Access log storage

---

# 2. 全体フロー

## システム全体フロー図

[![Face Authentication Flow](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/Flow.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/Flow.png)

---

## 処理概要

### 1. ログイン画面表示

ユーザーが WMS または TMS のログイン画面を開きます。

**English:**  
User opens the WMS or TMS login screen.

---

### 2. カメラ起動

ブラウザ上でカメラを起動します。

**English:**  
Frontend starts the device camera.

---

### 3. ライブネス確認

本人確認のため以下を実施します。

- まばたき検知
- 顔向き検知（左右）

**English:**  
Liveness verification is performed using:

- Blink detection
- Head turn detection

---

### 4. 5秒動画録画

顔認証用に5秒間の動画を録画します。

**English:**  
A 5-second face video is recorded for authentication.

---

### 5. Blob生成

録画した動画を Blob データへ変換します。

**English:**  
The recorded video is converted into a Blob object.

---

### 6. API送信

Blob 動画をバックエンド API に送信します。

**English:**  
Blob video is sent to backend API.

---

### 7. 動画解析

サーバー側で動画からフレーム抽出を行います。

**English:**  
Backend extracts frames from uploaded video.

---

### 8. 顔照合

抽出したフレームから顔特徴量ベクトルを生成し、登録済みデータと照合します。

**English:**  
Face embeddings are generated from extracted frames and matched against stored embeddings.

---

### 9. 権限確認

対象システムへのアクセス権限を確認します。

**English:**  
Check whether the user has permission to access the requested system.

---

### 10. JWT生成

認証成功後 JWT トークンを発行します。

**English:**  
JWT token is generated after successful authentication.

---

### 11. ログ保存

ログイン結果を access_logs テーブルへ保存します。

**English:**  
Authentication result is stored in `access_logs`.

---

### 12. ログイン完了

ダッシュボード画面へ遷移します。

**English:**  
User is redirected to dashboard.

---

# 3. データベース構成

## ER図

[![Database ER Diagram](https://github.com/nganaremba-rem/api-docs-face-api/raw/main/ER.png)](https://github.com/nganaremba-rem/api-docs-face-api/blob/main/ER.png)

---

# 4. テーブル一覧

## users

ユーザー基本情報を保存します。

**English:**  
Stores user profile information.

---

## face_embeddings

顔特徴量ベクトルを保存します。

**English:**  
Stores facial embedding vectors.

---

## systems

システム情報（WMS / TMS）を管理します。

**English:**  
Stores system information.

---

## roles

ロール定義を管理します。

**English:**  
Stores role definitions.

---

## user_roles

ユーザーとロールの紐付けを管理します。

**English:**  
Maps users to roles.

---

## role_permissions

ロール単位のアクセス権限を管理します。

**English:**  
Stores role-based permissions.

---

## user_system_permissions

ユーザー個別権限を管理します。

**English:**  
Stores per-user permission overrides.

---

## access_logs

認証履歴を保存します。

**English:**  
Stores authentication and login history.

---

# 5. 重要ポイント

- 顔データは静止画ではなく 5秒動画 Blob を使用します
- まばたき検知を実装しています
- 顔向き検知（左右）を実装しています
- サーバー側で動画からフレーム抽出後に顔照合を行います
- user_system_permissions は role_permissions より優先されます
- JWT に systemCode / roles を含みます
- 成功・失敗を含むすべてのログイン履歴を access_logs に保存します

### English Translation

- Face data is uploaded as a 5-second Blob video
- Blink detection is implemented
- Head turn detection is implemented
- Backend extracts frames before face matching
- User permission overrides role permissions
- JWT includes systemCode and roles
- All login attempts are stored in access_logs, including success and failure

---

# 6. 今後の改善案

- 顔認証の一致率しきい値の最適化
- 多要素認証（MFA）対応
- JWT Refresh Token API追加
- 管理者向けアクセスログ管理画面
- 一括ユーザー登録 API
- ロール階層管理対応

### English Translation

- Face recognition confidence threshold tuning
- Multi-factor authentication support
- JWT refresh token endpoint
- Admin dashboard for access log review
- Bulk user registration API
- Role hierarchy support
