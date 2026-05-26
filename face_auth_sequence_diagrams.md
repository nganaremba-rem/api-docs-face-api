# 顔認証API シーケンス図 / Face Authentication API Sequence Diagrams

> このドキュメントは `Doc.md` の補足資料です。API内部処理の流れを視覚的に把握するために使用します。<br/>
> This document supplements `Doc.md`. Use it to visually understand the API's internal processing flow.

すべての図は **Mermaid** 記法で書かれており、GitHub・GitLab・Notion・VSCode等で自動レンダリングされます。プレビューには [mermaid.live](https://mermaid.live) も使用できます。<br/>
All diagrams are written in **Mermaid** notation and render automatically in GitHub, GitLab, Notion, VSCode, and similar tools. For previewing, [mermaid.live](https://mermaid.live) also works.

---

## 📋 凡例 / Legend

| アイコン / Icon | 役割 / Role | 説明 / Description |
|:---:|---|---|
| 👤 | **User** ユーザー | エンドユーザー / End user logging into WMS or TMS |
| 🌐 | **Frontend** フロントエンド | ブラウザ上で動作するログイン画面 / Browser-side login UI |
| 📹 | **Camera** カメラ | `getUserMedia()` 経由のWebカメラ / Webcam via `getUserMedia()` |
| ⚙️ | **Backend** バックエンド | `/face-login` を提供するAPIサーバー / API server hosting `/face-login` |
| 🧠 | **FaceEngine** 顔認証エンジン | フレーム抽出と特徴量照合 / Frame extraction & embedding matching |
| 🗄️ | **Database** データベース | PostgreSQL等のRDBMS / RDBMS such as PostgreSQL |

---

## 1️⃣ 全体シーケンス / Complete Sequence

### 成功パス / Happy Path

ユーザーがログインに成功するまでの完全なフローです。フェーズごとに色分けしてあります。<br/>
The complete flow from user action to successful login. Color-coded by phase.

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '15px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'primaryTextColor': '#1F2937',
    'lineColor': '#6B7280',
    'actorBkg': '#FFF8EC',
    'actorBorder': '#D97706',
    'actorTextColor': '#1F2937',
    'actorLineColor': '#9CA3AF',
    'signalColor': '#1F2937',
    'signalTextColor': '#1F2937',
    'noteBkgColor': '#FEF3C7',
    'noteBorderColor': '#D97706',
    'noteTextColor': '#78350F',
    'activationBkgColor': '#FDE68A',
    'activationBorderColor': '#D97706',
    'sequenceNumberColor': '#FFFFFF',
    'labelBoxBkgColor': '#FEF3C7',
    'labelBoxBorderColor': '#D97706',
    'labelTextColor': '#78350F'
  }
}}%%
sequenceDiagram
    autonumber
    participant U as 👤 User<br/>ユーザー
    participant F as 🌐 Frontend
    participant C as 📹 Camera
    participant B as ⚙️ Backend
    participant E as 🧠 FaceEngine
    participant D as 🗄️ Database

    rect rgb(219, 234, 254)
        Note over U,C: 📱 フェーズ1: クライアント側処理 / Phase 1 — Client-side
        U->>F: ログイン画面を開く<br/>Open login screen
        F->>C: getUserMedia() でカメラ起動<br/>Start camera
        C-->>F: 映像ストリーム<br/>Video stream
        F->>F: まばたき検知<br/>Blink detection
        F->>F: 顔の向き検知<br/>Head turn detection
        F->>C: 5秒間動画録画<br/>Record 5-second video
        C-->>F: 動画Blob生成<br/>Video blob
    end

    rect rgb(254, 243, 199)
        Note over F,E: 🔍 フェーズ2: 顔認証 / Phase 2 — Face Authentication
        F->>+B: POST /face-login<br/>{ video: Blob, systemCode: "WMS" }
        B->>+E: 動画からフレーム抽出<br/>Extract frames from video
        E-->>-B: フレーム配列 / Frame array
        B->>+E: 顔特徴量を生成<br/>Generate face embeddings
        E-->>-B: 特徴量ベクトル / Embedding vectors
        B->>+D: SELECT * FROM face_embeddings<br/>WHERE is_active = true
        D-->>-B: 全登録特徴量<br/>All stored vectors
        B->>B: コサイン類似度計算 → 最良一致<br/>Cosine similarity → best match
        Note over B: ⚠️ スコア < 0.80 ならここで失敗<br/>If score < 0.80, fail here (401)
    end

    rect rgb(254, 226, 226)
        Note over B,D: 🔐 フェーズ3: 認可判定 / Phase 3 — Authorization
        B->>+D: SELECT id FROM systems<br/>WHERE code = 'WMS'
        D-->>-B: system_id = 'sys_001'
        B->>+D: SELECT can_access<br/>FROM user_system_permissions<br/>WHERE user_id = ? AND system_id = ?
        D-->>-B: 個別権限行 or NULL<br/>Override row or NULL

        alt 個別権限あり / Override exists
            Note over B: ✋ 個別権限が最優先<br/>Override wins — short-circuit
        else 個別権限なし / No override
            B->>+D: SELECT role_id FROM user_roles<br/>WHERE user_id = ?
            D-->>-B: ロール一覧 / Role list
            B->>+D: SELECT can_access<br/>FROM role_permissions<br/>WHERE role_id IN (...) AND system_id = ?
            D-->>-B: ロール権限結果 / Role grants
        end
    end

    rect rgb(220, 252, 231)
        Note over B,U: ✅ フェーズ4: レスポンス / Phase 4 — Response
        B->>B: JWT生成<br/>Generate JWT { userId, roles, systems, ts }
        B->>+D: INSERT INTO access_logs<br/>(authenticated, authorized, confidence, ...)
        D-->>-B: ログ保存完了 / Log saved
        B-->>-F: 200 OK<br/>{ success, token, user, redirectTo }
        F->>U: ダッシュボードへ遷移<br/>Redirect to /dashboard
    end
```

### 各フェーズの責任 / Phase Responsibilities

| # | フェーズ / Phase | 責任 / Responsibility |
|:-:|---|---|
| 1 | **クライアント側** Client-side | ライブネス検証と動画キャプチャ。サーバーに到達する前に「なりすまし」を排除。<br/>Liveness verification + video capture. Filters out spoof attempts before they hit the server. |
| 2 | **顔認証** Face Auth | 「あなたは誰か？」を判定。一致しなければ **401**。<br/>Answers "who are you?". Fails with **401** if no match. |
| 3 | **認可判定** Authorization | 「そのシステムにアクセスできるか？」を判定。失敗時は **403**。<br/>Answers "can you access this system?". Fails with **403**. |
| 4 | **レスポンス** Response | JWT発行とログ保存。**成功・失敗にかかわらずログは必ず記録される**。<br/>JWT issuance and log persistence. **Logs are written regardless of outcome.** |

---

## 2️⃣ 認可優先順位 / Authorization Priority

### このAPIの最も重要なルール / The Most Important Rule of This API

> **`user_system_permissions` は `role_permissions` を上書きします。**<br/>
> **`user_system_permissions` overrides `role_permissions`.**

つまり、管理者ロールを持っていても、そのユーザーに対して個別に `can_access = false` の設定があれば、アクセスは拒否されます。逆も同様です。<br/>
This means: even if a user has the admin role, an explicit `can_access = false` entry in `user_system_permissions` will **deny** access. The reverse also applies.

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '16px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'primaryTextColor': '#1F2937',
    'lineColor': '#6B7280',
    'tertiaryColor': '#F3F4F6'
  }
}}%%
flowchart TD
    START([👤 認証済みユーザー<br/>Authenticated user]) --> Q1{個別権限の行が<br/>存在する？<br/>Override row<br/>exists?}

    Q1 -->|YES| Q2{can_access<br/>= true?}
    Q1 -->|NO<br/>個別設定なし<br/>No override| Q3{いずれかのロールが<br/>許可している？<br/>Any role grants<br/>access?}

    Q2 -->|YES<br/>明示的許可<br/>Explicit grant| ALLOW
    Q2 -->|NO<br/>明示的拒否<br/>Explicit deny| DENY

    Q3 -->|YES| ALLOW
    Q3 -->|NO| DENY

    ALLOW([✅ 認可成功<br/>AUTHORIZED<br/>JWT発行 → 200 OK])
    DENY([❌ 認可失敗<br/>DENIED<br/>403 Forbidden])

    classDef startStyle fill:#FFF8EC,stroke:#D97706,stroke-width:2px,color:#1F2937
    classDef decision fill:#FEF3C7,stroke:#D97706,stroke-width:2px,color:#78350F
    classDef allow fill:#D1FAE5,stroke:#059669,stroke-width:3px,color:#064E3B
    classDef deny fill:#FEE2E2,stroke:#DC2626,stroke-width:3px,color:#7F1D1D

    class START startStyle
    class Q1,Q2,Q3 decision
    class ALLOW allow
    class DENY deny
```

### 具体例 / Concrete Examples

| ユーザー / User | ロール / Role | 個別権限 / Override | リクエスト / Request | 結果 / Result |
|---|---|---|---|---|
| Rem | operator | `WMS = true` | WMS | ✅ **個別許可で成功** Allowed via override |
| Hiroshi | admin | （なし / none） | TMS | ✅ **ロール許可で成功** Allowed via admin role |
| Anna | admin | `TMS = false` | TMS | ❌ **個別拒否が優先** Denied — override beats admin role |
| Anna | admin | `TMS = false` | WMS | ✅ **TMSの拒否はWMSに無関係** TMS deny doesn't affect WMS |

---

## 3️⃣ 失敗パターン / Failure Modes

ハッピーパス以外の3つの失敗パターンを示します。どこで失敗するかによって、HTTPステータスと `access_logs` の内容が変わります。<br/>
Three failure modes besides the happy path. Where the failure occurs determines both the HTTP status and what gets written to `access_logs`.

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'primaryTextColor': '#1F2937',
    'lineColor': '#6B7280',
    'actorBkg': '#FFF8EC',
    'actorBorder': '#D97706',
    'actorTextColor': '#1F2937',
    'signalColor': '#1F2937',
    'signalTextColor': '#1F2937',
    'noteBkgColor': '#FEE2E2',
    'noteBorderColor': '#DC2626',
    'noteTextColor': '#7F1D1D',
    'sequenceNumberColor': '#FFFFFF',
    'labelBoxBkgColor': '#FEF3C7',
    'labelBoxBorderColor': '#D97706',
    'labelTextColor': '#78350F'
  }
}}%%
sequenceDiagram
    participant U as 👤 User
    participant F as 🌐 Frontend
    participant B as ⚙️ Backend
    participant D as 🗄️ Database

    Note over U,D: ❌ パターンA: ライブネス失敗 / Pattern A — Liveness Fails
    U->>F: ログイン試行 / Try login
    F->>F: まばたき・顔向き検知 / Blink + head turn
    F-->>U: 「もう一度試してください」<br/>"Please try again"
    Note over F: 🛑 サーバー未到達 — ログ記録なし<br/>Never reaches server — no log row written

    Note over U,D: ❌ パターンB: 認証失敗 (顔不一致) / Pattern B — Authentication Fails
    U->>F: ライブネスOK → 動画送信 / Liveness OK → upload
    F->>B: POST /face-login
    B->>D: SELECT face_embeddings
    D-->>B: 全特徴量 / All vectors
    B->>B: 最高スコア = 0.62 (< 0.80)<br/>Top score 0.62 (below threshold)
    B->>D: INSERT access_logs<br/>(authenticated=false, authorized=false)
    B-->>F: 401 Unauthorized<br/>{ error: "authentication_failed" }
    F-->>U: 「顔を認識できませんでした」<br/>"Face not recognized"

    Note over U,D: ❌ パターンC: 認可失敗 (権限なし) / Pattern C — Authorization Fails
    U->>F: ライブネスOK → 動画送信 / Liveness OK → upload
    F->>B: POST /face-login (systemCode=TMS)
    B->>D: 顔認証成功 → user_id = u_003 / Auth OK
    B->>D: user_system_permissions チェック<br/>Check override
    D-->>B: can_access = false (TMS拒否 / TMS denied)
    B->>D: INSERT access_logs<br/>(authenticated=true, authorized=false)
    B-->>F: 403 Forbidden<br/>{ error: "authorization_failed" }
    F-->>U: 「このシステムへのアクセス権がありません」<br/>"You don't have access to this system"
```

### `access_logs` の見方 / Reading `access_logs`

ログには2つのブール値があり、これによって失敗の種類が一目で分かります。<br/>
The log has two boolean flags that make the failure type immediately readable:

| `authenticated` | `authorized` | 意味 / Meaning |
|:-:|:-:|---|
| ❌ `false` | ❌ `false` | 顔が認識されなかった (401) / Face not recognized — 401 |
| ✅ `true`  | ❌ `false` | 認識されたがアクセス権なし (403) / Recognized but no access — 403 |
| ✅ `true`  | ✅ `true`  | ログイン成功 (200) / Login successful — 200 |
| ❌ `false` | ✅ `true`  | ⚠️ 発生しないはず / Should never occur — investigate if seen |

> **重要 / Important**: ライブネス失敗時はサーバーに到達しないため、ログには残りません。ライブネス問題を追跡したい場合はクライアント側テレメトリが必要です。<br/>
> When liveness fails, the request never reaches the server, so nothing is logged. Client-side telemetry is required if you want to track liveness issues.

---

## 4️⃣ データベース関係図 / Database Relationships

認可判定で参照される4つのテーブルの関係です。<br/>
Relationships between the four tables consulted during authorization.

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706'
  }
}}%%
erDiagram
    users ||--o{ user_roles : "持つ / has"
    users ||--o{ user_system_permissions : "個別権限 / overrides"
    users ||--o{ access_logs : "記録 / logged"
    users ||--o{ face_embeddings : "顔データ / face data"
    roles ||--o{ user_roles : "割当 / assigned to"
    roles ||--o{ role_permissions : "権限 / grants"
    systems ||--o{ role_permissions : "対象 / target"
    systems ||--o{ user_system_permissions : "対象 / target"
    systems ||--o{ access_logs : "対象 / target"

    users {
        string id PK
        string name
        timestamp created_at
    }
    face_embeddings {
        string id PK
        string user_id FK
        vector embedding_vector
        boolean is_active
    }
    systems {
        string id PK
        string code "WMS / TMS"
        string name
    }
    roles {
        string id PK
        string name "admin / operator"
    }
    user_roles {
        string user_id FK
        string role_id FK
    }
    role_permissions {
        string role_id FK
        string system_id FK
        boolean can_access
    }
    user_system_permissions {
        string user_id FK
        string system_id FK
        boolean can_access "⚠ overrides role"
    }
    access_logs {
        string id PK
        string user_id FK
        string system_id FK
        boolean authenticated
        boolean authorized
        float confidence_score
        timestamp created_at
    }
```

---

## 📖 使い方 / How to Use This Document

### チーム間の使い方 / Team-to-Team Usage

**API提供チーム / Producing team:**
- このファイルを `Doc.md` の隣に置き、コード変更時に図も更新する<br/>
  Keep this file next to `Doc.md` and update the diagrams when code changes.
- 新しい失敗パターンが追加されたら、パターンD・E…として図3に追加する<br/>
  When new failure modes are introduced, add them as Pattern D, E… to Diagram 3.

**API利用チーム / Consuming team:**
- 統合バグが発生したら、まずこの図のどのステップで何が起きたかを特定する<br/>
  When integration bugs occur, first identify *which step* in the diagram is misbehaving.
- 「Step 13で `user_system_permissions` のチェックがおかしい」のように、具体的なステップで会話する<br/>
  Discuss issues at the step level: "Step 13's `user_system_permissions` check seems off" rather than vague reports.

### レンダリング / Rendering

| ツール / Tool | 対応 / Support |
|---|---|
| GitHub (README, .md files) | ✅ ネイティブ対応 / Native |
| GitLab | ✅ ネイティブ対応 / Native |
| Notion | ✅ `/code` → Mermaid を選択 / Use `/code` block with Mermaid |
| VSCode | ✅ Markdown Preview Mermaid Support 拡張 / extension |
| Confluence | ✅ Mermaid マクロ / Mermaid macro |
| プレビュー / Preview | [mermaid.live](https://mermaid.live) |

### バージョン履歴 / Version History

| 日付 / Date | 変更 / Change |
|---|---|
| 2026-05-26 | 初版作成 / Initial version |

---

<p align="center"><sub>このドキュメントは <code>Doc.md</code> の付録です / This document is an appendix to <code>Doc.md</code></sub></p>
